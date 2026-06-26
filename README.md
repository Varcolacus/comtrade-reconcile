# comtrade-reconcile

[![validate](https://github.com/Varcolacus/comtrade-reconcile/actions/workflows/validate.yml/badge.svg)](https://github.com/Varcolacus/comtrade-reconcile/actions/workflows/validate.yml)

**A share-faithful reconstruction of bilateral trade from raw UN Comtrade (BACI-style) — and a nowcast
for the years BACI hasn't released yet.**

> Scope claim, stated precisely: this reproduces BACI's **shares, ranks and concentration** (validated
> below), *not* its exact levels — current Comtrade runs ~1.5–1.8× above BACI's published values (see
> "the level offset", diagnosed). It is Comtrade-mirror reconciliation in BACI's spirit, not a bit-for-bit
> BACI replica.

CEPII's [BACI](http://www.cepii.fr/CEPII/en/bdd_modele/bdd_modele_item.asp?id=37) is the standard
"clean" bilateral trade dataset, but it lags ~1.5 years. This is a small, self-contained pipeline that
reconstructs the same thing from **raw UN Comtrade** — matching the two mirror reports of every flow,
correcting CIF/FOB, weighting reporters by reliability, and reconciling — then **validates the result
against official BACI** and uses it to nowcast the missing recent years.

Built as the trade engine behind the [critical-materials-atlas](https://varcolacus.github.io/critical-materials-atlas/)
(where it powers the 2025–2026 layers), but it is general — it works for any HS6 codes.

Method follows **Gaulier & Zignago (2010)**, *BACI: International Trade Database at the Product-Level*,
CEPII WP 2010-23.

## The problem
Every flow *i → j* is reported twice: the exporter declares it **FOB**, the importer declares its mirror
**CIF** (with freight + insurance). The two rarely agree — valuation, misreporting, timing, one-sided
reporting. A single reconciled value has to be recovered from these noisy double reports.

## The method (`reconcile.py`)
1. **Match mirrors** — pair `x_fob` (exporter) and `m_cif` (importer) for each (i, j, HS6).
2. **CIF → FOB** — deflate imports to an FOB basis. *Finding:* the gravity regression BACI uses to
   estimate CIF rates is **not identifiable on a narrow product slice** (R² ≈ 0.01 on 31 codes — at HS6
   the M/X ratio is dominated by valuation noise, not transport). BACI estimates it on the full
   ~5,000-product universe; here we fall back to a robust per-product median markup. An honest negative
   result, kept rather than hidden.
3. **Reliability weights** — each reporter's quality from a **variance-components** decomposition of the
   mirror discrepancy: E[(ln x_fob − ln m_fob)²] = var_i + var_j (OLS on reporter dummies).
4. **Reconcile** — two-sided flows → **inverse-variance average on logs**; one-sided → the single
   report (FOB-adjusted if it is the importer's).

## Validation against official BACI (`validate.py`)
Validated on **what matters downstream — shares and concentration**, not just a global correlation:

Validated on **both sides** (exporter and importer), since the atlas shows both.

**2024** (the newest, still-settling BACI year):

| metric | exporter | importer |
|---|---|---|
| top-1 match | **25 / 30** | **22 / 30** |
| top-3 overlap (mean) | 2.57 / 3 | 2.27 / 3 |
| share MAE | **3.5%** (med 3.2%) | 4.2% (med 4.1%) |
| HHI correlation | 0.92 | 0.97 |

Flow-level log-value correlation 0.975 (21.7k flows); level ratio ~1.8× (disclosed offset, below).

**2022** (a fully settled year):

| metric | exporter | importer |
|---|---|---|
| top-1 match | 22 / 30 | 21 / 30 |
| share MAE | 3.9% | 4.5% |
| HHI correlation | 0.885 | 0.907 |

Level ratio ~1.5×.

### The level offset — diagnosed honestly
Our reconciled totals run **~1.8× BACI for 2024 and ~1.5× for the settled year 2022**. A flow-level
diagnostic shows it is **not a method artefact**: on flows where both raw Comtrade reports and BACI
exist, the raw exporter report is already **1.94×** BACI and the importer **1.81×** — the raw data
itself sits well above BACI, and our reconciled value (**1.67×**) falls *between and below* the two raw
reports. So the engine faithfully reconciles what Comtrade reports; **current Comtrade simply runs above
BACI's published values.** The gap has two parts: a **recency** component (2024's 1.8× vs 2022's 1.5× —
revisions and late filers still settling into a newly-released BACI year) and a **persistent ~1.5×** that
reflects BACI's own downward adjustments — outlier/quality filtering and reconciliation choices that pull
below the raw mirror reports. Either way it is a near-constant multiple, so it **cancels out of shares**
— which is exactly what is validated above and what the atlas shows. (For the nowcast years, levels are
calibrated back to BACI's scale per material; shares are untouched.)

## Nowcasts
- **2025** (`build_recon_flows.py`) — full reconciliation of partial 2025 Comtrade, level-calibrated to
  BACI 2024. Provisional.
- **2026** (`build_2026_nowcast.py`) — only ~Q1 monthly Comtrade exists, so 2025's reconciled structure
  is carried forward and scaled per material by reporter-matched Q1 export momentum blended with the
  World Bank Pink Sheet price change. Shares stay at 2025; only levels tilt. Directional, not bilateral.

## Run
```bash
export ATLAS_ROOT=/path/to/critical-materials-atlas   # provides out/data.json, raw/baci/country_codes…
COMTRADE_KEY=<key> python pull_comtrade.py 2024        # raw bilateral pull (one call per code×flow)
python reconcile.py 2024                               # -> recon_2024.csv
python validate.py 2024                                # vs BACI (results/ has saved output)
```
`COMTRADE_KEY` is read from the environment — never hardcode or commit it.

## Inputs (all public)
UN Comtrade API · CEPII `dist_cepii` (gravity: distance, contiguity) · CEPII BACI `country_codes`
(M49 ↔ ISO) · World Bank [Pink Sheet](https://www.worldbank.org/en/research/commodity-markets)
(commodity prices). Raw downloads are gitignored; `results/` holds the committed validation output.

— Independent work, public data only.
