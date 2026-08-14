# Provenance

Where every dataset came from, how it was retrieved, and how it was checked.

This file is the authoritative record. The collection scripts were recreated
from this record on 12 August 2026 (scripts `01`–`07`, plus the new `17`), and
each carries an offline `--check` mode that re-parses the frozen raw payloads
through the same code used for fetching and compares against the shipped tidy
CSVs — the recreated parsers reproduce every shipped file to machine precision
(TGJU daily bit-exact; aggregated series to ~1e-16 relative).

- **Collected:** core sources 8 August 2026; the labor / multi-country /
  PPP / energy expansion (§8–§9 below) 12 August 2026.
- **Frozen:** recorded in `MANIFEST.json` — 140 files, SHA-256 each.
- **Verification:** every shipped CSV was re-derived offline from its retained
  raw payload and matched (bit-exact for copied values, ~1e-16 relative for
  aggregated means); representative figures were traced from raw bytes through
  the tidy CSVs to the modelling dataset with the calendar mapping recomputed
  independently at each step.
- **Units:** prices are **rial (IRR)**, never toman. Divide by 10 for toman.
- **Grid:** the Solar Hijri (Jalali) month. Daily and Gregorian-monthly series
  are aggregated *up* into it; Iranian data is never pushed *down* into
  Gregorian months.

Every file in `sources/` shares one schema:

```
date_gregorian, date_jalali, year_jalali, month_jalali, series_id, value, unit, source
```

---

## Why the sources are what they are

The pipeline's requirement is programmatic, reproducible retrieval, so that the
panel can be rebuilt and refreshed with up-to-date data at any time. The
domestic primary sources — `amar.org.ir` (Statistical Centre of Iran),
`cbi.ir` and `tsd.cbi.ir` (Central Bank), `tsetmc.com` (Tehran Stock
Exchange) — do not meet it: they publish no documented public API, no stable
download endpoints and no machine-readable time-series interface that an
automated collector can rely on. Rather than build a fragile scraper on top of
them, we followed the usual practice in this literature and used reputable
international sources with proper programmatic access.

Raw payloads are retained in `raw/` rather than re-fetched on demand, so the
panel stays reproducible when a source revises or moves.

Every source below was chosen because it exposes a stable, scriptable
interface while still carrying Iranian data.

---

## 1. `sources/tgju_daily.csv` — free-market FX, gold and coin

**51,792 rows · 12 series · daily · 1979-12 → 2026-08**
Also aggregated to `sources/tgju_monthly.csv` (2,378 rows, Jalali-month means).

| Field | Value |
|---|---|
| Source | TGJU (tgju.org), Iranian market data aggregator |
| Endpoint | `https://api.tgju.org/v1/market/indicator/summary-table-data/{symbol}` |
| Method | HTTP GET per symbol, JSON. One call returns the full history. |
| Raw files | `raw/tgju_*.json`, one per symbol |
| Headers | Browser User-Agent plus `Referer: https://www.tgju.org/` |

**Why this source is reachable:** TGJU sits behind a global CDN (Cloudflare),
unlike the government hosts on Iran's national network.

**Symbol map** — the API symbol is not the series name, and getting this
mapping wrong is silent:

| API symbol | `series_id` | Unit |
|---|---|---|
| `price_dollar_rl` | `usd_free_market` | IRR per USD |
| `price_eur` | `eur_free_market` | IRR per EUR |
| `price_gbp` | `gbp_free_market` | IRR per GBP |
| `price_aed` | `aed_free_market` | IRR per AED |
| `sekee` | `coin_emami` | IRR per coin |
| `sekeb` | `coin_bahar` | IRR per coin |
| `nim` | `coin_half` | IRR per coin |
| `rob` | `coin_quarter` | IRR per coin |
| `geram18` | `gold_18k` | IRR per gram |
| `geram24` | `gold_24k` | IRR per gram |
| `mesghal` | `gold_mesghal` | IRR per mesghal |
| `ons` | `gold_ounce_usd` | **USD** per troy ounce |

Note `ons` is quoted in **dollars**, not rial. It is the global gold price and
belongs to the imported-commodity block, not the domestic one.

The API returns a Jalali date column of its own. That column is trusted rather
than re-derived, and then checked against the Gregorian column (below).

### How it was verified

1. **Independent cross-source agreement — the strongest check available.**
   Gold quoted in USD by TGJU (`ons`) against the World Bank Pink Sheet gold
   series: **r = 0.9996 over 559 months, median deviation 0.75%**. The TGJU
   parser and the calendar transformation are both ours; the World Bank series
   passes through neither. Agreement this close would not arise if either the
   extraction or the calendar fold were wrong. This one comparison validates
   both at once.
2. **Unit sanity via physical content.** An Emami coin contains 8.133 g of
   21.6-carat gold ≈ 9.76 g of 18-carat equivalent, which puts a *physical
   floor* under the coin-to-gold ratio. The observed ratio ranges 9.75–12.68 and
   never falls below that floor — so no rial/toman conversion error (which would
   show as a factor of 10) has entered either series.
3. **Magnitude.** USD rose more than 50× since 2011 (observed: 13,915 → 1,905,846
   IRR, a 137-fold move). A smaller figure would mean the wrong column or unit.
4. **No implausible daily jumps.** Every day-on-day move is under 40%. Larger
   moves indicate a parsing error, not a market event.
5. **Date-column agreement.** 200 randomly sampled rows: the Jalali date
   recomputed from `date_gregorian` matches the stored `date_jalali` exactly.
6. **Strict positivity** on all price series.

---

## 2. `sources/wb_inflation_monthly.csv` — the dependent variable

**710 rows · 4 series · monthly · 1383-01 → 1403-12**

| Field | Value |
|---|---|
| Source | World Bank Global Inflation Database (Ha, Kose & Ohnsorge 2021) |
| URL | `https://thedocs.worldbank.org/en/doc/1ad246272dbbc437c74323719506aa0c-0350012021/original/Inflation-data.xlsx` |
| Method | HTTP GET the workbook, read sheets `hcpi_m`, `fcpi_m`, `ecpi_m`, `ccpi_m`, `ppi_m`, take the `IRN` row |
| Raw file | `raw/wb_inflation_database.xlsx` |

Series obtained: `cpi_headline_wb` (252 months), `cpi_core_wb` (168),
`cpi_energy_wb` (168), `ppi_wb` (122). The food sheet carries an Iran row with
**no values** — absent at source, not mis-parsed.

> **Two caveats travel with this series.** It is a *secondary compilation*, not
> the primary Statistical Centre of Iran release, and it *ends 17 months before
> the drivers do*. It is named `cpi_headline_wb` throughout so it can never be
> silently confused with `cpi_sci`, which stays reserved for the primary series.

### How it was verified

**The calendar alignment is validated, not assumed.** This is the check that
matters most, because the workbook stamps Solar Hijri observations with
*Gregorian* months and documents no per-country mapping.

Two independent arguments fix the offset at zero:

- **Structural.** The Iran series begins at exactly 2004-04 and ends at exactly
  2025-03. Those are *Iranian* year boundaries, not Gregorian ones — the pattern
  you see if Solar Hijri monthly data was stamped with the Gregorian month each
  period begins in.
- **Empirical.** Annual rates derived from the mapped monthly index reproduce
  the officially published SCI figures:

  | Jalali year | Derived | SCI official | Diff |
  |---|---:|---:|---:|
  | 1401 | 47.1% | 46.5% | +0.6 pp |
  | 1402 | 41.4% | 40.7% | +0.7 pp |
  | 1403 | 32.4% | 32.5% | −0.1 pp |

**A rejected alternative, recorded because rejecting it was a judgement call.**
Shifting the stamp forward by two months fits the World Bank's *annual*
inflation series considerably better — correlation 0.969 → 0.992, median gap
2.33 → 0.43 pp. It was rejected anyway: that annual series is itself a distinct
compilation sitting ~3 pp from the official Iranian figures in several years, so
the improvement was fitting one secondary source's idiosyncrasies rather than
correcting a calendar error. **Officially published rates adjudicate; a second
compilation only corroborates.**

Anyone using this panel should still check reported lag structures for
robustness to a ±1 month shift. (The paper does this in §7.5: the zero-offset
specification is also the most accurate, RMSE ratio 0.76 against 0.92 and 0.78.)

Also checked: no month-on-month step below −0.6%, which would betray an
unlinked base change. The Iranian CPI has been rebased 1383 → 1390 → 1395 →
1400, and segments are chain-linked on overlapping periods with the older
segment rescaled so the newest keeps its own base. The chain-link routine
reproduces a known underlying series to a maximum absolute error of 1.4 × 10⁻¹⁴,
and **raises an exception rather than concatenating** when consecutive segments
fail to overlap — a silent gap there would corrupt every subsequent growth rate.

---

## 3. `sources/commodities_monthly.csv` — imported-inflation channel

**15,122 rows · 20 series · monthly · 1338-10 → 1405-05**

| Field | Value |
|---|---|
| Brent source | Yahoo Finance `https://query1.finance.yahoo.com/v8/finance/chart/BZ=F?range=20y&interval=1d` (the Stooq primary failed at collection time and the fallback was used) |
| Pink Sheet source | World Bank Commodity Markets Outlook, `CMO-Historical-Data-Monthly.xlsx` |
| Method | Daily Brent aggregated to Jalali-month means; Pink Sheet sheets `Monthly Prices` and `Monthly Indices` parsed and folded onto the Jalali grid |
| Raw files | `raw/worldbank_pinksheet_monthly.xlsx`. **The Brent daily payload was not retained** — the frozen `brent_crude` series (229 months) is therefore the one series in `sources/` that cannot be re-derived from `raw/`; the offline re-derivation check reports it as skipped, and any future refresh retains the payload as `raw/yahoo_brent_bzf.json` |

Series carried: Brent (both the Pink Sheet monthly series and the daily-derived
`brent_crude`), crude average, natural gas (index and Europe), wheat, maize,
soybean oil, sugar, beef, DAP, gold, plus the total, energy, non-energy,
agriculture, beverages, food, raw-materials and fertiliser indices — 20 series.
(An earlier version of this file also listed rice and urea; they were never
extracted, an error found when the collectors were recreated against the
shipped CSVs.) Several reach back to 1338.

> **A trap worth recording.** The Pink Sheet document id rotates roughly
> annually, and a **stale id keeps serving an outdated workbook instead of
> returning 404**. The fetcher tried ids newest-first. If you refresh this
> source, verify the last observation date rather than trusting HTTP 200.
> Known ids, newest first:
> `74e8be41ceb20fa0da750cda2f6b9e4e-0050012026`,
> `18675f1d1639c7a34d463f59263ba0a2-0050012025`.

The workbook's header block is stacked across up to three rows and each level is
sparse (only the leftmost cell of a merged span carries its label), so column
names are resolved by taking the *lowest non-empty* label per column — otherwise
`energy`, which sits two rows above `food` in `Monthly Indices`, resolves wrong.

### How it was verified

- **Cross-source agreement:** Brent from Yahoo Finance against the Pink Sheet
  Brent series — **r = 0.9922 over 228 months, median gap 1.93%**. Two
  independent providers, so the commodity block is internally consistent.
- **Lossless calendar fold:** every complete Solar Hijri year in the commodity
  block holds exactly twelve observations.

---

## 4. `sources/worldbank_annual.csv` — macro, fiscal, monetisation

**1,357 rows · 26 series · annual · 1960 → 2025**

| Field | Value |
|---|---|
| Source | World Bank World Development Indicators |
| Endpoint | `https://api.worldbank.org/v2/country/IRN/indicator/{code}` |
| Method | HTTP GET per indicator code, JSON, paginated |
| Raw files | `raw/worldbank_{CODE}.json`, one per indicator |

Beyond the usual GDP/trade/unemployment block this carries the fiscal channel:
`claims_on_govt_pct_m3` and `claims_on_govt_pct_gdp` (monetisation),
`net_domestic_credit_lcu`, `net_lending_pct_gdp`, `govt_expense_pct_gdp`,
`oil_rents_pct_gdp`, `govt_consumption_pct_gdp`.

> **Known truncations — these matter for what can be concluded.** Broad money
> ends 2016. The direct fiscal-balance series end **2009**, sixteen years before
> the end of the sample. Monetisation ends 2016. Real interest rate exists only
> 2004–2016. Official exchange rate ends 2023. **None of it overlaps the monthly
> estimation window**, which is why the fiscal channel cannot enter the monthly
> model and is examined separately at annual frequency instead.

### How it was verified

Cross-checked against the Iran Data Portal series (below) on 55 overlapping
years: **r = 0.974, median absolute gap 1.40 pp**. Two independent compilations
of official Iranian statistics agreeing at that level is the available check;
neither is a primary release.

---

## 5. `sources/iran_annual_inflation_idp.csv` — long-run annual inflation

**78 rows · 1 series · annual · 1316 → 1393**

| Field | Value |
|---|---|
| Source | Iran Data Portal, Syracuse University |
| URL | `https://irandataportal.syr.edu/wp-content/uploads/iran-inflation-f-.xlsx` |
| Method | HTTP GET the workbook; it is already keyed on Jalali years |
| Raw file | `raw/irandataportal_inflation.xlsx` |

**Why this source:** Syracuse University mirrors Iranian official statistics and
is hosted in the US, so it stays reachable when `amar.org.ir` does not. It
reaches back to 1315, far earlier than the World Bank's 1960 start.

### How it was verified

The same cross-source comparison as §4, from the other side: r = 0.974 against
the World Bank annual series over 55 overlapping years, median gap 1.40 pp.

---

## 6. Not collected — and why

**No monthly Iranian fiscal or monetary series, and no primary SCI/CBI release.**
A crawler for the SCI and CBI price-index and monetary spreadsheet portals was
written and tested against the portal structure — it crawls the landing pages
and collects every spreadsheet it finds rather than hard-coding URLs, which the
SCI portal reshuffles between releases. It is best-effort and not part of the reproducible pipeline: neither host
exposes a stable, scriptable interface (see the top of this file). Should
those institutions publish machine-readable series, it would resolve all four
consequences below.

Consequences, in order of severity:

1. **The fiscal channel cannot enter the monthly model.** The currency and gold
   blocks absorb whatever it contributes, so the ~55–63% attributed to
   currency-linked variables is an **upper bound** on their independent
   contribution.
2. **The target is a secondary compilation** (see §2).
3. **The target ends 17 months before the drivers.** The 1404–1405 surge can be
   forecast but not scored.
4. **No COICOP or provincial breakdown**, which would lift the sample from ~140
   rows into the thousands.

---

## 7. `dataset_monthly.csv` — the derived modelling dataset

**816 Jalali months × 113 columns.** Not a source: derived from §1–§5, and
rebuildable from the frozen snapshot at any time.

- Every monthly series is a column on a **gap-free** Jalali calendar, so a
  missing observation is an explicit `NaN` in a present row, never an absent row.
- **Nothing is destructively transformed.** `_logdiff` and `_yoy` columns are
  added; levels stay.
- Sixteen edge months are entirely empty because the builder fills whole Jalali
  years. Harmless for joins; drop them before fitting.
- Derived columns of note: `usd_depreciation_{1,3,6,12}m`; `gold_rial_premium`
  (log domestic gold − log global gold, the monetary component net of the global
  commodity move); `coin_gold_ratio` (expectations proxy); `month_sin`,
  `month_cos`, `is_nowruz_month`; and 12 within-month shape statistics
  (`_h2h1`, `_eom`, `_dvol`, `_dmax`) recovered from the daily block.

### How it was verified

A verification suite of 25 checks was run against the panel. Beyond
the source-level checks already listed, at panel level: no duplicate months;
Gregorian dates strictly increasing; consecutive Jalali months 29–31 Gregorian
days apart; core-driver window ≥ 120 months; CPI present, rising, and free of
unlinked base breaks; estimation sample ≥ 100 months.

Calendar handling itself is tested against known dates: Nowruz 1400 = 21 March
2021; Esfand 1399 has 30 days (leap); Esfand 1400 has 29; 2026-08-06 = 1405-05-15.
And a test asserts Jalali months do **not** start on the first of a Gregorian
month — if they did, the whole calendar design would be pointless.

An adversarial data audit additionally records open data-quality findings by
severity in `results/audit_issues.csv`; it reports problems with this dataset
rather than confirming it is fine.

---

## Corrections made after freezing

Recorded here because a snapshot that changes without a record is not a
snapshot. Both were found by the raw-to-dataset trace.

**1. The `unit` column was a placeholder (11 August 2026).** Four source files —
`tgju_daily.csv`, `tgju_monthly.csv`, `commodities_monthly.csv` and
`worldbank_annual.csv` — carried the literal string `see unit column` in every
row instead of a unit. No numeric value was affected and no result changed
(verified: model output is bit-identical before and after), but a dataset that
does not state its own units cannot safely be reused. Units were filled in from,
in order of authority:

- **TGJU:** the symbol→unit mapping in §1 above, which is the collector's own
  table.
- **Commodities:** the Pink Sheet workbook's *own units row* — row 6 of the
  `Monthly Prices` sheet, directly beneath the header — read out of
  `raw/worldbank_pinksheet_monthly.xlsx` rather than recalled. Indices are
  `2010=100`, per the sheet's header text.
- **World Bank annual:** derived from the `series_id` suffix convention this
  project already uses (`_pct`, `_pct_gdp`, `_lcu`, …). Nothing inferred from
  outside the naming.

The trace now asserts that no placeholder units remain and that the stored
commodity units still agree with the raw workbook.

**2. An overstated claim in the paper (11 August 2026).** §4 asserted the
coin-to-gold ratio "never falls below" its physical floor of 9.76 g. It dips
0.11% below in 2 months of 158 — the width of a dealer spread, since coin and
gram are independently quoted and a coin can trade just under its bullion
content in weak demand. The check's actual purpose is to exclude a
rial↔toman confusion, which would displace the ratio by a *factor of ten*. The
paper now states this accurately and the trace tolerance is set at 5%.

---

## 8. `sources/ilostat_annual.csv` — labor markets, IRN/EGY/TUR

**544 rows · annual · 1990 → 2025 · collected 12 August 2026**

| Field | Value |
|---|---|
| Source | ILOSTAT (the ILO's statistics API) |
| Endpoint | `https://rplumber.ilo.org/data/indicator/?id={INDICATOR}&ref_area={ISO3}&timefrom=1990&format=.csv` |
| Raw files | `raw/ilostat_{INDICATOR}_{ISO3}.csv`, exact bytes as served |
| Collector | ILOSTAT SDMX API, one request per indicator and country |

**Why this source:** Iran reports its Labour Force Survey results to the ILO,
so the primary survey's numbers are obtainable through ILOSTAT's documented
API even though `amar.org.ir` offers no scriptable interface. The API
throttles rapid-fire requests to empty responses — the
collector paces itself and treats an empty payload as "not retained".

Indicators: unemployment rate (`UNE_DEAP_SEX_AGE_RT_A`),
employment-to-population (`EMP_DWAP_SEX_AGE_RT_A`), participation
(`EAP_DWAP_SEX_AGE_RT_A`), employment level (`EMP_TEMP_SEX_AGE_NB_A`),
time-related underemployment (`EMP_XTRU_SEX_AGE_RT_A`), informal employment
share (`SDG_0831_SEX_ECO_RT_A`), average monthly earnings by currency
(`EAR_EMTA_SEX_CUR_NB_A`, split into `_lcu` / `_ppp` series). Totals only
(`SEX_T`, all-ages / whole-economy classifications); duplicate country-years
across survey vintages are deduplicated deterministically, keeping the
longest-spanning source (recorded per row in `ilo_source`).

**Absences that are data:** Iran reports neither the SDG informality
indicator nor earnings — the API returns a header-only payload. The paper
proxies Iranian informality with the World Bank's modelled vulnerable
employment (`SL.EMP.VULN.ZS`, §9) and says so.

**Verified:** the tidy CSV is rebuilt from the raw payloads to zero
deviation; all rates are bounded to [0,100],
requires all three countries present, rejects duplicate country-years, and
requires the Iranian unemployment rate to sit in a 5–25% band (a youth rate
sneaking in as the headline would not). the trace (§7) follows one figure
from raw bytes to the tidy CSV.

---

## 9. `sources/worldbank_annual_multi.csv` — growth, PPP, labor, energy, IRN/EGY/TUR

**3,462 rows · annual · 1960 → 2025 · collected 12 August 2026**

Same API, method and JSON shape as §4
(`fetch_multi` pass). 23 indicator codes for Iran, Egypt and Türkiye plus
three Iran-only trade codes for the counterfactual (food import share
`TM.VAL.FOOD.ZS.UN`, merchandise imports `TM.VAL.MRCH.CD.WT`, G&S imports
`NE.IMP.GNFS.CD`). Raw files `raw/worldbank_multi_{ISO3}_{CODE}.json`.
`series_id = {slug}_{iso3lower}`. Note `PA.NUS.PPPC.RF` (price level ratio)
was archived by the World Bank; it is recomputed as
`PA.NUS.PPP / PA.NUS.FCRF`.

**A caveat that matters:** the World Bank converts Iranian national accounts
to dollars at administered exchange rates, which inflates recent
dollar-denominated aggregates (`NE.IMP.GNFS.CD`, the recent
`imports_pct_gdp`). The scenario analysis anchors on the
customs merchandise series instead, and the paper's §16.7 records the issue.

**Verified:** both WDI files are rebuilt from raw payloads (core exact;
multi to 1e-12 relative), and the checks and the trace (§8) additionally assert that the Iranian rows of this file are
**bit-equal to the frozen core file** wherever indicator and year overlap —
412 shared rows, zero deviation.

---

## 10. The 12 August 2026 collection pass — what was and was not reachable

Every source endpoint was probed before collection: the World Bank API and
document hosts, Yahoo, Syracuse, TGJU, ILOSTAT and GitHub all answered.
`amar.org.ir`, `cbi.ir`, `tsd.cbi.ir` and `tsetmc.com` still offered nothing an
automated collector could rely on, so the best-effort SCI/CBI crawler recorded
the attempt in `results/iran_official_inventory.csv` and retrieved nothing. It
deliberately never writes into `data/sources/`: promoting a parsed SCI CPI to
the model target is a human decision.

---

## Reproducing the collection

Nothing needs re-collecting — that is the point of freezing. The collectors
exist again (`01`–`07`, `17`); each refuses to run against the frozen
manifest, `--amend` fetches only files the snapshot does not hold,
`--refresh` deliberately overwrites, and `--check` proves offline that the
shipped CSVs are what these parsers produce from the retained payloads.
Expect a re-collection to differ from this snapshot: TGJU keeps quoting, the
World Bank revises, and the Pink Sheet rotates its workbook. Any figure in
`iran-inflation.pdf` is computed from `data/` at build time, so a re-collection
**will** move published numbers. Any re-collection must be re-frozen deliberately and the manifest re-recorded.
