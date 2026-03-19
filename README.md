# Causal Inference for Conflict with Machine Learning

## Research Question

Does UN peacekeeping deployment reduce violence against civilians, and does this effect vary across different conflict contexts?

## Motivation

Prediction models (Project 4) show what correlates with violence, but they cannot answer whether an intervention *causes* a change in outcomes. This project applies causal machine learning methods to estimate the effect of UN peacekeeping on civilian violence using observational panel data from African countries.

## Methods

- **OLS Regression**: Baseline estimate of the average treatment effect
- **Double Machine Learning (DML)**: Uses ML to control for high-dimensional confounders while preserving valid causal inference (Chernozhukov et al., 2018)
- **Causal Forests**: Estimates heterogeneous treatment effects, revealing where and for whom peacekeeping works best (Athey and Wager, 2018)

## Data Sources

| Dataset | Source | Role |
|---------|--------|------|
| UCDP GED v25.1 | Uppsala University | Outcome: civilian violence events and fatalities |
| IPI Peacekeeping Database | International Peace Institute | Treatment: UN troop deployment by mission |
| World Bank WDI | World Bank | Confounders: GDP per capita, population |
| EPR | ETH Zurich | Confounder: ethnic fractionalization |
| UCDP/PRIO ACD | Uppsala/PRIO | Confounder: active armed conflict |

## Project Structure

```
conflict-causal-inference/
├── data/
│   ├── raw/              # Original downloaded files
│   └── processed/        # Cleaned, merged panel
├── notebooks/
│   ├── 01_data_assembly.ipynb
│   ├── 02_exploratory_analysis.ipynb
│   ├── 03_causal_estimation.ipynb
│   └── 04_heterogeneity_robustness.ipynb
├── docs/                 # Explanatory guides for each notebook
├── figures/              # Saved plots
├── requirements.txt
└── README.md
```

## Notebooks

1. **01 Data Assembly**: Download, clean, and merge all sources into a country-month panel
2. **02 Exploratory Analysis**: Descriptive statistics, treatment/outcome distributions, balance
3. **03 Causal Estimation**: OLS baseline, Double ML, Causal Forests
4. **04 Heterogeneity and Robustness**: CATE mapping, SHAP analysis, sensitivity checks

## Key References

- Chernozhukov, V. et al. (2018). Double/Debiased ML for Treatment and Structural Parameters. *Econometrics Journal*.
- Athey, S. and Wager, S. (2018). Estimation and Inference of Heterogeneous Treatment Effects using Random Forests. *JASA*.
- Hultman, L., Kathman, J., and Shannon, M. (2013). United Nations Peacekeeping and Civilian Protection in Civil War. *AJPS*.
- Wager, S. (2025). *Causal Inference: A Statistical Learning Approach*. Stanford.
- PyWhy: [EconML](https://github.com/py-why/EconML) and [DoWhy](https://github.com/py-why/dowhy)

## Skills Demonstrated

Causal identification with observational data, Double Machine Learning, Causal Forests for heterogeneous effects, multi-source panel data assembly, sensitivity analysis, SHAP interpretation of causal models.
