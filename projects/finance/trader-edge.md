# trader-edge

> A deterministic, LLM-free pre-trade risk engine for Indian equities and F&O. Given a long-bracket trade idea (symbol, entry, target, stop, horizon), it computes four probabilities/numbers entirely from observable market data: the chance the stop is hit by realized-volatility noise alone (reflection-principle first-passage on GBM), the chance of touching target/stop implied by the live option chain (IV-smile fit + first-passage at each barrier's IV), the joint "who-hits-first" probabilities (vectorized NumPy Monte Carlo at the chain's ATM implied vol), and a true expected value in rupees net of brokerage and slippage. It then emits concrete alternatives (widen the stop, tighten the target, or skip) and logs every verdict to CSV for personal calibration. Driven via a Click CLI, a FastAPI REST API, and a vanilla-JS web UI — all backed by the same pure-function math core.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/trader-edge |
| **Visibility** | Public |
| **Category** | finance |
| **Primary language(s)** | Python (NumPy / SciPy / FastAPI / Click); small vanilla JS/HTML/CSS frontend |
| **Local path** | `C:\Users\ameya\Documents\GrowwProject` |
| **Default branch** | `main` |
| **Lines of code (computed)** | **2,929** lines of Python (2,406 in `trader_edge/`, 523 in `tests/`) + **812** lines of frontend (320 JS, 372 CSS, 120 HTML). Total ≈ **3,741** lines. |
| **Source files (computed)** | 29 Python files under `trader_edge/`, 10 under `tests/` (39 `.py` total); 3 static frontend files (`app.js`, `index.html`, `style.css`) |
| **Key dependencies** | numpy ≥1.26, scipy ≥1.11, requests ≥2.31, click ≥8.1, rich ≥13.7, fastapi ≥0.110, uvicorn[standard] ≥0.27, pydantic ≥2.5, python-dotenv ≥1.0, pytest ≥7.4 |
| **License** | None — no `LICENSE` file in the repo (the stale doc and README both show "—") |
| **Last commit** | `8fe0470` — "Show calibration status banner with NOT CALIBRATED, BUILDING, ACTIVE states" (2026-05-04) |
| **Version** | `0.1.0` (`trader_edge/__init__.py`) |

---

## What it actually is

A small, self-contained Python package (`trader_edge`) that turns a trade *idea* into a probabilistic verdict using only closed-form math and numerical operations on market prices — there is **no LLM anywhere in the codebase** (confirmed: no `openai`/`anthropic`/`langchain`/`requests`-to-LLM imports; `requests` is used solely for the Groww broker REST API).

The package treats the **option chain as a free oracle**: it backs out implied volatility per strike from listed option mids, fits a smile, and uses that IV to price the probability of a cash/index trade hitting its levels. Everything reduces to three families of math:

1. **Black-Scholes** pricing, Greeks, and IV inversion (`math_core/black_scholes.py`).
2. **First-passage / barrier probabilities** — a closed-form reflection-principle formula for a single barrier, plus a vectorized Monte Carlo for the joint two-barrier ("who hits first") case (`math_core/barrier.py`).
3. **Risk-neutral density extraction** via Breeden-Litzenberger from the fitted smile (`math_core/implied_pdf.py`).

These feed an **expected-value** calculation net of costs (`math_core/ev.py`), orchestrated by the **pre-trade engine** (`analysis/pretrade.py`) which combines realized vol (history), implied vol (chain), the joint Monte Carlo, and EV into a single `PreTradeVerdict`. A second mode (`analysis/journal.py`) FIFO-pairs historical orders into round-trips and flags behavioral leaks.

Three interfaces expose the same core: a **Click CLI** (`cli.py`, the richest interface — `analyze`/`journal`/`chain`/`log`), a **FastAPI REST API** (`server/`), and a static **web UI** (`server/static/`). Data comes from either a live **Groww broker REST client** (read-only) or a deterministic **mock provider** that synthesizes internally-consistent chains and candles so the engine runs end-to-end with no credentials.

---

## Architecture & how it's structured

```
GrowwProject/                         (repo root; package is trader_edge/)
├── .env.example                      Env template: GROWW_ACCESS_TOKEN, TRADER_EDGE_PROVIDER, etc.
├── requirements.txt                  10 pinned deps (numpy, scipy, fastapi, click, rich, ...)
├── README.md                         (note: claims "31 tests" — actual is 38, see README vs code)
├── tests/                            10 files, 38 test functions, pytest
└── trader_edge/
    ├── __init__.py                   __version__ = "0.1.0"
    ├── config.py                     Constants: TRADING_DAYS_PER_YEAR=252, RISK_FREE_RATE=0.07,
    │                                 brokerage/slippage defaults, VRP vol-points
    ├── env.py                        Idempotent .env loader (python-dotenv, OS env wins)
    ├── cli.py                        Click CLI: analyze / journal / chain / log   (345 lines)
    │
    ├── math_core/                    ── PURE FUNCTIONS, no I/O ──
    │   ├── black_scholes.py          Quote dataclass; price, vega, delta, gamma, theta_per_day;
    │   │                             implied_vol (Newton + bisection fallback)
    │   ├── volatility.py             close_to_close, parkinson, garman_klass, ewma (annualized σ)
    │   ├── barrier.py                prob_hit_single_barrier (reflection principle);
    │   │                             joint_barrier_probs (NumPy Monte Carlo, JointBarrierResult)
    │   ├── implied_pdf.py            ChainSlice; implied_vol_smile, fit_smile (quadratic in
    │   │                             log-moneyness), terminal_pdf (Breeden-Litzenberger),
    │   │                             prob_above/below_at_expiry, iv_at_strike
    │   └── ev.py                     expected_value -> EVResult (gross/net EV, headline & true R:R)
    │
    ├── api/                          ── DATA PROVIDERS ──
    │   ├── base.py                   DataProvider ABC (get_ltp, get_candles, get_option_chain,
    │   │                             get_recent_orders)
    │   ├── models.py                 Candle, OptionRow (+ call_mid/put_mid), OptionChain, Order
    │   ├── groww_client.py           Live Groww REST client (read-only, Bearer auth)
    │   └── mock_provider.py          Deterministic offline provider (BS-priced synthetic chains)
    │
    ├── analysis/                     ── ORCHESTRATION ──
    │   ├── pretrade.py               analyze_trade -> PreTradeVerdict (THE engine; 263 lines)
    │   ├── suggestions.py            suggest_alternatives (widen stop / tighten target / skip)
    │   ├── trade_log.py              Append-only CSV log + Pearson calibration vs actuals
    │   └── journal.py                pair_trades (FIFO) + journal -> JournalSummary + leak rules
    │
    └── server/                       ── HTTP LAYER ──
        ├── main.py                   FastAPI app, CORS, mounts /static, serves index.html at /
        ├── dependencies.py           lru_cache'd provider resolution (mock/groww/auto)
        ├── schemas.py                Pydantic v2 request/response models
        ├── routes/                   health.py (/health,/symbols), analyze.py, chain.py, journal.py
        └── static/                   index.html, app.js, style.css (vanilla, no framework)
```

### Data flow (the `analyze` path)

```
CLI/REST request (symbol, entry, target, stop, horizon_days, qty)
        │
        ▼
_resolve_provider / get_provider ──► MockProvider | GrowwClient   (api/)
        │
        ▼  analysis/pretrade.analyze_trade(provider, request)
   ┌────┴──────────────────────────────────────────────────────────┐
   │ 1. provider.get_candles() → garman_klass()  → realized σ        │
   │      → prob_hit_single_barrier()  → P(stop hit by noise)        │
   │      → binary-search noise-safe stop (~30% hit)                 │
   │ 2. provider.get_option_chain() → ChainSlice                     │
   │      → implied_vol_smile() + fit_smile() → ATM/target/stop IV   │
   │      → prob_hit_single_barrier() at each IV  → P(touch …)       │
   │      → terminal_pdf() (Breeden-Litzenberger) → P(… at expiry)   │
   │ 3. joint_barrier_probs(σ = ATM IV)  → P(target/stop/neither 1st)│
   │ 4. expected_value() twice: risk-neutral, and VRP-adjusted "RW"  │
   └────┬──────────────────────────────────────────────────────────┘
        ▼
   PreTradeVerdict ─► suggest_alternatives() ─► render (rich panel / JSON)
                  └─► trade_log.append_verdict() → ~/.trader_edge/log.csv
```

---

## Code walkthrough (the important parts)

### `math_core/black_scholes.py` (121 lines)
European options on a non-dividend-paying underlying. `Quote` is a frozen dataclass (`spot, strike, t_years, rate, is_call`). `_d1_d2` computes the standard `d1 = (ln(S/K) + (r + σ²/2)T) / (σ√T)`, `d2 = d1 − σ√T`. `price` is exact BS call/put. Greeks: `vega = S·φ(d1)·√T`; `delta` = `N(d1)` (call) / `N(d1)−1` (put); `gamma = φ(d1)/(S·σ·√T)`; `theta_per_day` returns the annual theta divided by 365 (note: **365**, calendar days, not the 252 trading days used elsewhere).

`implied_vol` is the workhorse: it first checks the **no-arbitrage bounds** (`max(0, S − Ke^{−rT}) ≤ C ≤ S` for calls; the mirror for puts) and raises `ValueError` if the market price is outside `[bound − 1e-6, bound + 1e-6]`. It then runs **Newton-Raphson** seeded at σ=0.30, stepping by `(price − market)/vega`, bailing if σ leaves `(0, 5.0]` or vega < 1e-10, then falls back to **200-iteration bisection** on `[1e-4, 5.0]`. Tolerance `1e-6`.

### `math_core/volatility.py` (64 lines)
Four annualized realized-vol estimators, all multiplying daily variance by `periods_per_year` (default 252):
- `close_to_close`: sample std (`ddof=1`) of log-returns × √252.
- `parkinson`: `σ²_daily = mean(ln(H/L)²) / (4·ln2)`.
- `garman_klass`: `mean(0.5·ln(H/L)² − (2ln2−1)·ln(C/O)²)`.
- `ewma`: RiskMetrics-style exponentially-weighted (λ=0.94) mean of squared log-returns.

The engine prefers `garman_klass` (OHLC) and falls back to `close_to_close` if OHLC is degenerate. `parkinson`/`ewma` are implemented and tested but **not wired into the pretrade engine**.

### `math_core/barrier.py` (148 lines)
Two functions, the mathematical heart:

- **`prob_hit_single_barrier`** — closed-form first-passage via the **reflection principle** on arithmetic BM of `log(S/S₀)` with drift `μ_x = (drift − σ²/2)` (default `drift=0`). With signed log-distance `b = ln(barrier/spot)` and `σ√T`:
  - lower barrier (`b < 0`): `P = N((b − μ_x T)/σ√T) + exp(2μ_x b/σ²)·N((b + μ_x T)/σ√T)`
  - upper barrier (`b > 0`): the sign-flipped mirror.
  Result clamped to `[0,1]`. Horizon converts via `TRADING_DAYS_PER_YEAR`.

- **`joint_barrier_probs`** — vectorized NumPy **Monte Carlo** for the two-sided case (target above, stop below). Defaults: `n_paths=25_000`, `steps_per_day=8`, `seed=42` (so it's **deterministic**), `n_steps = max(horizon_days·8, 50)`. Simulates log-increments in `float32` (`mu_dt + σ√dt·z`), `cumsum`s to log-paths, finds the first step each barrier is crossed via `argmax` over boolean masks, and decides target-first/stop-first/neither. Returns `JointBarrierResult` with the three probabilities (which sum to 1 — tested) plus `expected_exit_days` conditional on early exit. Validates `stop < spot < target`.

### `math_core/implied_pdf.py` (148 lines)
- `ChainSlice` (frozen) sorts strikes/prices on construction.
- `implied_vol_smile` inverts BS **only on the OTM side** (calls for K ≥ spot, puts for K < spot — better bid/ask), skips mids < 0.05 and IVs outside `(0.01, 4.0)`.
- `fit_smile` fits `IV(K) = a + b·x + c·x²` where `x = ln(K/spot)` (quadratic in log-moneyness via `np.polyfit`, degree 2). Returns a callable. README/code both flag SVI as a future upgrade.
- `terminal_pdf` — **Breeden-Litzenberger**: reprices calls on a dense even grid (`n_grid=401`, clipped to `[0.4·spot, 2.5·spot]` ∩ listed range) using the smile, takes a **central-difference 2nd derivative**, multiplies by `e^{rT}`, clips negatives, and renormalizes via `np.trapezoid` so it integrates to ~1.
- `prob_above_at_expiry` / `prob_below_at_expiry` integrate that PDF; `iv_at_strike` evaluates the fitted smile.

### `math_core/ev.py` (72 lines)
`expected_value` for a long bracket. `win = target − entry`, `loss = stop − entry` (negative), `flat = spot_at_no_hit − entry` (defaults to entry → flat=0). `gross = p_target·win + p_stop·loss + p_neither·flat`. **Costs** = `2·brokerage_per_share + slippage`, where slippage = `(slippage_bps/10_000)·entry·2` (round-trip). Reports `headline_rr = win/|loss|` and a `true_rr` that weights the EV-positive vs EV-negative parts of the distribution. Returns `EVResult`.

### `analysis/pretrade.py` (263 lines) — the engine
`analyze_trade` validates `stop < entry < target`, pulls `history_days·2` of candles, computes realized σ (garman_klass → close_to_close fallback), then `P(stop hit by noise)` and a **binary-searched noise-safe stop** (`_binary_search_safe_stop`, 60 iterations, targets `SAFE_NOISE_HIT_THRESHOLD = 0.30`). It builds the `ImpliedReport` from the chain (ATM/target/stop IV, risk-neutral and VRP-adjusted touch probs, expiry-density probs), runs `joint_barrier_probs` at the **chain's ATM IV** with horizon capped at days-to-expiry, and computes EV twice:
- **risk-neutral** EV from the joint MC probs;
- **real-world** EV by subtracting the variance-risk-premium (`INDEX_VRP_VOL_POINTS=4.0` for index symbols `{NIFTY, BANKNIFTY, FINNIFTY, SENSEX}`, else `SINGLE_NAME_VRP_VOL_POINTS=1.5`) from IV (floored at 5%), re-running the MC with `seed=43`.

It appends diagnostic `notes` (implied vs realized vol divergence, risky stop, non-positive RW EV).

### `analysis/suggestions.py` (95 lines)
Given a verdict, proposes up to three alternatives by **re-running `analyze_trade`** on modified requests: (1) widen the stop to the noise-safe level (rounded to ₹0.50) if `p_stop_hit_noise > 0.40`; (2) move the target to the entry-target midpoint if `p_touch_target < 0.20`; (3) recommend "skip" if real-world EV ≤ 0.

### `analysis/trade_log.py` (156 lines)
Append-only CSV (25 columns incl. blank `actual_outcome`/`actual_pnl` for manual fill). `default_log_path()` → `~/.trader_edge/log.csv` or `$TRADER_EDGE_LOG_PATH`. `calibration()` computes a **Pearson correlation** between predicted EV/share and actual P&L/share over closed (non-skipped, outcome-filled) trades, plus direction-accuracy counts — but only computes the correlation once there are **≥ 5** closed trades.

### `analysis/journal.py` (171 lines)
`pair_trades` walks `EXECUTED` orders chronologically and **FIFO-pairs** BUY/SELL into `TradePair`s per symbol (handling partial fills by re-inserting stub orders), computing P&L for LONG and SHORT round-trips. `journal()` aggregates win rate, avg win/loss, expectancy, profit factor, median hold days, and emits **deterministic "leak" rules** (cut-winners/hold-losers, disposition effect via hold-time asymmetry, profit factor < 1, negative expectancy).

### `api/groww_client.py` (135 lines) & `mock_provider.py` (135 lines)
The Groww client hits four **read-only** REST endpoints with a Bearer token + `X-API-VERSION: 1.0` header (base `https://api.groww.in`); order placement is deliberately absent. The mock provider builds **internally-consistent** chains by BS-pricing a synthetic smile (`atm_iv + skew·x + curv·x²`, skew −0.20, curv 0.6) so IV extraction round-trips cleanly, and simulates GBM candles — with fixed seeds, so the whole demo is deterministic. Known mock symbols: RELIANCE, NIFTY, BANKNIFTY, TCS, HDFCBANK.

### `cli.py` (345 lines) & `server/`
The CLI uses Click + Rich (panels/tables). `analyze` renders the four-section verdict and alternatives, then logs unless `--no-log`. `log` shows a Rich table + a calibration **status banner** (NOT CALIBRATED / BUILDING / ACTIVE — the subject of the last commit). The FastAPI app exposes the same logic under `/api/v1` and serves the static UI; `app.js` is plain `fetch` calls with no framework and an explicit "No reasoning is delegated to a language model" note in the UI.

---

## Quantitative methods — the actual math implemented

| Method | Formula (as coded) | Location |
|---|---|---|
| **Black-Scholes price** | `C = S·N(d1) − Ke^{−rT}·N(d2)`; put by parity | `black_scholes.py: price` |
| **d1/d2** | `d1 = (ln(S/K)+(r+σ²/2)T)/(σ√T)`, `d2 = d1 − σ√T` | `black_scholes.py: _d1_d2` |
| **Greeks** | vega `S·φ(d1)√T`; delta `N(d1)` / `N(d1)−1`; gamma `φ(d1)/(Sσ√T)`; theta (annual)/365 | `black_scholes.py` |
| **Implied vol** | Newton on vega, seed σ=0.30, then bisection on `[1e-4, 5]`; no-arb bounds enforced | `black_scholes.py: implied_vol` |
| **Realized vol** | close-to-close `std(Δln C)·√252`; Parkinson `√(mean(ln(H/L)²)/(4ln2)·252)`; Garman-Klass; EWMA(λ=0.94) | `volatility.py` |
| **Single-barrier first passage** | reflection principle on `log(S/S₀)`, drift `μ−σ²/2`: `N((b−μ_xT)/σ√T)+exp(2μ_xb/σ²)N((b+μ_xT)/σ√T)` | `barrier.py: prob_hit_single_barrier` |
| **Joint two-barrier (who-first)** | NumPy Monte Carlo, 25k paths × ≥50 steps, GBM log-paths, argmax first-crossing; deterministic (seed 42) | `barrier.py: joint_barrier_probs` |
| **IV smile** | `IV(K)=a+b·ln(K/S)+c·ln(K/S)²` via `np.polyfit` deg 2, OTM side only | `implied_pdf.py: fit_smile` |
| **Risk-neutral terminal PDF** | Breeden-Litzenberger `f(K)=e^{rT}·∂²C/∂K²` via central difference, clip+renormalize (trapezoid) | `implied_pdf.py: terminal_pdf` |
| **Expiry probabilities** | integrate the terminal PDF above/below a level (trapezoid) | `implied_pdf.py: prob_above/below_at_expiry` |
| **Expected value** | `E = ΣpᵢΔᵢ − costs`; costs `= 2·brokerage + (bps/1e4)·entry·2`; headline & true R:R | `ev.py: expected_value` |
| **VRP adjustment** | `σ_RW = max(0.05, σ_implied − VRP/100)`, VRP = 4.0 (index) / 1.5 (single name) vol-points | `pretrade.py`, `config.py` |
| **Noise-safe stop** | 60-iteration binary search for the stop where single-barrier hit prob ≈ 0.30 | `pretrade.py: _binary_search_safe_stop` |
| **Calibration** | Pearson corr of predicted EV/sh vs actual P&L/sh (≥5 closed trades) | `trade_log.py: calibration` |

Constants (`config.py`): `TRADING_DAYS_PER_YEAR=252`, `RISK_FREE_RATE=0.07`, `DEFAULT_BROKERAGE_PER_SHARE=0.20`, `DEFAULT_SLIPPAGE_BPS=3`, `INDEX_VRP_VOL_POINTS=4.0`, `SINGLE_NAME_VRP_VOL_POINTS=1.5`.

---

## Data sources & integrations

- **Groww broker REST API** (`api/groww_client.py`) — live source for Indian equities/F&O. Bearer-token auth (`GROWW_ACCESS_TOKEN`), `Accept: application/json`, `X-API-VERSION: 1.0`, base `https://api.groww.in` (overridable via `GROWW_API_BASE_URL`). Four **read-only** GET endpoints:
  - `/v1/live-data/ltp` (last traded price)
  - `/v1/historical/candle/range` (daily candles, `interval_in_minutes=1440`)
  - `/v1/live-data/option-chain` (the IV oracle)
  - `/v1/order/list` (order history for the journal)
  Order placement is intentionally not implemented (read-only by design).
- **Mock provider** (`api/mock_provider.py`) — offline, deterministic, BS-consistent chains + GBM candles; the default when no token is present, and what every test uses.
- **No market-data vendor, no LLM API.** `requests` is used only for Groww. The "oracle" is the option chain itself, not any external model.

Provider selection precedence (`cli._resolve_provider` / `server/dependencies._get_provider`): explicit `mock`/`groww` (CLI flag or `TRADER_EDGE_PROVIDER`) wins; otherwise `auto` → Groww if `GROWW_ACCESS_TOKEN` is set, else mock.

---

## Tech stack & dependencies

From `requirements.txt` (10 deps, all `>=` pins, no `pyproject.toml` / no packaging metadata in the tree):

| Dependency | Role in this codebase |
|---|---|
| `numpy>=1.26` | All array math, Monte Carlo, smile fitting, integration |
| `scipy>=1.11` | `scipy.stats.norm` (BS/greeks/barrier); `scipy.interpolate.CubicSpline` is **imported in implied_pdf.py but never used** (smile uses `np.polyfit`) |
| `requests>=2.31` | Groww REST client only |
| `click>=8.1` | CLI command group / options |
| `rich>=13.7` | CLI panels/tables/colorized output |
| `fastapi>=0.110` | REST API + auto OpenAPI docs |
| `uvicorn[standard]>=0.27` | ASGI server to run the API |
| `pydantic>=2.5` | Request/response schemas (`field_validator`, `Field` — Pydantic v2) |
| `python-dotenv>=1.0` | `.env` loading (optional at import — guarded try/except) |
| `pytest>=7.4` | Test runner |

Frontend: hand-written `index.html` + vanilla `app.js` (no framework, `fetch`-based, 320 lines) + `style.css` (newsprint theme, 372 lines). Python target is 3.11+ (uses `X | None` syntax, `from __future__ import annotations`).

---

## Build / run / test (actual commands)

```bash
# Install
pip install -r requirements.txt

# CLI — pre-trade analysis (long bracket: SYMBOL ENTRY TARGET STOP)
python -m trader_edge.cli analyze RELIANCE 2850 3000 2800 --days 5 --qty 10
python -m trader_edge.cli analyze NIFTY 24500 25000 24300 --provider mock
python -m trader_edge.cli analyze RELIANCE 2850 3000 2800 --no-log --tag breakout

# CLI — trade journal (FIFO-paired round-trips + leaks)
python -m trader_edge.cli journal --days 30 --provider mock

# CLI — inspect the option chain
python -m trader_edge.cli chain RELIANCE --provider mock

# CLI — trade log + calibration
python -m trader_edge.cli log                # last 20 entries + calibration banner
python -m trader_edge.cli log --stats        # calibration only

# REST API + web UI
uvicorn trader_edge.server.main:app --reload --port 8000
#   http://localhost:8000/        web UI
#   http://localhost:8000/docs    OpenAPI reference
#   API under /api/v1: GET /health, GET /symbols, POST /analyze,
#                      GET /chain/{symbol}, GET /journal?days=N

# Tests
python -m pytest -q
```

CLI options on `analyze`: `--days` (default 5), `--qty` (1), `--provider {auto,mock,groww}`, `--no-log`, `--tag`. The CLI reconfigures stdout/stderr to UTF-8 on Windows so the ₹ symbol renders.

---

## Tests

`tests/` has 10 files and **38 test functions** (counted from `def test_*`; the README and stale doc both say "31"). Two use `@pytest.mark.parametrize`, so the executed-case count is higher (BS round-trip ×6, reflection-vs-MC ×5). Coverage by area:

- `test_black_scholes.py` (3) — BS↔IV round-trip near-exact (`<1e-4`) across moneyness; vega positivity; rejects below-intrinsic prices.
- `test_barrier.py` (4) — closed-form reflection matches a **Brownian-bridge-corrected** Monte Carlo to within 2.5%; joint probs sum to 1; closer target ⇒ higher P(target first); input validation.
- `test_implied_pdf.py` (5) — flat-smile recovers the input vol; `iv_at_strike` interpolation; terminal PDF integrates to ≈1; monotone/sane expiry probabilities.
- `test_volatility.py` (4) — estimators recover ground-truth σ from simulated paths.
- `test_pretrade.py` (3) — end-to-end verdict on mock data; bracket validation; tight stop triggers a widen suggestion.
- `test_journal.py` (3) — FIFO pairing (simple + partial exits); coherent summary stats.
- `test_trade_log.py` (6) — CSV header/append behavior, parent-dir creation, calibration math (correlation > 0.9 on correlated synthetic data), skipped-trade exclusion.
- `test_server.py` (6) — FastAPI `TestClient` smoke tests for all routes incl. 400 on invalid bracket.
- `test_env.py` (4) — `.env` loader idempotency and provider-override precedence.

All tests run against the deterministic `MockProvider`, so no credentials/network are needed. (`.pytest_cache` shows the suite was last run under Python 3.13 / pytest 9.0.2.)

---

## Status, completeness & notable gaps

**Working and coherent.** The math core, both providers, all three interfaces, the trade log, and the journal are all implemented and tested end-to-end. It runs offline out of the box.

Notable gaps / observations from the code:
- **Long-only.** Every analysis path (engine, EV, joint MC, suggestions, schema validation) assumes `stop < entry < target` — a long bracket. Shorts are handled only in the journal's P&L pairing, not in pre-trade analysis.
- **Unused/under-wired pieces:** `CubicSpline` is imported in `implied_pdf.py` but never used; `parkinson` and `ewma` vol estimators are tested but not invoked by the engine; `delta`/`gamma` Greeks exist but aren't surfaced in any output.
- **Single nearest expiry only** — no term structure; `get_option_chain` takes one expiry.
- **VRP is a flat constant subtraction** (4.0 / 1.5 vol-points), not estimated from data; the "real-world" EV is explicitly an approximation.
- **Theta uses /365** (calendar) while the rest of the engine annualizes on 252 trading days — internally inconsistent, though theta isn't used downstream.
- **No `LICENSE` file**, no `pyproject.toml`/packaging, no CI config, no Dockerfile in the tree.
- **Groww endpoint shapes are best-effort** — the client header comments note Groww's API is young and fields may change; payload parsing is defensive (`.get(...)` with fallbacks).
- README's example numbers (e.g. "expiry 2026-06-03", specific EVs) are illustrative, not pinned to the deterministic mock seeds.

---

## README vs. code

Mostly accurate, with a few discrepancies worth flagging:

- **Test count.** README (and the stale doc) say **"31 tests, ~6s runtime"**; the actual suite has **38 `def test_*` functions** across 10 files (more once parametrized cases are expanded). The architecture comment in the README likewise says "pytest, 31 tests."
- **Dependency list.** README's prose ("Dependencies: numpy, scipy, requests, click, rich, pytest") **omits fastapi, uvicorn, pydantic, and python-dotenv**, which are all in `requirements.txt` and required for the server. The README's own "REST API + Web UI" section does use them, so the prose summary is just incomplete.
- **Math claims verified true:** reflection-principle formula, Breeden-Litzenberger, Newton+bisection IV inversion, and the VRP description all match the code exactly. The "no LLM" claim is genuinely true (no LLM imports or calls anywhere).
- **`scipy` usage overstated by omission:** README lists scipy as a core dep (correct — `scipy.stats.norm`), but the only scipy *interpolation* import (`CubicSpline`) is dead code.
- **Stale doc note.** The pre-existing doc at `projects/trader-edge.md` is a 2026-05-16 GitHub-API snapshot that pastes the README verbatim and inherits its "31 tests" error and a since-rounded size figure; this code-derived doc supersedes it.

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\GrowwProject. Source of truth: https://github.com/AmeyaBorkar/trader-edge*
