# Notebook 03: Causal Estimation Guide

## What This Notebook Does

This notebook estimates the causal effect of UN peacekeeping on civilian
violence using three methods of increasing sophistication: OLS regression,
Double Machine Learning (DML), and Causal Forests. Each method answers a
slightly different version of the same question.


## The Core Problem: Confounding

You want to know: does peacekeeping reduce civilian violence? The naive
approach compares violence levels in country-months with peacekeeping vs.
without. But this comparison is contaminated by confounding.

Confounding happens when a third variable influences both the treatment
(peacekeeping deployment) and the outcome (civilian violence). For example,
high conflict intensity makes a country more likely to receive a UN mission
AND more likely to have violence against civilians. If you ignore conflict
intensity, you might conclude peacekeeping increases violence, when the
real story is that the UN sends missions to the worst places.

The solution is to control for confounders: compare country-months that are
similar in GDP, population, ethnic composition, and conflict history, and
then check whether the ones with peacekeeping have less violence.


## Method 1: OLS Regression (Baseline)

OLS (Ordinary Least Squares) is the standard starting point. You regress
the outcome on the treatment plus confounders:

    onesided_events = b0 + b1 * pk_present + b2 * gdp_pc + b3 * population
                      + b4 * ethnic_frac + b5 * active_conflict + ...

The coefficient b1 estimates the Average Treatment Effect (ATE): the average
change in one-sided violence events associated with peacekeeping presence,
holding confounders constant.

OLS assumes:
- The relationship between confounders and outcome is LINEAR.
- The relationship between confounders and treatment is LINEAR.
- No important confounders are missing (unconfoundedness).

The linearity assumptions are strong. If the true relationship is nonlinear
(e.g., the effect of GDP on violence is different for very poor vs.
middle-income countries), OLS will be biased.

We include OLS as a benchmark. It is what most political science papers
report. Comparing OLS to DML shows whether the linearity assumption matters.


## Method 2: Double Machine Learning (DML)

### The Idea

DML, introduced by Chernozhukov, Chetty, Demirer, Duflo, Hansen, Newey,
and Robins (2018), solves the linearity problem. Instead of assuming a linear
relationship between confounders and outcome, it uses ML models (random
forests, gradient boosting) to learn flexible, nonlinear relationships. Then
it extracts the causal effect from the residuals.

### The Three Steps

Step 1 (Outcome Model): Train an ML model to predict the outcome (violence)
from the confounders only (not the treatment). Call the prediction error
the "outcome residual." This residual is the part of violence that the
confounders cannot explain.

Step 2 (Treatment Model): Train an ML model to predict the treatment
(peacekeeping) from the confounders only. Call the prediction error the
"treatment residual." This residual is the part of peacekeeping assignment
that the confounders cannot explain, i.e., the as-if-random variation.

Step 3 (Causal Estimate): Regress the outcome residual on the treatment
residual. The coefficient is the ATE.

### Why This Works

After step 1, you have removed everything the confounders explain about
violence. After step 2, you have removed everything the confounders explain
about peacekeeping. What remains is the variation in peacekeeping that is
not driven by observable confounders, and the variation in violence that is
not driven by observable confounders. The correlation between these two
residuals is the causal effect, under the assumption that no unobserved
confounders exist.

The key advantage: ML models handle nonlinearities and interactions among
confounders automatically. You do not need to specify the functional form.

### Cross-Fitting

DML uses cross-fitting to avoid overfitting bias. The data is split into
K folds (typically 5). For each fold, the ML models are trained on the
other K-1 folds and used to predict on the held-out fold. This prevents
the same data from being used for both fitting and prediction, which would
introduce bias.

### Implementation

We use `econml.dml.LinearDML` from the EconML package. The syntax is:

    from econml.dml import LinearDML
    from sklearn.ensemble import GradientBoostingRegressor, GradientBoostingClassifier

    model = LinearDML(
        model_y=GradientBoostingRegressor(),   # outcome model
        model_t=GradientBoostingClassifier(),  # treatment model (binary)
        cv=5                                    # cross-fitting folds
    )
    model.fit(Y, T, X=None, W=confounders)

Here:
- Y is the outcome vector (onesided_events)
- T is the treatment vector (pk_present)
- W is the confounder matrix (GDP, population, etc.)
- X is optional effect modifiers (we use these in Causal Forests)

The output is `model.ate_` (average treatment effect) with a confidence
interval from `model.ate__inference()`.


## Method 3: Causal Forests

### The Idea

OLS gives one number: the ATE. DML also gives one number. But the effect
of peacekeeping probably varies. A mission in a country with low ethnic
fractionalization and moderate conflict intensity might be very effective,
while the same mission in a country with high ethnic fragmentation and
intense fighting might have a smaller or even null effect.

Causal Forests (Athey and Wager, 2018) estimate the Conditional Average
Treatment Effect (CATE) for each observation. Instead of one average, you
get an estimated effect for every country-month, conditional on its
characteristics.

### How It Works

A Causal Forest is an ensemble of "causal trees." Each tree:

1. Splits the data into subgroups based on the covariates (like a
   classification tree).
2. Within each leaf (terminal node), estimates the treatment effect using
   a local version of the DML procedure.
3. The split criterion is not prediction accuracy (as in random forests)
   but treatment effect heterogeneity. The tree splits where the effect
   differs most between subgroups.

By growing many trees and averaging, the Causal Forest provides a smooth,
stable estimate of the CATE for every observation.

### Implementation

We use `econml.dml.CausalForestDML`:

    from econml.dml import CausalForestDML

    cf_model = CausalForestDML(
        model_y=GradientBoostingRegressor(),
        model_t=GradientBoostingClassifier(),
        cv=5,
        n_estimators=1000
    )
    cf_model.fit(Y, T, X=effect_modifiers, W=confounders)

Here X contains the variables along which you want to explore heterogeneity:
GDP per capita, ethnic fractionalization, prior conflict intensity, etc.

After fitting, you get:
- `cf_model.effect(X)`: the estimated CATE for each observation
- `cf_model.effect_inference(X)`: CATEs with confidence intervals


### Interpreting CATEs

A negative CATE means peacekeeping reduces violence for that observation.
A CATE near zero means no effect. A positive CATE (unlikely but possible)
means peacekeeping is associated with more violence in that context.

The distribution of CATEs tells the policy story: where does peacekeeping
work, where does it not, and what drives the difference?


## Comparing the Three Methods

| Feature | OLS | DML | Causal Forest |
|---------|-----|-----|---------------|
| Output | Single ATE | Single ATE | CATE per observation |
| Confounder control | Linear | Flexible (ML) | Flexible (ML) |
| Handles nonlinearities | No | Yes | Yes |
| Heterogeneous effects | No | No | Yes |
| Interpretation | Coefficient | Coefficient | Distribution of effects |
| Assumptions | Linear, unconfoundedness | Unconfoundedness | Unconfoundedness |

The comparison across methods is itself informative:
- If OLS and DML give similar ATEs, the linearity assumption was reasonable.
- If they differ, nonlinear confounding was present and OLS was biased.
- If Causal Forest CATEs vary widely, there is meaningful heterogeneity
  that a single ATE would hide.


## What to Report

For each method, report:
1. The point estimate (ATE or average of CATEs)
2. The 95% confidence interval
3. The p-value or significance level
4. For Causal Forests: the distribution of CATEs (histogram, summary stats)

Frame results carefully. These are estimates under the assumption of
unconfoundedness. If important confounders are missing, the estimates
are biased. State this clearly. A portfolio that acknowledges limitations
is stronger than one that oversells results.


## Output

This notebook produces:
- OLS regression table
- DML ATE estimate with confidence interval
- Causal Forest ATE and CATE distribution
- Comparison table across all three methods
- Histogram of individual CATEs from the Causal Forest

All outputs prepare the ground for Notebook 04, which maps CATEs
geographically and investigates what drives the heterogeneity.
