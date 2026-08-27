# Can Financial Ratios Predict ESG Scores?
## A Machine Learning Analysis of UK Listed Companies

MSc Business Analytics dissertation, University of Greenwich (BUSI 1783)
Author: Md Aminul Islam Tushar

---

## Overview

This repository contains the full analytical pipeline for a study examining whether firm-level
financial ratios predict ESG performance among UK-listed companies, and whether machine
learning methods improve predictive accuracy over traditional regression.

**Research questions**

- **RQ1**: To what extent can financial ratios predict ESG scores among UK listed companies?
- **RQ2**: Do machine learning models provide better predictive performance than traditional
  regression in explaining ESG scores?

**Sample** — 389 firm-year observations covering 85 non-financial FTSE 250 constituents,
2019–2023.

---

## Data sources

| Source | Provider | Content |
|---|---|---|
| ORBIS | Bureau van Dijk (Moody's) | Balance sheet and income statement items |
| Refinitiv Eikon | LSEG | ESG composite and pillar scores |

Both databases are licensed to the University of Greenwich. FTSE 250 constituent lists were
taken at each year end rather than applying current membership retrospectively, so that firms
enter and exit the sample as index membership changed. This mitigates survivorship bias.

Financial sector firms (NACE section K) and closed-end investment trusts were excluded, as
balance sheet structure and the economic meaning of liquidity and leverage ratios differ
fundamentally for these entities.

The two extracts were merged on ISIN and fiscal year, achieving a 95.8% match rate.

---

## Data availability

The derived analytical dataset (`data/final/analytical_dataset.csv`) is provided to support
verification of the analysis. It contains computed financial ratios and ESG scores at
firm-year level.

Raw extracts from ORBIS and Refinitiv Eikon are **not** redistributed, under the terms of the
University of Greenwich institutional licence. The complete extraction and merge procedure is
documented in `merge_and_clean.py`, allowing full reproduction by any user with access to
these databases.

---

## Repository contents

```
├── README.md                        this file
├── merge_and_clean.py               builds the analytical dataset from raw extracts
├── esg_analysis.ipynb               modelling, evaluation, and interpretation
├── data/
│   └── final/
│       └── analytical_dataset.csv   derived dataset used in the analysis
└── outputs/                         tables and figures reported in Chapter 4
```

---

## Method

Six financial ratios spanning profitability (ROA, ROE), leverage (debt-to-equity, debt-to-assets),
and liquidity (current ratio, quick ratio) are modelled against the Refinitiv ESG composite
score, with controls for firm size, firm age, industry sector, and year.

Three models are compared: Multiple Linear Regression, Random Forest, and XGBoost. Performance
is assessed by R², RMSE, and MAE, with SHAP values applied for feature-level interpretability.

**A note on validation.** ESG scores are highly persistent within firms — between-firm variation
exceeds within-firm variation by a factor of 2.6. All train/test partitions and cross-validation
folds are therefore **grouped by company**, so that no firm appears in both training and test
data. Section 7 of the notebook quantifies the consequence of ignoring this: a conventional
random split inflates apparent XGBoost performance by 0.42 in R², as tree-based models identify
individual firms rather than learning the financial–ESG relationship.

---

## Reproducing the analysis

```bash
pip install pandas numpy matplotlib seaborn statsmodels scikit-learn xgboost shap
jupyter notebook esg_analysis.ipynb
```

Then run all cells. All tables and figures are written to `outputs/`.

To rebuild the dataset from raw database extracts, place the ORBIS and Refinitiv exports in
`data/raw/` and run `python merge_and_clean.py`.
