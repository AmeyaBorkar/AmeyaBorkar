# Nifty Pricing Mirror

> Live spot-vs-nearest-futures basis surface for any NSE index, powered by the Groww trading API. CLI plus a small dashboard server; classifies each constituent as PREMIUM / DISCOUNT / FLAT and annualises the basis for quick term-structure reads.

**Repository:** [`AmeyaBorkar/Nifty-Pricing-Mirror`](https://github.com/AmeyaBorkar/Nifty-Pricing-Mirror)  
**Category:** FinTech / Derivatives  
**Visibility:** Public  
**Primary language:** Python  
**Default branch:** `main`  
**License:** —  
**Created:** 2026-05-02  
**Last pushed:** 2026-05-02  
**Metadata updated:** 2026-05-02  
**Size (GitHub reported):** 40 KB  

---

## What it is (one-paragraph version)

Live spot-vs-nearest-futures basis surface for any NSE index, powered by the Groww trading API. CLI plus a small dashboard server; classifies each constituent as PREMIUM / DISCOUNT / FLAT and annualises the basis for quick term-structure reads.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Python | 45,654 | 73.0% |
| CSS | 6,793 | 10.9% |
| JavaScript | 6,573 | 10.5% |
| HTML | 2,886 | 4.6% |
| PowerShell | 636 | 1.0% |

## File tree

- Total entries indexed: **21** (19 files, 2 directories)

```
.env.example  (467 B)
.gitignore  (249 B)
__main__.py  (276 B)
requirements.txt  (95 B)
run.ps1  (636 B)
smoke_test.py  (3 KB)
nifty_pricing_mirror/    [13 files]
  nifty_pricing_mirror/__init__.py
  nifty_pricing_mirror/cli.py
  nifty_pricing_mirror/config.py
  nifty_pricing_mirror/csv_export.py
  nifty_pricing_mirror/display.py
  nifty_pricing_mirror/groww_client.py
  nifty_pricing_mirror/instruments.py
  nifty_pricing_mirror/pricing.py
  nifty_pricing_mirror/server.py
  nifty_pricing_mirror/static/app.js
  nifty_pricing_mirror/static/index.html
  nifty_pricing_mirror/static/styles.css
  nifty_pricing_mirror/universe.py
```

## Module-by-module summary

*(There is no README on the default branch yet. The summary below is reconstructed from the source files.)*

| Module | Role |
|--------|------|
| `cli.py` | Entry point. Wires auth → instrument resolution → live basis loop. Exposes `nifty-mirror` CLI with `--index`, `--interval`, `--once`, `--symbols-file` flags. Uses `rich` for the live console UI. |
| `config.py` | `Settings` dataclass — environment-driven config (Groww API key, refresh interval, output dirs). |
| `universe.py` | Bundled NSE index universes (NIFTY 50, etc.) loaded from packaged symbol lists. |
| `instruments.py` | Resolves each underlying symbol to its **(spot, nearest-future)** instrument pair against the Groww instrument master. `InstrumentPair` data class; `InstrumentsRepo` for caching. |
| `groww_client.py` | Authenticated HTTP client for the Groww trading API; raises `AuthenticationError` on token failure; exposes LTP fetches needed by the pricing engine. |
| `pricing.py` | The core derivative math. `PricingEngine` pairs spot & nearest-futures LTPs and produces an `IndexSnapshot` of `PriceRow` records — each carries `basis = future − spot`, `basis_pct`, `annualised_pct = basis_pct × (365 / dte)`, and a `Stance` of `PREMIUM` / `DISCOUNT` / `FLAT` / `UNKNOWN`. |
| `display.py` | `LiveSurface` — `rich`-based terminal renderer that re-paints the basis surface in place. |
| `csv_export.py` | `write_snapshot` (single tick) and `append_history` (running log) for downstream analysis. |
| `server.py` | Optional `DashboardServer` — small Flask-style HTTP service that exposes the same snapshot to a browser. |
| `static/` | Vanilla HTML/CSS/JS dashboard UI — `index.html`, `styles.css`, `app.js`. |
| `smoke_test.py` | End-to-end sanity check used to verify auth + basis pipeline without entering the live loop. |
| `run.ps1` | Windows convenience launcher (PowerShell). |
| `.env.example` | Template for the Groww API credential / token env vars. |

## What the output looks like (in concept)

For each constituent of the chosen NSE index, on every tick:

```
SYMBOL    SPOT     FUTURE   EXPIRY      DTE   BASIS    BASIS%    ANNUALISED%   STANCE
RELIANCE  2,941.5  2,952.7  2026-05-29  27    +11.20   +0.38%    +5.15%        PREMIUM
TCS       3,512.0  3,508.4  2026-05-29  27    -3.60    -0.10%    -1.39%        DISCOUNT
...
```

Aggregates printed alongside: `avg_basis_pct`, `avg_annualised_pct`, `premium_count` — a quick read on whether the index term structure is leaning bullish (broad premium) or bearish (broad discount).

## README (verbatim)

> *(no README on default branch — see module-by-module summary above)*

---

*Generated 2026-05-02 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/Nifty-Pricing-Mirror*
