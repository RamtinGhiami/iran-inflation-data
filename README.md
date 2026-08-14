# Forecasting Iranian Inflation from Mixed-Frequency Market Signals

Paper and data for a study of Iranian inflation: a validated monthly panel of
observable inflation predictors, an interpretable machine-learning attribution
with an out-of-sample ablation, and the limitations that follow.

- **`iran-inflation.pdf`** — the paper (IEEE conference format, 6 pages).
- **`data/`** — the frozen data snapshot the paper is computed from.
- **`results/`** — model output and diagnostics the paper's numbers come from.
- **`DATA.md`** — what the data is, where it came from, and what is missing.

## Layout

```
iran-inflation.pdf          the paper
DATA.md                     data documentation
data/
  PROVENANCE.md             where every dataset came from and how it was checked
  MANIFEST.json             SHA-256 of all snapshot files (140 files, frozen)
  dataset_monthly.csv       816 months x 113 columns  <- the modelling dataset
  sources/                  one tidy CSV per source
  raw/                      untouched downloads  <- the evidence
results/
  model/                    forecasts, scores, importance, ablation, diagnostics
  audit_issues.csv          open data-quality findings
```

## The data is frozen

Collected in two passes (8 and 12 August 2026) and fixed. `data/MANIFEST.json`
records every file in `data/sources/` and `data/raw/` with a SHA-256
(140 files). Every number in the paper is computed from this snapshot.

## Conventions that matter

**The Solar Hijri month is the grid.** Iranian CPI is defined on it; daily and
Gregorian-monthly series are aggregated *up* into it. Solar Hijri months begin
on the 19th–22nd of a Gregorian month, never the 1st.

**Prices are in rial**, not toman. **Nothing is destructively transformed** —
`_logdiff` and `_yoy` columns are added; levels stay. **Gaps are explicit** —
a missing observation is a `NaN` in a present row; sixteen edge months are
entirely empty and must be dropped before fitting.

## Headline results (as reported in the paper)

Across 56 rolling-origin one-step-ahead forecasts, Ridge improves on the random
walk by 24% in RMSE; a Diebold–Mariano test does not find the margin significant
(p = 0.101), and no model survives a Holm correction. Currency-linked variables
carry about 60% of permutation importance, but an ablation shows the gold and
coin block adds nothing once the exchange rate is included, so the expectations
channel suggested by the importance ranking is not supported. Annual inflation
shifted from a chronic 18% regime (1973–2017) to 37% after 2018 (Chow F = 29.5).
An ex-post conditional projection over the 17 months not covered by the index
gives about 44% annualised (bootstrap 95% interval 40–67%), well below the
roughly 88% year-on-year rate reported for mid-1405; the two figures still need
to be reconciled on a common definition and source before the gap is read as a
forecast error.

## Authors

Ramtin Ghiyami, Matin Askari, Ali Zafari, Mahdi Movahedian Moghaddam,
Kourosh Parand — Department of Computer and Data Sciences, Faculty of
Mathematical Sciences, Shahid Beheshti University, Tehran, Iran.
