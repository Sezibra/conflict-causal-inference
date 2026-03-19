# Notebook 01: Data Assembly Guide

## What This Notebook Does

This notebook downloads, cleans, and merges five data sources into a single
country-month panel dataset covering African countries from 2000 to 2022.
The result is one CSV file where each row represents one country in one month,
with columns for the outcome (civilian violence), the treatment (UN peacekeeping
presence), and confounders (GDP, population, ethnic composition, prior conflict).

This panel is the input for all causal estimation in Notebooks 03 and 04.


## Why Panel Data?

Causal inference from observational data requires comparing treated and untreated
units while controlling for confounding variables. A panel dataset (also called
longitudinal data) observes the same units (countries) across multiple time periods
(months). This structure has two advantages over a single cross-section:

1. You observe the same country before and after peacekeeping deployment,
   which helps control for time-invariant country characteristics (geography,
   colonial history, institutional legacies) that you cannot measure directly.

2. You get more observations. With 54 African countries over 276 months
   (23 years), the theoretical maximum is about 14,900 country-months.
   After filtering to countries with at least some conflict activity, the working
   panel will be smaller, but still large enough for ML-based causal methods.

The unit of analysis is country-month. We aggregate event-level UCDP data up
to this level. We do the same with peacekeeping deployment data.


## The Five Data Sources

### 1. UCDP GED v25.1 (Outcome Variable)

You already know this dataset from Project 1. The UCDP Georeferenced Event
Dataset records individual conflict events with coordinates, dates, actors, and
fatality estimates.

For this project, we filter to Africa and aggregate to country-month level. The
outcome variables are:

- **onesided_events**: count of one-sided violence events (type_of_violence == 3)
  in a country-month. One-sided violence is defined as violence by an organized
  armed group against civilians.
- **onesided_deaths**: sum of the "best" fatality estimate for those events.
- **civilian_deaths**: sum of the deaths_civilians column across all event types.

We use one-sided violence as the primary outcome because it directly captures
what peacekeeping is supposed to prevent: deliberate targeting of civilians by
armed actors.

**Download**: https://ucdp.uu.se/downloads/

You already have `GEDEvent_v25_1.csv` from Project 1. Copy it into `data/raw/`.


### 2. IPI / SIPRI Peacekeeping Deployment Data (Treatment Variable)

The treatment variable is whether a UN peacekeeping mission is present in a
country during a given month, and how many troops are deployed.

The two main sources are:

- **IPI Peacekeeping Database** (International Peace Institute): provides monthly
  data on troop and police contributions to each UN mission. Available at
  https://www.ipinst.org/providing-for-peacekeeping

- **SIPRI Multilateral Peace Operations Database**: provides annual data on
  multilateral peace operations including mandate, personnel, and costs.
  Available at https://www.sipri.org/databases/pko

For a country-month panel, the IPI data is preferable because it is monthly.

If the IPI data requires registration or is hard to access, we will construct a
simplified treatment variable by hand: a binary indicator for whether a country
had an active UN peacekeeping mission in a given month. The list of UN
peacekeeping missions in Africa with their start/end dates is publicly available
from the UN Department of Peace Operations website. The major missions are:

| Mission | Country | Start | End |
|---------|---------|-------|-----|
| MONUC/MONUSCO | DRC | Nov 1999 | Ongoing |
| UNMISS | South Sudan | Jul 2011 | Ongoing |
| MINUSMA | Mali | Apr 2013 | Jun 2023 |
| MINUSCA | CAR | Sep 2014 | Ongoing |
| UNAMID | Sudan (Darfur) | Jul 2007 | Dec 2020 |
| UNOCI | Côte d'Ivoire | Apr 2004 | Jun 2017 |
| UNMIL | Liberia | Sep 2003 | Mar 2018 |
| UNAMSIL | Sierra Leone | Oct 1999 | Dec 2005 |
| UNMEE | Ethiopia/Eritrea | Jul 2000 | Jul 2008 |
| ONUB | Burundi | Jun 2004 | Dec 2006 |
| MINURSO | Western Sahara | Apr 1991 | Ongoing |
| UNMIS | Sudan | Mar 2005 | Jul 2011 |

We construct two treatment variables:
- **pk_present**: binary (0/1), is there any UN peacekeeping mission in this
  country-month?
- **pk_troops**: troop count (if available from IPI data), or missing/zero if not.

The binary variable is the primary treatment for DML and Causal Forests.


### 3. World Bank Development Indicators (Confounders)

GDP per capita and population are standard controls in cross-national conflict
studies. They proxy for state capacity, economic development, and opportunity
cost of rebellion.

We download these using the `wbgapi` Python package, which provides direct
access to the World Bank API. The two indicators are:

- **NY.GDP.PCAP.KD**: GDP per capita (constant 2015 USD)
- **SP.POP.TOTL**: Total population

World Bank data is annual. We merge it onto the monthly panel by year, so all
months within a year get the same GDP and population value. This is standard
practice. Within-year variation in GDP is not meaningful for our analysis.


### 4. Ethnic Power Relations (EPR) Dataset (Confounder)

The EPR dataset from ETH Zurich tracks the access of politically relevant
ethnic groups to state power across countries and years. From it, we derive
ethnic fractionalization, which measures how ethnically divided a country is.

Ethnic fractionalization matters because ethnically fragmented countries tend
to have more localized violence and because peacekeeping effectiveness
varies with the ethnic composition of conflict parties (Hultman et al., 2013).

**Download**: https://icr.ethz.ch/data/epr/

We use the country-year level ethnic fractionalization index. Like GDP, this
merges onto the monthly panel by year.


### 5. UCDP/PRIO Armed Conflict Dataset (Confounder)

The UCDP/PRIO Armed Conflict Dataset (ACD) records whether a country
experienced an active armed conflict (25+ battle deaths) in a given year, and
the conflict type (interstate, intrastate, internationalized intrastate).

This variable matters because peacekeeping missions deploy to countries with
active conflicts, so conflict presence is a strong confounder. Without controlling
for it, we would confuse "peacekeeping causes more violence" (spurious) with
"peacekeeping deploys where violence already exists" (selection).

**Download**: https://ucdp.uu.se/downloads/ (UCDP/PRIO Armed Conflict
Dataset, same page as GED)


## The Merging Logic

The merge follows a clear sequence:

1. **Start with the skeleton**: create a complete country-month grid. Every
   African country gets a row for every month from January 2000 to
   December 2022. This ensures we have zero-count months (no violence,
   no peacekeeping) in the data, which is essential. Without them, we only
   observe months when something happened, biasing the sample.

2. **Merge UCDP GED (outcome)**: aggregate events to country-month, then
   left-join onto the skeleton. Country-months with no events get zeros.

3. **Merge peacekeeping (treatment)**: left-join mission presence onto the
   skeleton by country and month.

4. **Merge World Bank (confounders)**: left-join by country and year.

5. **Merge EPR (confounder)**: left-join by country and year.

6. **Merge UCDP/PRIO ACD (confounder)**: left-join by country and year.

All merges use left joins from the skeleton, so the panel stays balanced
(every country-month row is preserved). Missing values in confounders will
be handled through interpolation or forward-filling for slow-moving variables
like GDP.


## Country Identification: ISO Codes

Different datasets identify countries differently. UCDP uses country names and
a numeric "gwno" (Gleditsch-Ward number). The World Bank uses ISO 3166
three-letter codes (e.g., ETH for Ethiopia). EPR uses its own country IDs.

To merge cleanly, we standardize everything to ISO3 alpha-3 codes. We build
a mapping table from UCDP country names to ISO3 codes. The `pycountry`
package helps with this, though some manual corrections are needed for
contested or historical entities.


## What to Watch For

**Selection bias**: UN peacekeeping deploys to the hardest cases. Countries with
missions tend to be poorer, more violent, and more ethnically divided than
countries without missions. This is exactly why naive OLS will be biased:
it compares very different countries. DML and Causal Forests handle this
through flexible control for confounders, but the confounders must be in the
data. The data assembly step determines what you are able to control for.

**Temporal alignment**: peacekeeping effects take time. Troops do not reduce
violence the month they arrive. Many studies use a one-month lag between
treatment and outcome. We will build lagged treatment variables in the
panel so we can test different lag structures.

**Zero inflation**: most country-months have zero civilian violence events.
This is expected. African countries without active conflict will have long
stretches of zeros. The Causal Forest handles this naturally because it
estimates conditional effects, not a single global parameter.

**Missing data**: World Bank GDP is missing for some conflict-affected countries
in some years (Somalia, South Sudan before 2011). We will document
these gaps and handle them with interpolation or by dropping those
country-years from the analysis. Transparency about missingness matters
more than any single imputation technique.


## Output

The notebook produces one file: `data/processed/panel_country_month.csv`

Each row is a country-month. Columns include:

| Column | Description |
|--------|-------------|
| iso3 | Country ISO3 code |
| country_name | Country name |
| year | Year |
| month | Month (1-12) |
| date | First day of the month (for plotting) |
| onesided_events | Count of one-sided violence events |
| onesided_deaths | Fatalities from one-sided violence |
| civilian_deaths | Civilian deaths across all violence types |
| pk_present | Binary: UN peacekeeping mission present |
| pk_troops | Troop count (if available) |
| gdp_pc | GDP per capita, constant 2015 USD |
| population | Total population |
| ethnic_frac | Ethnic fractionalization index |
| active_conflict | Binary: active armed conflict (25+ deaths/year) |
| onesided_events_lag1 | One-sided events in previous month |
| pk_present_lag1 | Peacekeeping presence in previous month |
