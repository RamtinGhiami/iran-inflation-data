# Data

Collected 8 August 2026 (core) and 12 August 2026 (labor / multi-country /
PPP / energy expansion), and **frozen**: `data/MANIFEST.json` carries a
SHA-256 for all 140 files in `data/sources/` and `data/raw/`. Nothing
re-fetches, so no published figure can move on its own. Every number cited in
`iran-inflation.pdf` is computed from this snapshot.

**`PROVENANCE.md`** records where each dataset came from and how it was checked;
`MANIFEST.json` carries the SHA-256 of every snapshot file.

Every file in `data/sources/` shares one schema:

`date_gregorian, date_jalali, year_jalali, month_jalali, series_id, value, unit, source`

Prices are stored in **rial**, not toman. Divide by 10 for toman.

---

## The modelling dataset

**`data/dataset_monthly.csv`** — 816 Solar Hijri months (1338-01 → 1405-12) ×
113 columns. This is the dataset the models in the paper are estimated on.

Every monthly series appears as a column on a gap-free Solar Hijri calendar, so
a missing observation is an explicit `NaN` in a present row, never an absent
row. Levels are kept; `_logdiff` and `_yoy` columns are added alongside them.

Derived columns worth knowing:

| Column | Meaning |
|---|---|
| `usd_depreciation_{1,3,6,12}m` | rial depreciation over that horizon |
| `gold_rial_premium` | log(domestic gold) − log(global gold); the monetary component, net of the global commodity move |
| `coin_gold_ratio` | Emami coin price ÷ 18k gram price; an expectations proxy |
| `month_sin`, `month_cos`, `is_nowruz_month` | Solar Hijri seasonality |

**Estimation sample** (target + core drivers present): 141 months, 1392-04 →
1403-12. **Usable for the model** after 12-month differences: 128 months,
1393-05 → 1403-12.

---

## Sources

| File | Rows | Series | Coverage | Freq. |
|---|---:|---:|---|---|
| `tgju_daily.csv` | 51,792 | 12 | 1979-12 → 2026-08 | daily |
| `tgju_monthly.csv` | 2,378 | 12 | 1358-10 → 1405-05 | monthly |
| `commodities_monthly.csv` | 15,122 | 20 | 1338-10 → 1405-05 | monthly |
| `wb_inflation_monthly.csv` | 710 | 4 | 1383-01 → 1403-12 | monthly |
| `worldbank_annual.csv` | 1,357 | 26 | 1960 → 2025 | annual |
| `iran_annual_inflation_idp.csv` | 78 | 1 | 1316 → 1393 | annual |
| `worldbank_annual_multi.csv` | 3,462 | 26 codes × IRN/EGY/TUR | 1960 → 2025 | annual |
| `ilostat_annual.csv` | 544 | LFS indicators × IRN/EGY/TUR | 1990 → 2025 | annual |

The last two are the 12 August expansion feeding the extended branch analyses
(`18`–`23`); `PROVENANCE.md` §8–§9 record them.
The Gregorian-annual files stay off the monthly Jalali grid deliberately and
never enter `dataset_monthly.csv`.

### Market prices (TGJU, daily, rial)

`usd_free_market` (3,921 days from 2011-11), `eur_free_market`,
`gbp_free_market`, `aed_free_market`, `coin_emami` (4,259 days from 2010-04),
`coin_bahar`, `coin_half`, `coin_quarter`, `gold_18k`, `gold_24k`,
`gold_mesghal`, and `gold_ounce_usd` (12,107 days from 1979-12, in **USD**).

TGJU exposes a stable JSON endpoint per symbol, which is what makes it usable
in an automated pipeline; the government portals do not.

### Consumer prices

From the World Bank Global Inflation Database (Ha, Kose & Ohnsorge 2021):
`cpi_headline_wb` (252 months), `cpi_core_wb` (168), `cpi_energy_wb` (168),
`ppi_wb` (122). The food sheet carries an Iran row with no values, so that
series is absent at source.

**Calendar alignment is validated, not assumed.** The workbook stamps Solar
Hijri observations with Gregorian months. A zero-month offset reproduces the
official SCI annual rates to within 0.7 pp (1401: 47.1 vs 46.5; 1402: 41.4 vs
40.7; 1403: 32.4 vs 32.5). A +2-month shift fits the World Bank's *annual*
series better and was rejected — that series itself sits ~3 pp from the SCI
figures, so the improvement was fitting one compilation's quirks.

### Commodities

Brent from Yahoo Finance (Stooq fallback), plus 19 World Bank Pink Sheet series
including the food, energy, agriculture and fertiliser indices, several
reaching back to 1338. The Pink Sheet document id rotates annually and a stale
id silently serves an outdated workbook, so the fetcher tries ids newest-first.

### Macro, fiscal and monetisation

26 annual World Bank indicators. Beyond the usual GDP/trade/unemployment block
this carries the fiscal channel: `claims_on_govt_pct_m3` and
`claims_on_govt_pct_gdp` (monetisation), `net_domestic_credit_lcu`,
`net_lending_pct_gdp`, `govt_expense_pct_gdp`, `oil_rents_pct_gdp`,
`govt_consumption_pct_gdp`.

**Known truncations:** broad money ends 2016; the direct fiscal-balance series
end **2009**; monetisation ends 2016; real interest rate exists only 2004–2016;
official exchange rate ends 2023.

---

## What is not here, and why

**No monthly Iranian fiscal or monetary series.** The domestic primary
sources — `amar.org.ir` (SCI), `cbi.ir` / `tsd.cbi.ir` (CBI) and `tsetmc.com` —
offer no documented public API, no stable download endpoints and no
machine-readable time-series interface, so an automated, refreshable pipeline
cannot rely on them. Rather than build a fragile scraper we used reputable
international sources with proper programmatic access.

Consequences, in order of severity:

1. **The fiscal channel cannot enter the monthly model.** The currency and gold
   blocks absorb whatever it contributes, so the ~55% attributed to
   currency-linked variables is an **upper bound** on their independent
   contribution.
2. **The target is a secondary compilation.** `cpi_headline_wb` is the World
   Bank's series, not the primary SCI release. Named with a `_wb` suffix so it
   can never be confused with `cpi_sci`, which stays reserved.
3. **The target ends 17 months before the drivers.** The 1404–1405 surge can be
   forecast but not scored.
4. **No COICOP or provincial breakdown**, which would lift the sample from ~140
   rows into the thousands and make causal forests viable.

A best-effort crawler for the SCI/CBI landing pages exists (`PROVENANCE.md`
§6) but is not part of the reproducible pipeline. If those institutions publish
proper machine-readable series, items 1–4 all resolve.

`data/raw/` retains every payload so the panel is reproducible even when a
source revises or moves.

---

## Results

| File | Contents |
|---|---|
| `results/model/scores.csv` | Out-of-sample RMSE, MAE, direction, Diebold–Mariano, Holm-adjusted p |
| `results/model/forecasts.csv` | 56 one-step-ahead forecasts per model |
| `results/model/feature_importance.csv` | Permutation importance and mean \|SHAP\| |
| `results/model/block_attribution.csv` | Importance rolled up into driver blocks |
| `results/model/ablation.csv` | Each block withheld and re-scored out of sample |
| `results/model/stationarity.csv` | ADF and KPSS on the target and headline drivers |
| `results/model/stationarity_robustness.csv` | Refit with the I(1) levels dropped |
| `results/model/calendar_robustness.csv` | Refit under a ±1 month target shift |
| `results/model/rolling_passthrough.csv` | 48-month rolling pass-through coefficient |
| `results/model/forecast_gap.csv` | Projection over the 17 uncovered months, with bootstrap bands |
| `results/misery_index.csv` | Okun, extended and Hanke-style misery indices |
| `results/fiscal_annual_wide.csv` | Annual fiscal/monetary panel |
| `results/labor_panel.csv` + `labor_analysis.json` | Labor branch: LFS indicators, jobs gap vs EGY/TUR |
| `results/growth_regimes.json` + `.csv` | Growth stagnation since 1387; the two inflation regimes |
| `results/ppp_analysis.json` + `ppp_annual.csv` | Purchasing power at parity; rial undervaluation |
| `results/energy_analysis.json` + `energy_annual.csv` | Implicit energy subsidy, with assumptions and band |
| `results/scenario.json` + `scenario_paths.csv` | Labeled counterfactual: oil-revenue thresholds, invested-capital table |
| `results/audit_issues.csv` | Open data-quality findings, by severity |
| `results/source_availability.csv` | Which hosts answered, from this network |
| `results/iran_official_inventory.csv` | Outcome of the SCI/CBI crawl attempts |
