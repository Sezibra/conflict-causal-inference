# Notebook 04: Heterogeneity and Robustness Guide

## What This Notebook Does

This notebook takes the Causal Forest results from Notebook 03 and asks:
where does peacekeeping work best, what drives the variation, and how
sensitive are the results to our assumptions? It turns statistical output
into a policy-relevant narrative.


## Part 1: Heterogeneity Analysis

### What Is Heterogeneity?

The Average Treatment Effect (ATE) from DML tells you: on average, across
all country-months, peacekeeping changes civilian violence by X events.
But "on average" hides important variation.

Suppose the ATE is -2.0 (peacekeeping reduces violence by 2 events per
month on average). This average is consistent with several stories:
- Peacekeeping reduces violence by 2 everywhere (homogeneous effect).
- Peacekeeping reduces violence by 10 in some places and has zero effect
  elsewhere (heterogeneous effect, partial effectiveness).
- Peacekeeping reduces violence by 5 in some places and increases it by 1
  in others (heterogeneous effect, mixed effectiveness).

These stories have different policy implications. The first says deploy
everywhere. The second says figure out where it works and focus there.
The third says the UN might be doing harm in some contexts.

Causal Forests give you the Conditional Average Treatment Effect (CATE)
for every country-month, so you can distinguish between these stories.


### CATE Analysis by Subgroup

Group the estimated CATEs by meaningful categories and compare:

- **By country**: average CATE for each country. Which countries benefit
  most from peacekeeping? Which benefit least?
- **By conflict intensity**: split country-months into low, medium, and high
  prior violence (using lagged outcome). Does peacekeeping work better in
  low-intensity or high-intensity settings?
- **By GDP per capita**: split into income terciles. Does state capacity
  moderate the peacekeeping effect?
- **By ethnic fractionalization**: split into low, medium, and high. Does
  ethnic diversity weaken or strengthen the peacekeeping effect?

Present these as grouped bar charts or box plots with confidence intervals.


### Geographic Mapping of CATEs

Map the average CATE for each country onto an Africa choropleth. Countries
where peacekeeping has the largest violence-reducing effect appear in one
color. Countries with weaker or null effects appear in another. Countries
without peacekeeping missions are grayed out.

This is the signature visualization for the project. It answers the
question "where does peacekeeping work?" in a single image.


### SHAP Analysis on the Causal Forest

SHAP (SHapley Additive exPlanations) values decompose the CATE prediction
for each observation into contributions from each covariate. Applied to
the Causal Forest, SHAP tells you which covariates drive treatment effect
heterogeneity.

For example, SHAP might show:
- High ethnic fractionalization pushes the CATE toward zero (peacekeeping
  less effective in diverse settings).
- Low GDP pushes the CATE more negative (peacekeeping more effective in
  poorer countries).
- High prior violence pushes the CATE more negative (peacekeeping more
  effective where violence is already severe).

Use the SHAP summary plot (beeswarm plot) to show all covariate
contributions at once. This is the second signature visualization.


## Part 2: Robustness and Sensitivity

### Why Robustness Matters

Every causal estimate relies on assumptions. The most critical assumption
is unconfoundedness: that the confounders you control for are sufficient to
remove all selection bias. You cannot test this assumption directly. But
you can stress-test it.


### Sensitivity Analysis: What If Unconfoundedness Fails?

The question is: how much unobserved confounding would need to exist to
eliminate the estimated effect?

The DoWhy package provides sensitivity analysis tools. The most common
approach is the E-value or the bias-adjusted estimate:

- **E-value**: the minimum strength of association (on the risk ratio scale)
  that an unobserved confounder would need to have with both the treatment
  and the outcome to fully explain away the observed effect. A large E-value
  means the result is robust: only a very strong unobserved confounder could
  overturn it.

- **Bias-adjusted plot**: shows how the estimated ATE changes as you
  assume increasing levels of unobserved confounding. If the effect stays
  negative (protective) even under moderate confounding assumptions, the
  finding is more credible.


### Alternative Specifications

Run the DML and Causal Forest with variations to check stability:

1. **Different ML base learners**: replace GradientBoosting with RandomForest
   as the first-stage model. If the ATE changes substantially, the result
   is sensitive to model choice.

2. **Different lag structures**: use pk_present_lag1 (one-month lag) instead
   of pk_present (contemporaneous). Test a two-month lag. If the effect
   appears only at one specific lag, consider whether the timing makes
   substantive sense.

3. **Different outcome**: switch from onesided_events to onesided_deaths or
   civilian_deaths. If the effect on event counts and fatality counts point
   in the same direction, the result is more credible.

4. **Subsample analysis**: restrict to countries that had at least one
   peacekeeping mission during the sample period (dropping permanently
   untreated countries). This changes the comparison from "PK countries vs.
   non-PK countries" to "PK months vs. non-PK months within PK countries."
   Both are valid questions with different policy implications.


### Overlap Diagnostics

After fitting the treatment model in DML (the "propensity score" equivalent),
plot the predicted probability of treatment for treated and untreated
observations. If the distributions overlap substantially, the positivity
assumption holds. If they do not (e.g., all treated observations have
predicted probability > 0.8 and all untreated have < 0.2), the model
extrapolates into regions where it has no data, and the estimates are
unreliable.


### Placebo Test

Assign treatment randomly (shuffling pk_present across country-months)
and re-run DML. The estimated ATE on shuffled treatment should be
approximately zero. If it is not, something is wrong with the estimation
procedure. This is a basic sanity check.


## Part 3: Connecting to Theory

The final section of this notebook interprets results through the lens of
conflict theory. The numbers from DML and Causal Forests need to connect
to scholarly arguments about why peacekeeping works (or does not).

Three theoretical mechanisms from the literature:

1. **Deterrence**: the presence of armed peacekeepers raises the cost of
   attacking civilians for armed groups. Predicts stronger effects where
   peacekeepers are more numerous or better equipped.

2. **Monitoring and reporting**: peacekeepers observe and report violence,
   creating accountability. Predicts stronger effects where international
   attention is high.

3. **Facilitating negotiation**: peacekeepers provide a buffer that enables
   ceasefire negotiations. Predicts stronger effects in conflicts with
   identifiable negotiating parties.

Your CATE heterogeneity results should speak to which mechanism is most
consistent with the data. If the effect is strongest in high-intensity
conflicts, deterrence is a plausible explanation. If the effect is strongest
in lower-intensity settings, monitoring or negotiation might matter more.


## Output

This notebook produces:
- Subgroup CATE comparisons (bar charts with confidence intervals)
- Choropleth map of average CATE by country
- SHAP beeswarm plot for treatment effect heterogeneity drivers
- Sensitivity analysis plot (E-value or bias-adjusted ATE)
- Overlap diagnostic (propensity score distributions)
- Robustness table (ATE across alternative specifications)
- Placebo test result
- Brief theoretical interpretation connecting findings to the literature

These outputs complete the project and form the basis for the README
narrative on GitHub.
