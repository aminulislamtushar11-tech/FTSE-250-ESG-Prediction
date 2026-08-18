# Can Financial Ratios Predict ESG Scores? A Machine Learning Analysis of UK-Listed Companies

**Author:** Md Aminul Islam Tushar
**Module:** BUSI 1783, University of Greenwich

## Research objective

To identify the key financial determinants of ESG performance among UK-listed (FTSE 250) firms, and evaluate whether machine learning models improve predictive accuracy over traditional regression in modelling that relationship.

## Data sources

- **ORBIS (Bureau van Dijk / Moody's)** — company financial statement data (total assets, current assets, inventory, current liabilities, shareholders' funds, long-term debt, short-term loans, net income, NACE sector code, date of incorporation). Accessed via University of Greenwich institutional licence.
- **LSEG Workspace (Refinitiv)** — ESG Score and Environmental / Social / Governance pillar scores (`TR.TRESGScore`, not the media-controversy-adjusted Combined Score). Accessed via University of Greenwich institutional licence.
- **FTSE 250 index membership** — point-in-time constituent lists reconstructed for each year-end 2019–2023 from LSEG's Index Leavers & Joiners changelog, applied backward from the current constituent list. This avoids survivorship bias: firms that left the index during the sample period are included for the years in which they were actually constituents, rather than being silently dropped.

## Sample definition

- **Population:** FTSE 250 constituents, point-in-time membership, 2019–2023.
- **Exclusions:** financial-sector firms (NACE Rev. 2 divisions 64, 65, 66 — banks, insurers, financial services), firms with fewer than 3 years of usable data, firm-years missing more than 2 of the 6 core financial ratios.
- **Merge key:** ISIN + fiscal year (inner join across all three sources).
- **Final sample:** 94 firms, 431 firm-year observations, 2019–2023.
- **ISIN merge match rate (ORBIS ↔ ESG):** 96.4%.

### Known limitations (disclosed, not concealed)

- The point-in-time FTSE 250 universe could not be matched to an ISIN for all historical (since-delisted) constituents — roughly 34% of firm-years in the full reconstructed universe lack an ISIN and are therefore excluded from this pipeline. This is a data-availability constraint of the extraction tools used, not a methodological choice.
- The LSEG ESG export can only report history for companies that are FTSE 250 constituents *today*; it cannot retrieve ESG scores for companies that have since left the index, even where their financials are available in ORBIS. This is disclosed here and in Chapter 3.6 as a source of residual survivorship-style attrition specific to the ESG variable, separate from the point-in-time universe construction (which itself is unaffected by this limitation).

## Files in this repository

| File | Description |
|---|---|
| `merge_and_clean.py` | Loads the three raw data sources, restricts ORBIS to the point-in-time FTSE 250 universe, excludes financial-sector firms, computes six financial ratios (ROA, ROE, Debt-to-Equity, Debt-to-Assets, Current Ratio, Quick Ratio) plus firm-size and firm-age controls, merges with ESG scores on ISIN + Year, applies the missing-data and minimum-history rules, winsorises at the 1st/99th percentiles, and outputs the final analytical dataset. |
| `descriptive_statistics.csv` | Summary statistics (count, mean, std, min/max, quartiles) for all ratios and the ESG score across the final 431 firm-year sample. |
| `sample_schema.csv` | A small, anonymised sample (5 rows, ISINs replaced with placeholder codes) showing the exact column structure of the final analytical dataset, provided in place of the full dataset — see licensing note below. |

**Licensing note:** the full merged dataset is not published in this repository, because it is derived from data obtained under University of Greenwich's institutional licences with ORBIS (Moody's/Bureau van Dijk) and LSEG (Refinitiv), which do not permit redistribution of the underlying data. `sample_schema.csv` documents the exact structure and column definitions of the dataset that `merge_and_clean.py` produces, so the pipeline is fully reproducible by anyone with their own institutional access to these two databases.

## How to reproduce

```bash
pip install pandas numpy openpyxl python-calamine
python merge_and_clean.py
```

Place your own raw exports at:
- `data/interim/ftse250_universe.xlsx` — point-in-time FTSE 250 constituent list (ISIN | CompanyName | RIC | Year)
- `data/raw/refinitiv_esg_YYYY-MM-DD.xlsx` — LSEG ESG export
- `data/raw/orbis_financials_YYYY-MM-DD.xlsx` — ORBIS financial export

and update the three filenames at the top of `merge_and_clean.py` to match.
