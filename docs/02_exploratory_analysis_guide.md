# Notebook 02: Exploratory Analysis Guide

## What This Notebook Does

This notebook examines the country-month panel built in Notebook 01. It
produces descriptive statistics, visualizations, and balance checks that
answer one question: is this data suitable for causal inference?

You do not run causal models on data you do not understand. This notebook
forces you to look at the data structure, the treatment distribution, the
outcome distribution, and the relationship between confounders and treatment
before estimating any effects.


## Why Exploratory Analysis Matters for Causal Inference

In a predictive ML project (Project 4), EDA helps you understand feature
distributions and spot data quality issues. In a causal inference project,
EDA serves an additional purpose: it helps you assess whether the core
assumptions of your causal design are plausible.

The two key assumptions for DML and Causal Forests are:

1. **Unconfoundedness (conditional ignorability)**: given the confounders we
   control for, treatment assignment is as good as random. In our case,
   this means that after conditioning on GDP, population, ethnic
   fractionalization, prior conflict, and other variables, whether a country
   gets a UN peacekeeping mission in a given month is unrelated to
   unobserved factors that also affect civilian violence.

   This assumption is not testable. But EDA helps you see whether the
   confounders you have plausibly capture the main drivers of both treatment
   and outcome. If treatment correlates strongly with something you cannot
   measure, the assumption is less credible.

2. **Overlap (positivity)**: for every combination of confounder values,
   there must be some probability of being treated and some probability
   of being untreated. If all high-conflict countries have peacekeeping
   and all low-conflict countries do not, there is no overlap: you never
   observe a high-conflict country without peacekeeping. In that case,
   the model cannot estimate what would happen to a high-conflict country
   without the mission.

   EDA lets you check overlap visually by comparing confounder distributions
   between treated and untreated country-months.


## What to Examine

### Treatment Distribution

- How many country-months have a peacekeeping mission present vs. absent?
- Which countries have missions, and for how long?
- Is there temporal variation (missions starting and ending)?
- A timeline plot of peacekeeping mission presence by country is the single
  most useful visualization here.

If treatment is nearly constant within countries (a country either always has
a mission or never does), then identification comes entirely from the few
countries where missions start or end during the sample period. This is not
fatal, but it is important to understand.


### Outcome Distribution

- What is the distribution of one-sided violence events per country-month?
- How many country-months have zero events? (Expect a large majority.)
- What does the time series of civilian violence look like for key countries?
- Are there outlier months with extremely high violence?

Zero-inflated outcomes are common in event count data. Plot the distribution
with a histogram. Then plot separate time series for the 5-10 countries with
the most one-sided violence events. This gives you a sense of where the
signal is.


### Confounder Balance

The goal is to compare treated and untreated country-months across the
confounders. Produce:

- A table of means for each confounder, split by pk_present (0 vs. 1).
  Include the standardized mean difference (SMD), which is the difference
  in means divided by the pooled standard deviation. An SMD above 0.25
  signals meaningful imbalance.

- Box plots or density plots comparing GDP, population, and ethnic
  fractionalization between treated and untreated groups.

Large imbalance is expected. Countries with peacekeeping missions are
different from countries without them. The point is to document this
imbalance so we know what the causal models must correct for. If the
imbalance is extreme and the overlap is poor, we note this as a limitation.


### Treatment-Outcome Relationship (Naive)

Plot the average number of one-sided violence events per month, split by
pk_present. This gives the naive (unadjusted) difference: do country-months
with peacekeeping have more or less violence?

The naive difference will almost certainly show MORE violence in peacekeeping
country-months. This is not because peacekeeping causes violence. It is
because peacekeeping deploys where violence exists. This is the selection
bias that DML and Causal Forests are designed to address. Showing this bias
explicitly in the EDA is a strong portfolio move because it demonstrates you
understand the problem.


### Correlation Heatmap

A correlation matrix of outcome, treatment, and all confounders helps
identify:
- Which confounders are most correlated with treatment (drivers of selection)
- Which confounders are most correlated with the outcome
- Multicollinearity among confounders (which DML handles well, but good
  to know about)


### Temporal Patterns

Plot aggregate monthly violence across the full panel (all African countries
combined) alongside the total number of countries with active peacekeeping
missions. This shows whether the time trends move together, suggesting
common temporal shocks (e.g., Arab Spring, COVID) that could confound
the analysis.

If both treatment and outcome share a strong time trend, consider adding
year fixed effects or time trends to the confounder set.


## Output

This notebook produces:
- Summary statistics table
- Treatment timeline by country
- Outcome distribution (histogram and time series)
- Balance table with standardized mean differences
- Confounder density plots by treatment group
- Naive treatment-outcome comparison
- Correlation heatmap
- Temporal trend plots

All figures save to `figures/`. The key finding from this notebook is the
magnitude of selection bias (the naive difference), which sets up the
motivation for the causal methods in Notebook 03.
