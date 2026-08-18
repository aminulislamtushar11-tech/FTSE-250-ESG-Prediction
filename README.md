# Can Financial Ratios Predict ESG Scores? A Machine Learning Analysis of UK-Listed Companies

**Author:** Md Aminul Islam Tushar
**Module:** BUSI 1783, University of Greenwich

## What this project is about

My dissertation looks at whether a company's financial numbers (like profit, debt, and cash position) can be used to predict its ESG score. I focus on companies in the FTSE 250 index between 2019 and 2023, and compare a machine learning model against a normal regression model to see which one predicts better.

## Where the data comes from

- **ORBIS (Bureau van Dijk / Moody's)** — I used this to get company financial data: total assets, current assets, inventory, current liabilities, shareholders' funds, long-term debt, short-term loans, net income, sector code (NACE), and date the company was set up. I got access through the University of Greenwich.
- **LSEG Workspace (Refinitiv)** — I used this to get each company's ESG score and its three separate pillar scores (Environmental, Social, Governance). I used the plain ESG Score, not the "Combined" score, because the Combined one gets adjusted for news stories/controversies, which isn't what I'm trying to measure. Also accessed through the University of Greenwich.
- **FTSE 250 company list** — I couldn't just use today's list of 250 companies, because that would leave out companies that were in the index at some point but have since left (which would bias the results). So I built a list of which companies were actually in the FTSE 250 at the end of each year from 2019 to 2023, using LSEG's record of companies joining and leaving the index over time.

## How I built the final sample

- I started with FTSE 250 companies, year by year, 2019–2023.
- I removed banks, insurers, and other financial companies, because their balance sheets work differently and the ratios don't mean the same thing for them (this is a common thing to do in this kind of study).
- I also removed 9 investment trusts, funds, and REITs that were hiding under a miscellaneous industry code rather than the normal financial-sector codes. I found them by checking which companies had near-zero debt and unusually extreme liquidity ratios, then looked up their names to confirm (things like Fidelity European Trust, Murray Income Trust, and a couple of REITs). These get excluded for the same reason as banks — their balance sheets are shaped by fund/REIT regulation, not normal business operations.
- I removed companies that didn't have enough years of data (less than 3 years), and any company-year missing too many of the ratios (more than 2 out of 6).
- I matched everything up using each company's ISIN code plus the year, since company names can be spelled differently across databases.
- I also grouped my industry codes into 12 broader categories (like Manufacturing, Retail, Real Estate) instead of leaving 38 narrow codes, since some of the narrow ones only had 2-3 companies in them — too small to use properly as a control variable.
- **What I ended up with: 85 companies, 389 company-year rows of data, covering 2019–2023.**
- **Match rate between the financial data and ESG data: 95.8%.**

## Things I want to be upfront about (limitations)

- When I built the company list, I couldn't find an ISIN code for every single company that used to be in the FTSE 250 but isn't anymore — some of the codes just weren't findable through the tools I had access to. About a third of those older company-year entries had to be left out because of this.
- The ESG data tool only lets me pull history for companies that are STILL in the FTSE 250 today — it can't show me ESG scores for a company that has already left the index, even if I have its financial data from ORBIS. So some companies with good financial data still don't have an ESG score, just because they're no longer in the index. I explain this in more detail in Chapter 3.6.

## What's in this repository

| File | What it is |
|---|---|
| `merge_and_clean.py` | The Python script I wrote to combine the three data sources, remove financial companies, calculate the 6 ratios (ROA, ROE, Debt-to-Equity, Debt-to-Assets, Current Ratio, Quick Ratio), match everything to ESG scores, clean out incomplete rows, and produce the final dataset. |
| `descriptive_statistics.csv` | A summary table (averages, min/max, etc.) of the ratios and ESG scores across my final 389 rows of data. |
| `sample_schema.csv` | A small example (5 rows) showing what my final dataset looks like — company names/codes replaced with fake ones. See the note below on why I couldn't upload the real dataset. |

**Why the real dataset isn't here:** I'm not allowed to publish the actual ORBIS and Refinitiv data outside of the University's systems, because of the licence terms I access them under. So instead I've included the script that builds the dataset, plus a small fake example showing exactly what the real output looks like, so anyone with their own access to ORBIS and Refinitiv could run the same script and reproduce my results.

## How to run this yourself

```bash
pip install pandas numpy openpyxl python-calamine
python merge_and_clean.py
```

You'd need your own exports saved in these locations first:
- `data/interim/ftse250_universe.xlsx` — the FTSE 250 company list by year (ISIN | CompanyName | RIC | Year)
- `data/raw/refinitiv_esg_YYYY-MM-DD.xlsx` — the LSEG ESG export
- `data/raw/orbis_financials_YYYY-MM-DD.xlsx` — the ORBIS financial export

and change the three filenames at the top of `merge_and_clean.py` to match whatever you named your files.
