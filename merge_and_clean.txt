"""
merge_and_clean.py
------------------
Builds the analytical dataset for:
"Can Financial Ratios Predict ESG Scores? A Machine Learning Analysis of UK Listed Companies"

Merges ORBIS financial data with Refinitiv ESG scores for FTSE 250 constituents, 2019-2023.
Implements the sample construction and cleaning rules documented in Chapter 3.

Author: Md Aminul Islam Tushar
Module: BUSI 1783, University of Greenwich

Usage:
    pip install pandas numpy openpyxl python-calamine
    python merge_and_clean.py

ADAPTATION NOTE (kept here deliberately, for the GitHub README / Ch3 methods note):
The original version of this script expected the ESG file to arrive as one sheet
per year. In practice, LSEG Workspace's "Add Column" / Financial Period export
produces a WIDE file instead -- one row per company, with each year's value in
its own column (e.g. "ESG Score (FY-2)" for 2023). The same is true of the ORBIS
export, which produces columns like "Total assets th GBP 2023". Section 3 below
(LOAD) reshapes both wide exports into the long, one-row-per-company-year format
the rest of the pipeline (restriction, ratio computation, merge, cleaning) was
already built around. Nothing downstream of the LOAD section changed.
"""

import re
import pandas as pd
import numpy as np
from pathlib import Path

# =============================================================================
# 1. CONFIGURATION  --  EDIT THESE TO MATCH YOUR ACTUAL FILENAMES
# =============================================================================

RAW = Path("data/raw")
INTERIM = Path("data/interim")
FINAL = Path("data/final")
for d in (INTERIM, FINAL):
    d.mkdir(parents=True, exist_ok=True)

UNIVERSE_FILE = INTERIM / "ftse250_universe.xlsx"             # RIC | CompanyName | ISIN | Year
ESG_FILE      = RAW / "refinitiv_esg_2026-08-15.xlsx"         # wide: one row per co, FY-2..FY-6 columns
ORBIS_FILE    = RAW / "orbis_financials_2026-08-17.xlsx"      # wide: one row per co, year-suffixed columns

YEARS = [2019, 2020, 2021, 2022, 2023]

# LSEG "Financial Period" (FY-N) to calendar year, as configured during extraction
# (extraction date 15 Aug 2026 -> FY0 = FY2025 -> FY-2 = 2023 ... FY-6 = 2019)
FY_MAP = {"FY-2": 2023, "FY-3": 2022, "FY-4": 2021, "FY-5": 2020, "FY-6": 2019}

# NACE Rev.2 Section K = Financial and insurance activities (divisions 64, 65, 66)
FINANCIAL_NACE_PREFIXES = ("64", "65", "66")

# NACE Rev.2 division 94 = "Activities of membership organisations" -- but ORBIS
# codes several closed-end investment trusts/funds and REITs under 94.99 as a
# catch-all. Verified by name lookup (Chapter 3.2): 7 are genuine investment
# trusts/funds (e.g. Fidelity European Trust, Murray Income Trust, Schroder
# Oriental Income Fund) and 2 are REITs (Grainger PLC, Sirius Real Estate).
# Both groups have regulator-driven balance sheets (near-zero gearing, unusual
# liquidity ratios) that don't behave like an ordinary operating company --
# the same logic Chapter 3.2 already applies to banks and insurers.
NON_OPERATING_NACE_PREFIXES = ("94",)

# NACE Rev.2 divisions grouped into sections, for the sector control variable.
# 38 individual 2-digit divisions is too many dummies for 431 (now 389)
# observations -- several would have fewer than 5 observations, and a 10-fold
# CV split risks folds with zero observations for some sectors. Aggregating to
# NACE sections (~13 groups after exclusions) keeps sector as a usable control.
NACE_SECTION_RANGES = [
    ("A", range(1, 4)), ("B", range(5, 10)), ("C", range(10, 34)),
    ("D", range(35, 36)), ("E", range(36, 40)), ("F", range(41, 44)),
    ("G", range(45, 48)), ("H", range(49, 54)), ("I", range(55, 57)),
    ("J", range(58, 64)), ("K", range(64, 67)), ("L", range(68, 69)),
    ("M", range(69, 76)), ("N", range(77, 83)), ("O", range(84, 85)),
    ("P", range(85, 86)), ("Q", range(86, 89)), ("R", range(90, 94)),
    ("S", range(94, 97)), ("T", range(97, 99)), ("U", range(99, 100)),
]

def nace_division_to_section(division):
    """Map a 2-digit NACE division (as int) to its NACE section letter (A-U)."""
    try:
        div = int(division)
    except (TypeError, ValueError):
        return np.nan
    for letter, rng in NACE_SECTION_RANGES:
        if div in rng:
            return letter
    return np.nan

MIN_YEARS_PER_FIRM = 3          # Chapter 3.2 exclusion rule
MAX_MISSING_RATIOS = 2          # Chapter 3.4 rule: drop if missing MORE than this
WINSOR_LOWER, WINSOR_UPPER = 0.01, 0.99

CORE_RATIOS = ["ROA", "ROE", "DebtToEquity", "DebtToAssets", "CurrentRatio", "QuickRatio"]


# =============================================================================
# 2. HELPERS
# =============================================================================

def read_excel_robust(path, **kwargs):
    """Some ORBIS exports carry a style record openpyxl can't parse
    (CellStyle 'applyNumFmt'). Fall back to the calamine engine when that happens.
    """
    try:
        return pd.read_excel(path, **kwargs)
    except Exception:
        return pd.read_excel(path, engine="calamine", **kwargs)


def clean_isin(s):
    """ISINs must match exactly across providers. Strip whitespace, force uppercase."""
    return s.astype(str).str.strip().str.upper().replace({"NAN": np.nan, "": np.nan})


def safe_divide(numerator, denominator):
    """Divide, returning NaN where the denominator is zero or missing.

    Financial ratios with a zero denominator are undefined, not infinite.
    Returning NaN lets the missing-data rule handle them consistently.
    """
    denominator = denominator.replace(0, np.nan)
    return numerator / denominator


def winsorise(series, lower=WINSOR_LOWER, upper=WINSOR_UPPER):
    """Compress extreme values to percentile bounds rather than deleting them.

    Chapter 3.4: extreme leverage and negative equity are economically
    meaningful in this context, so the observations are retained.
    """
    lo, hi = series.quantile(lower), series.quantile(upper)
    return series.clip(lower=lo, upper=hi)


def load_esg_wide(path, sheet_name=0):
    """Reshape the LSEG wide ESG export (one row per company, FY-2..FY-6 columns)
    into a long frame: ISIN | Year | ESG_Score | E_Score | S_Score | G_Score.
    """
    raw = read_excel_robust(path, sheet_name=sheet_name, header=0)
    # Drop the two LSEG header/index rows ("Totals (250)" and the ".FTMC" index row)
    junk_markers = {"Totals (250)", ".FTMC"}
    raw = raw[~raw.iloc[:, 0].astype(str).isin(junk_markers)].copy()
    raw["ISIN"] = clean_isin(raw["ISIN"])

    metrics = {
        "ESG Score": "ESG_Score",
        "Environmental Pillar Score": "E_Score",
        "Social Pillar Score": "S_Score",
        "Governance Pillar Score": "G_Score",
    }

    frames = []
    for fy_tag, year in FY_MAP.items():
        row = {"ISIN": raw["ISIN"].values}
        ok = True
        for metric_prefix, std_name in metrics.items():
            col = next((c for c in raw.columns if c.startswith(metric_prefix) and fy_tag in c), None)
            if col is None:
                ok = False
                print(f"  ! ESG file: could not find '{metric_prefix}' for {fy_tag} ({year})")
                break
            row[std_name] = pd.to_numeric(raw[col], errors="coerce").values
        if ok:
            d = pd.DataFrame(row)
            d["Year"] = year
            frames.append(d)

    if not frames:
        raise ValueError(f"No FY-tagged ESG columns found in {path}")
    return pd.concat(frames, ignore_index=True)


def load_orbis_wide(path, sheet_name="Results"):
    """Reshape the ORBIS wide export (one row per company, year-suffixed columns)
    into a long frame: one row per company-year, with the original ORBIS item
    names preserved as column headers (so the existing COLMAP matching below
    keeps working unchanged).
    """
    raw = read_excel_robust(path, sheet_name=sheet_name, header=0)

    item_prefixes = [
        "Profit (loss) for the period [Net income]",
        "Total assets",
        "Current assets",
        "Stock",
        "Shareholders funds",
        "Long term debt",
        "Current liabilities",
        "Loans & short-term debt",
    ]
    static_cols = {
        "Company name Latin alphabet": "Company name",
        "ISIN number": "ISIN",
        "Date of incorporation": "Date of incorporation",
        "Country ISO code": "Country ISO code",
        "NACE Rev. 2, core code (4 digits)": "NACE Rev. 2, core code (4 digits)",
    }

    frames = []
    for year in YEARS:
        row = {}
        for orig, std in static_cols.items():
            row[std] = raw[orig].values
        ok = True
        for item in item_prefixes:
            col = next((c for c in raw.columns if c.startswith(item) and str(year) in c), None)
            if col is None:
                ok = False
                print(f"  ! ORBIS file: could not find '{item}' for {year}")
                break
            row[item] = pd.to_numeric(raw[col], errors="coerce").values
        if ok:
            d = pd.DataFrame(row)
            d["Year"] = year
            frames.append(d)

    if not frames:
        raise ValueError(f"No year-suffixed ORBIS columns found in {path}")
    out = pd.concat(frames, ignore_index=True)
    out["ISIN"] = clean_isin(out["ISIN"])
    return out


# =============================================================================
# 3. LOAD
# =============================================================================

print("=" * 70)
print("LOADING SOURCES")
print("=" * 70)

universe = read_excel_robust(UNIVERSE_FILE)
universe.columns = [c.strip() for c in universe.columns]
universe["ISIN"] = clean_isin(universe["ISIN"])
universe = universe.dropna(subset=["ISIN"])
print(f"Universe (FTSE 250 firm-years):  {len(universe):>6,}")
print(f"  unique firms:                  {universe['ISIN'].nunique():>6,}")

esg = load_esg_wide(ESG_FILE)
esg.columns = [c.strip() for c in esg.columns]
esg = esg.dropna(subset=["ISIN"])
print(f"ESG records loaded:              {len(esg):>6,}")

orbis = load_orbis_wide(ORBIS_FILE)
orbis.columns = [c.strip() for c in orbis.columns]
orbis = orbis.dropna(subset=["ISIN"])
print(f"ORBIS records loaded:            {len(orbis):>6,}")


# =============================================================================
# 4. RESTRICT ORBIS TO THE FTSE 250 UNIVERSE
# =============================================================================
# Point-in-time membership: a firm only enters the sample for the years in
# which it was actually an index constituent (Chapter 3.2, survivorship bias).

print()
print("=" * 70)
print("APPLYING SAMPLE RULES")
print("=" * 70)

orbis = orbis.merge(universe[["ISIN", "Year"]].drop_duplicates(), on=["ISIN", "Year"], how="inner")
print(f"After restricting to FTSE 250 universe:   {len(orbis):>6,} firm-years")


# =============================================================================
# 5. EXCLUDE FINANCIAL SECTOR
# =============================================================================
# Chapter 3.2: balance sheet structure and the meaning of liquidity ratios
# differ fundamentally for banks and insurers (Brogi and Lagasio, 2018).

nace_col = next((c for c in orbis.columns if "NACE" in c.upper()), None)
if nace_col:
    nace_str = orbis[nace_col].astype(str).str.replace(r"\D", "", regex=True)

    is_financial = nace_str.str.startswith(FINANCIAL_NACE_PREFIXES)
    n_fin = int(is_financial.sum())
    orbis = orbis[~is_financial].copy()
    nace_str = nace_str[~is_financial]
    print(f"Financial-sector firm-years removed:      {n_fin:>6,}")
    print(f"Remaining:                                {len(orbis):>6,} firm-years")

    # Investment trusts, funds and REITs (NACE 94.99 catch-all) -- see note
    # above NON_OPERATING_NACE_PREFIXES. Removed for the same reason as banks
    # and insurers: regulator-driven balance sheets, not ordinary operations.
    is_trust = nace_str.str.startswith(NON_OPERATING_NACE_PREFIXES)
    n_trust = int(is_trust.sum())
    trust_firms = orbis.loc[is_trust, "ISIN"].nunique()
    orbis = orbis[~is_trust].copy()
    print(f"Investment trust/fund/REIT firm-years removed: {n_trust:>3,} ({trust_firms} firms)")
    print(f"Remaining:                                {len(orbis):>6,} firm-years")
else:
    print("  ! No NACE column found - financial sector NOT excluded.")
    print("    Add 'NACE Rev. 2 core code' to your ORBIS export and re-run.")


# =============================================================================
# 6. COMPUTE RATIOS
# =============================================================================
# Computed from raw components rather than taken pre-built from ORBIS, so the
# formulas are explicit, auditable, and match those stated in Chapter 3.3.

print()
print("=" * 70)
print("COMPUTING RATIOS")
print("=" * 70)

COLMAP = {
    "TotalAssets":        ["Total assets", "TotalAssets"],
    "CurrentAssets":      ["Current assets", "CurrentAssets"],
    "Inventory":          ["Stocks", "Inventory", "Stock"],
    "CurrentLiabilities": ["Current liabilities", "CurrentLiabilities"],
    "Equity":             ["Shareholders funds", "Shareholders' funds", "Equity"],
    "LongTermDebt":       ["Long term debt", "LongTermDebt", "Non-current liabilities"],
    "ShortTermDebt":      ["Loans", "ShortTermDebt", "Short term debt"],
    "NetIncome":          ["Net income", "Profit for period", "NetIncome", "Net profit"],
    "Incorporation":      ["Date of incorporation", "Incorporation date"],
}

def find_col(df, candidates):
    for cand in candidates:
        for c in df.columns:
            if c.strip().lower() == cand.strip().lower():
                return c
    for cand in candidates:                       # fall back to partial match
        for c in df.columns:
            if cand.strip().lower() in c.strip().lower():
                return c
    return None

resolved = {}
for std, candidates in COLMAP.items():
    col = find_col(orbis, candidates)
    resolved[std] = col
    status = f"-> '{col}'" if col else "-> NOT FOUND"
    print(f"  {std:<20} {status}")

missing_required = [k for k in ["TotalAssets", "CurrentLiabilities", "Equity", "NetIncome"]
                    if resolved[k] is None]
if missing_required:
    raise SystemExit(f"\nSTOP: required columns not found: {missing_required}\n"
                     f"Check your ORBIS export includes them, or add the exact "
                     f"column name to COLMAP above.")

d = pd.DataFrame({"ISIN": orbis["ISIN"], "Year": orbis["Year"]})
for std, col in resolved.items():
    if col is not None and std != "Incorporation":
        d[std] = pd.to_numeric(orbis[col], errors="coerce")

d["TotalDebt"] = d.get("LongTermDebt", 0).fillna(0) + d.get("ShortTermDebt", 0).fillna(0)

d["ROA"]           = safe_divide(d["NetIncome"], d["TotalAssets"])
d["ROE"]           = safe_divide(d["NetIncome"], d["Equity"])
d["DebtToEquity"]  = safe_divide(d["TotalDebt"], d["Equity"])
d["DebtToAssets"]  = safe_divide(d["TotalDebt"], d["TotalAssets"])
d["CurrentRatio"]  = safe_divide(d["CurrentAssets"], d["CurrentLiabilities"])
d["QuickRatio"]    = safe_divide(d["CurrentAssets"] - d.get("Inventory", 0).fillna(0),
                                 d["CurrentLiabilities"])

# Controls
d["LogTotalAssets"] = np.log(d["TotalAssets"].where(d["TotalAssets"] > 0))
if resolved["Incorporation"]:
    inc = pd.to_datetime(orbis[resolved["Incorporation"]], errors="coerce")
    d["FirmAge"] = d["Year"] - inc.dt.year.values
else:
    d["FirmAge"] = np.nan
    print("  ! Incorporation date not found - FirmAge will be missing.")

if nace_col:
    d["Sector"] = orbis[nace_col].astype(str).str.replace(r"\D", "", regex=True).str[:2]
    d["SectorSection"] = d["Sector"].apply(nace_division_to_section)

print(f"\nRatios computed for {len(d):,} firm-years")


# =============================================================================
# 7. MERGE WITH ESG
# =============================================================================

print()
print("=" * 70)
print("MERGING ON ISIN + YEAR")
print("=" * 70)

esg_col = next((c for c in esg.columns if "ESG" in c.upper() and "PILLAR" not in c.upper()
                and "ENV" not in c.upper() and "SOC" not in c.upper()
                and "GOV" not in c.upper()), None)
if esg_col is None:
    raise SystemExit("STOP: could not identify the ESG score column in your Refinitiv export.")
print(f"Using ESG column: '{esg_col}'")

esg_slim = esg[["ISIN", "Year", esg_col]].rename(columns={esg_col: "ESG_Score"})
for pillar, key in [("E_Score", "ENV"), ("S_Score", "SOCIAL"), ("G_Score", "GOVERN")]:
    col = next((c for c in esg.columns if key in c.upper()), None)
    if col:
        esg_slim[pillar] = esg[col]

esg_slim = esg_slim.drop_duplicates(subset=["ISIN", "Year"])

before = len(d)
merged = d.merge(esg_slim, on=["ISIN", "Year"], how="inner")
match_rate = 100 * len(merged) / before if before else 0

print(f"ORBIS firm-years in:             {before:>6,}")
print(f"Matched to ESG:                  {len(merged):>6,}")
print(f"MATCH RATE:                      {match_rate:>6.1f}%")
if match_rate < 85:
    print("  ! Below 85%. This reflects the ESG source only covering current FTSE 250")
    print("    constituents (see Ch3.6 disclosed limitation), not an ISIN formatting bug.")


# =============================================================================
# 8. MISSING DATA RULE (Chapter 3.4)
# =============================================================================

print()
print("=" * 70)
print("MISSING DATA TREATMENT")
print("=" * 70)

merged = merged.dropna(subset=["ESG_Score"])
print(f"After dropping rows without an ESG score: {len(merged):>6,}")

n_missing = merged[CORE_RATIOS].isna().sum(axis=1)
dropped = int((n_missing > MAX_MISSING_RATIOS).sum())
merged = merged[n_missing <= MAX_MISSING_RATIOS].copy()
print(f"Dropped (missing >{MAX_MISSING_RATIOS} core ratios):        {dropped:>6,}")

# Impute a single missing ratio at the sector-year median.
# Uses SectorSection (13 NACE-section groups) rather than the raw 2-digit
# Sector, since several individual divisions have too few observations for a
# stable median -- the aggregated groups give a more reliable fallback.
if "SectorSection" in merged.columns:
    for r in CORE_RATIOS:
        merged[r] = merged.groupby(["SectorSection", "Year"])[r].transform(
            lambda s: s.fillna(s.median()))
for r in CORE_RATIOS:                              # fallback: year median
    merged[r] = merged.groupby("Year")[r].transform(lambda s: s.fillna(s.median()))

# Minimum years per firm
counts = merged.groupby("ISIN")["Year"].nunique()
keep = counts[counts >= MIN_YEARS_PER_FIRM].index
removed_firms = int((counts < MIN_YEARS_PER_FIRM).sum())
merged = merged[merged["ISIN"].isin(keep)].copy()
print(f"Firms removed (<{MIN_YEARS_PER_FIRM} years of data):         {removed_firms:>6,}")


# =============================================================================
# 9. WINSORISE
# =============================================================================

for r in CORE_RATIOS + ["LogTotalAssets"]:
    if r in merged.columns:
        merged[r] = winsorise(merged[r])
print(f"Winsorised at {int(WINSOR_LOWER*100)}st/{int(WINSOR_UPPER*100)}th percentiles")


# =============================================================================
# 10. OUTPUT
# =============================================================================

merged = merged.sort_values(["ISIN", "Year"]).reset_index(drop=True)
out_path = FINAL / "analytical_dataset.csv"
merged.to_csv(out_path, index=False)

desc = merged[CORE_RATIOS + ["LogTotalAssets", "FirmAge", "ESG_Score"]].describe().T
desc.to_csv(FINAL / "descriptive_statistics.csv")

print()
print("=" * 70)
print("NUMBERS FOR YOUR CHAPTER 3 PLACEHOLDERS")
print("=" * 70)
print(f"  PLACEHOLDER 1  Firms after exclusions:     {merged['ISIN'].nunique():>6,}")
print(f"  PLACEHOLDER 2  Firm-year observations:     {len(merged):>6,}")
print(f"  PLACEHOLDER 3  ISIN merge match rate:      {match_rate:>6.1f}%")
print(f"  PLACEHOLDER 4  GitHub URL:                 (paste your repo link)")
print()
print("Also worth reporting in Chapter 4:")
print(f"  Years covered:                             {merged['Year'].min()}-{merged['Year'].max()}")
print(f"  Mean ESG score:                            {merged['ESG_Score'].mean():>6.1f}")
print(f"  ESG score range:                           {merged['ESG_Score'].min():.1f} - {merged['ESG_Score'].max():.1f}")
if "Sector" in merged.columns:
    print(f"  Distinct NACE divisions (2-digit):         {merged['Sector'].nunique():>6,}")
if "SectorSection" in merged.columns:
    print(f"  Distinct NACE sections (aggregated):       {merged['SectorSection'].nunique():>6,}")
    print(f"  Section sizes: {dict(merged['SectorSection'].value_counts())}")
print()
print(f"Saved: {out_path}")
print(f"Saved: {FINAL / 'descriptive_statistics.csv'}")
print("=" * 70)
