# bayesian-reliability: Plan

Goal: a public, end-to-end reliability project that shows **a modeling result on real
data** ("modeled X → found Y"), a **deployed ML model in the cloud**, **forecasting with
uncertainty**, and **Bayesian inference**. It mirrors the warranty work in a private repo
(same methods, public data).

## Data

**Backblaze Drive Stats** (primary). One row per drive per day: `date`, `serial_number`,
`model`, `capacity_bytes`, `failure` (1 on the day a drive fails), SMART attributes.
Released quarterly as zipped CSVs since 2013; tens of GB in total.

- Lifetime table, one row per drive: first/last day seen, failed (0/1), model, capacity.
- **Age** from `smart_9_raw` (power-on hours) at first observation. A drive first seen at
  age > 0 is **left-truncated**: it entered the risk set late, so its early life is unobserved.
- Drives still running at the end of the window are **right-censored**.
- Ignoring either biases the Weibull shape β. Phase 1 measures that bias.
- Gotchas to check: drives that disappear without failing (migrations, so treat as
  censored), reused serials, models with tiny counts, SMART 9 resets or missing values.

**NASA C-MAPSS** (Phase 6). Simulated turbofan run-to-failure with sensor traces, used for
a remaining-useful-life module.

## Phases

**Phase 0: Scaffold + core estimation** ← now
- [ ] Repo, pre-commit, CI
- [ ] `weibull.py`: log-likelihood with right censoring + left truncation, MLE fit
- [ ] Parameter-recovery tests on simulated data
- [ ] Download script for N quarters of Backblaze → `data/raw/`; checksum + manifest

**Phase 1: Data layer + baselines**
- [ ] DuckDB ingestion: daily CSVs → Parquet → `drive_lifetimes` table (SQL, tested)
- [ ] Data-quality report: counts per model, censoring %, truncation %, anomalies
- [ ] Kaplan–Meier curves per model (with left truncation)
- [ ] Weibull MLE per model: naive vs. censoring-aware vs. censoring + truncation; show the bias
- [ ] Naive forecast baseline: trailing annualized failure rate × drives in service

**Phase 2: Bayesian Weibull (single model)**
- [ ] PyMC model with censoring + truncation (custom logp via `pm.Potential` or `pm.Censored`)
- [ ] Prior predictive check, priors on β and on 1-year failure probability (interpretable)
- [ ] Diagnostics: R-hat < 1.01, ESS > 400, no divergences; posterior predictive check
- [ ] Recovery test against simulated data (same generator as Phase 0)

**Phase 3: Hierarchical model + forecasting + backtest**
- [ ] Partial pooling across drive models (grouped by manufacturer / capacity class)
- [ ] Forecast failures next quarter for the fleet in service, with 80/95% intervals
- [ ] Rolling backtest: fit through quarter Q, forecast Q+1. MAE vs. naive, interval coverage
- [ ] **Write-up:** which models are wearing out, how sure we are, and how much better the
      forecast is than the naive rate. This is the headline result

**Phase 4: Serve + deploy**
- [ ] FastAPI: `/models`, `/models/{model}/survival?age=`, `/forecast?horizon=`, `/health`
- [ ] Posterior summaries / draws stored as artifacts (no sampling at request time)
- [ ] Docker image; deploy to **Azure Container Apps**; artifacts in Azure Blob Storage
- [ ] GitHub Actions: build + push image, deploy on tag

**Phase 5: MLOps**
- [ ] Scheduled retrain when a new quarter is released (GitHub Actions)
- [ ] Monitoring: log forecast vs. actual per quarter; alert on coverage drift
- [ ] Model card: data, assumptions, known limits

**Phase 6: C-MAPSS remaining-useful-life module**
- [ ] Baseline (gradient boosting on windowed sensor features), then a Bayesian or survival
      variant; same API pattern

## Definition of done (v1)

- Public README with the result, plots and a live API link
- Backtest shows the hierarchical model beats the naive baseline, with calibrated intervals
- CI green; deploy and retrain automated
