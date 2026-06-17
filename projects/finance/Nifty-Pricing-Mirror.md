# Nifty-Pricing-Mirror

> A Python terminal/web tool that, every ~3 seconds, fetches the cash-market spot price and the nearest-expiry futures price for every constituent of an NSE index (Nifty 200 by default, Nifty 50 bundled) via the Groww trading API, then computes the spot-vs-futures **basis** (`future − spot`), its percentage, and an annualised yield, labels each name `PREMIUM` / `DISCOUNT` / `FLAT` / `UNKNOWN`, summarises the index as `CONTANGO` / `BACKWARDATION` / `MIXED`, and renders the result as a live Rich table, an optional Flask browser dashboard, and atomic CSV side-channels for Excel/Power Query.

| Field | Value |
| --- | --- |
| Repository | https://github.com/AmeyaBorkar/Nifty-Pricing-Mirror |
| Visibility | Public |
| Category | finance |
| Primary language(s) | Python (1,425 LOC across 11 `.py` files) + a vanilla JS/HTML/CSS dashboard (593 LOC) |
| Local path | `C:\Users\ameya\Documents\Nifty Pricing Mirror` |
| Default branch | `main` (no remote `origin/HEAD` configured locally) |
| Lines of code (computed) | **Python: 1,425** (incl. `smoke_test.py` 76, excl. cache/venv). Front-end: `app.js` 191, `styles.css` 309, `index.html` 93 = 593. Support: `requirements.txt` 7, `run.ps1` 23, `.env.example` 17, `README.md` 127. |
| Source files (computed) | 11 Python files; 3 static front-end files; 1 PowerShell wrapper. (Excludes `.git`, `__pycache__`, `.cache/*.csv` demo artefacts.) |
| Key dependencies | `growwapi>=1.5.0`, `pandas>=2.1`, `requests>=2.31`, `rich>=13.7`, `python-dotenv>=1.0`, `pyotp>=2.9`, `flask>=3.0` |
| License | **None** — no `LICENSE` file present in the repo and `requirements.txt`/README declare none. |
| Last commit | `320b870` — "docs: add README covering setup, auth, CLI flags, and architecture" — 2026-05-03 23:07:23 +0530 (26 commits total, all from 2026-05-02/03) |

## What it actually is

A single-package CLI (`nifty_pricing_mirror`) that turns Groww's live LTP (last-traded-price) feed into a **basis surface**: a per-stock view of how the nearest futures contract is priced relative to its underlying cash listing. For each underlying it answers "is the future trading rich (premium / contango), cheap (discount / backwardation), or flat?" and quantifies the gap in absolute terms, percentage terms, and an annualised carry yield.

It is read-only market-data tooling — there is **no order placement, position, or portfolio code**. The only Groww calls used are `get_access_token` (auth), `get_ltp` (batched live prices), and a `get_quote` helper that is defined but never invoked anywhere in the codebase.

Three output surfaces share one in-memory `IndexSnapshot`:
1. **Rich TUI** — a `rich.live.Live` table refreshed in place (the default).
2. **Flask web dashboard** — `--serve [PORT]` starts a threaded Werkzeug server; the in-terminal table is suppressed and the browser becomes the primary surface.
3. **CSV side-channels** — `--csv-out` (atomic single-snapshot file) and `--csv-history` (append-only time series).

## Architecture & how it's structured

Data flow per refresh:

```
universe.py  ──(symbol list)──▶ instruments.py ──(InstrumentPair[])──▶ pricing.py
   (Nifty 50/200 or                 │  download + cache Groww            │
    --symbols-file)                 │  instrument master CSV;            │
                                    │  resolve spot (CASH/EQ) +          │
                                    │  nearest FUT per symbol            │
                                    ▼                                    ▼
                            groww_client.py ◀──batched_ltp(CASH/FNO)── PricingEngine.snapshot()
                            (auth + get_ltp,                            (pair LTPs, compute
                             50/call, throttled)                         basis/%/annualised,
                                                                         stance + counts)
                                                                            │ IndexSnapshot
                                          ┌─────────────────────────────────┼───────────────────────────┐
                                          ▼                                  ▼                           ▼
                                   display.py (Rich Live)        server.py (Flask /api/snapshot)   csv_export.py
                                   render_snapshot()             _serialise() → JSON → static/      write_snapshot()/
                                                                 app.js polls every 3s             append_history()
```

`cli.py` is the orchestrator: it parses args, builds `Settings`, authenticates, loads/resolves instruments once at startup, then enters one of three loops — `--once` (single snapshot), the Rich live loop (`_run_live`), or the headless serving loop (`_run_headless_loop`). Both loops swallow per-refresh exceptions so transient API errors don't kill the process; CSV writes are best-effort.

Annotated tree:

```
Nifty Pricing Mirror/
  __main__.py            # `python "Nifty Pricing Mirror"` shim → cli.main()
  run.ps1                # PowerShell wrapper: -Interval/-Once/-SymbolsFile/-VerboseOutput/-InstallDeps
  smoke_test.py          # offline E2E test: real instrument CSV + FakeClient synthetic prices
  requirements.txt       # 7 pinned runtime deps
  .env.example           # 3 credential paths + 2 optional knobs
  nifty_pricing_mirror/
    __init__.py          # __version__ = "0.1.0" (docstring still says "Nifty 50" — stale)
    cli.py               # argparse + main loop orchestration (272 lines, largest module)
    config.py            # dotenv-backed Settings, INSTRUMENTS_URL, cache path
    groww_client.py      # auth (token / key+secret / key+TOTP) + batched throttled get_ltp
    instruments.py       # download/cache instrument master; resolve spot + near future
    pricing.py           # PriceRow/IndexSnapshot dataclasses, Stance enum, basis math
    display.py           # Rich table + footer panel + LiveSurface context manager
    server.py            # threaded Flask dashboard, thread-safe snapshot slot, JSON API
    csv_export.py        # atomic snapshot write + append-only history
    universe.py          # NIFTY_50_SYMBOLS (50), NIFTY_200_SYMBOLS (200), load_symbols()
    static/
      index.html         # dashboard markup (cards, top-10 movers, sortable table)
      app.js             # polls /api/snapshot every 3s, renders, sort/filter
      styles.css         # dark theme (309 lines)
  .cache/                # gitignored; contains demo CSVs (instruments.csv, *_demo.csv)
```

## Code walkthrough (the important parts)

**`config.py` (41 lines).** `Settings` is a frozen dataclass built from env vars via `Settings.from_env()`: `GROWW_ACCESS_TOKEN`, `GROWW_API_KEY`, `GROWW_API_SECRET`, `GROWW_TOTP_SECRET`, plus `NIFTY_REFRESH_SECONDS` (default `3`) and `NIFTY_INSTRUMENTS_CACHE_HOURS` (default `12`). `_clean()` trims and nullifies empties. Module-level it calls `load_dotenv(ROOT / ".env")` and defines `INSTRUMENTS_URL = "https://growwapi-assets.groww.in/instruments/instrument.csv"` and `INSTRUMENTS_CACHE_PATH = ROOT/.cache/instruments.csv`.

**`groww_client.py` (139 lines).** `GrowwClient._authenticate` tries three paths in strict order: (1) `access_token` → `GrowwAPI(token)`; (2) `api_key + totp_secret` → `pyotp.TOTP(secret).now()` then `GrowwAPI.get_access_token(api_key, totp=...)`; (3) `api_key + api_secret` → `GrowwAPI.get_access_token(api_key, secret=...)`; otherwise raises `AuthenticationError`. (Note the order differs from how the README's table lists them as A/B/C — see README vs. code.) `growwapi` and `pyotp` are imported lazily inside the function. `batched_ltp(segment, symbols)` splits the symbol list into chunks of `_BATCH_SIZE = 50`, calls `_throttle()` before each `get_ltp(segment=..., exchange_trading_symbols=tuple(chunk))`, and merges. `_throttle()` enforces `_MIN_GAP_SECONDS = 0.12` between calls using `time.monotonic()` + `time.sleep`. `_normalise_ltp_response` defensively handles three observed SDK response shapes: flat `{key: float}`, `{key: {ltp/last_price/lastPrice/price: ...}}`, and a `{"data": {...}}` wrapper. `quote()` wraps `get_quote` but is **never called** anywhere.

**`instruments.py` (185 lines).** `InstrumentsRepo` downloads the public instrument master CSV (no auth needed) with `requests.get(..., timeout=30)` and caches it on disk; `_is_cache_fresh()` compares file mtime age in hours against `cache_hours` (default 12). On load it coerces `expiry_date` to `date` and strips string columns. `resolve_pair(symbol)` returns an `InstrumentPair(spot, future)` or `None`:
- `_resolve_spot`: filters rows where `underlying_symbol` (falling back to `trading_symbol`) `== symbol` AND `instrument_type == "EQ"` AND `exchange == "NSE"` AND `segment == "CASH"`; a second fallback matches on `trading_symbol` directly. Takes `iloc[0]`.
- `_resolve_near_future`: filters `underlying_symbol == symbol`, `instrument_type == "FUT"`, `exchange == "NSE"`, `segment == "FNO"`, non-null `expiry_date >= as_of`, sorts by `expiry_date` ascending, takes the first → the **nearest active contract**. Captures `lot_size` and `tick_size` (default 0.05) though neither is used downstream in the pricing math.
`resolve_universe()` maps the symbol list to `(resolved_pairs, skipped_symbols)`. The `exchange_trading_symbol` properties produce the `NSE_<symbol>` keys used as the dictionary join keys against `get_ltp` results.

**`pricing.py` (159 lines).** Defines `Stance` (str-Enum), `PriceRow` (frozen dataclass, one per stock), and `IndexSnapshot` (the full refresh result with totals + averages + a `total` property). `PricingEngine.snapshot()` precomputes the spot/future key lists in `__init__`, then on each call issues exactly **two `batched_ltp` calls** — one `SEGMENT_CASH`, one `SEGMENT_FNO` — pairs them by `exchange_trading_symbol`, and builds rows. See the math section below.

**`display.py` (170 lines).** `render_snapshot` returns a `rich.console.Group(table, footer_panel)`. The table has 11 columns (#, Symbol, Spot, Futures, Contract, Expiry, DTE, Basis, Basis %, Annualised %, Stance). `_signed()` colours positive green / negative red. `_stance_text()` uses **ASCII-only markers** (`^ PREMIUM`, `v DISCOUNT`, `~ FLAT`, `.. N/A`) deliberately for legacy Windows code pages. `_build_footer` shows the index bias plus per-bucket counts. `LiveSurface` is a context manager wrapping `rich.live.Live(refresh_per_second=4, screen=False)`. **Notable bug:** the table title is hardcoded `"Nifty 50 - Spot vs Futures"` regardless of the actual universe (default is Nifty 200), so a Nifty-200 run is mislabelled.

**`server.py` (137 lines).** `DashboardServer` holds a single mutable `_snapshot_payload` dict guarded by a `threading.Lock`. `update()` serialises a snapshot and swaps the slot; the snapshot loop calls it each refresh. `start()` runs `werkzeug.serving.make_server(host, port, app, threaded=True)` inside a daemon thread. Routes: `GET /` → `index.html`; `GET /static/<path>` → static assets; `GET /api/snapshot` → JSON (or `{"status":"warming_up"}` with HTTP **503** before the first snapshot); `GET /api/health` → `{"ready": bool}`. `_serialise()` builds the JSON (timestamp, totals, averages, bias, ranked rows). `_index_bias()` is the JSON-side bias classifier (duplicated, with slightly different return strings, from `display.py`'s `_index_bias`).

**`csv_export.py` (89 lines).** `write_snapshot` writes to `path + ".tmp"` then `tmp.replace(path)` — atomic on Windows and POSIX, so an Excel/Power Query reader never sees a half-written file. `append_history` appends one row per stock per refresh, writing the header only when the file is new/empty. 12-column schema (`SNAPSHOT_HEADERS`). `_num()` formats floats with `{:.6g}` and emits an empty cell for `None`.

**`cli.py` (272 lines).** Builds the argparser, configures logging through `rich.logging.RichHandler`, prints the universe size, authenticates, loads the instrument master inside a `console.status` spinner, resolves pairs (printing skipped symbols), and dispatches to `--once` / `_run_live` / `_run_headless_loop`. The headless loop prints a status line every 20th refresh (`refresh_count % 20 == 1`). Exit codes: `0` ok, `2` auth error, `3` no tradable pairs, `4` dashboard bind failure.

**`app.js` (191 lines).** Vanilla IIFE. `POLL_MS = 3000`. Fetches `/api/snapshot` (`cache: "no-store"`), handles 503 ("warming up") and non-OK. Renders summary cards, top-10 premium/discount horizontal bars (sorted by `basis_pct`, bar width scaled to max abs), and a client-side **sortable + filterable** full table. A "pulse" dot animation restarts on each successful poll via a forced reflow (`void els.pulse.offsetWidth`).

## The basis/pricing math — exactly how spot-vs-futures basis & premium/discount are computed

Per-row computation lives in `PricingEngine._build_row` (`pricing.py:111-153`). For each `InstrumentPair`:

- **Days to expiry** (computed in `snapshot()`, line 81): `dte = max((pair.future.expiry − as_of.date()).days, 0)` — never negative.
- **Guard:** if `spot is None or future is None or spot <= 0`, the row is `UNKNOWN` with all derived fields `None` (counted as `missing`).
- Otherwise:
  - `basis = future − spot`
  - `basis_pct = (future / spot − 1.0) * 100.0`
  - `annualised = basis_pct * (365.0 / max(dte, 1))` — simple proportional annualisation; `dte` is floored at 1 so expiry day doesn't divide by zero (with `dte=0` the annualised figure equals `basis_pct * 365`, which the code comment acknowledges keeps the number "interpretable" rather than rigorous).
- **Stance thresholds:** `abs(basis_pct) < 0.01` → `FLAT`; `basis_pct > 0` → `PREMIUM`; else → `DISCOUNT`. (The flat band is ±0.01% of spot.)

Index aggregation (`snapshot()`, lines 100-109):
- `avg_basis_pct` / `avg_annualised_pct` = arithmetic mean over rows that have a non-`None` value (`_mean` returns `None` for an empty list). Missing rows are simply excluded from the averages.
- Bucket counts: `premium_count`, `discount_count`, `flat_count`, `missing_count`; `total = len(rows)`.

Index bias classifier (two near-identical copies — `display.py:_index_bias` and `server.py:_index_bias`):
- both 0 → `UNKNOWN`;
- `premium > discount * 1.5` → `CONTANGO` (display string: "CONTANGO (futures rich)");
- `discount > premium * 1.5` → `BACKWARDATION` ("BACKWARDATION (futures cheap)");
- otherwise → `MIXED`.

The math is intentionally simple: it uses raw LTP (no bid/ask, no cost-of-carry / interest-rate / dividend model), and lot/tick sizes are resolved but not factored in.

## Data sources & integrations (Groww API usage)

- **Instrument master:** plain HTTPS GET of `https://growwapi-assets.groww.in/instruments/instrument.csv` (public, no auth), cached to `.cache/instruments.csv` for 12h. Parsed with pandas; the code relies on columns `underlying_symbol`, `trading_symbol`, `instrument_type` (`EQ`/`FUT`), `exchange` (`NSE`), `segment` (`CASH`/`FNO`), `expiry_date`, `lot_size`, `tick_size`.
- **Auth:** `growwapi.GrowwAPI` — either `GrowwAPI(access_token)` directly, or `GrowwAPI.get_access_token(api_key, totp=...)` / `(api_key, secret=...)`. TOTP path uses `pyotp`.
- **Live prices:** `GrowwAPI.get_ltp(segment, exchange_trading_symbols)` — the only live call actually made. Batched at 50 symbols/call, throttled to a 0.12s minimum gap. Comment cites Groww's documented limits (10 req/sec, 300 req/min). **Reality check on call volume:** the README claims "2 calls per refresh." That holds only for Nifty 50 (50 symbols = 1 chunk × 2 segments). The **default Nifty 200** universe = 4 chunks per segment × 2 segments = **8 `get_ltp` calls per refresh**, still comfortably inside the rate limit.
- **Unused:** `GrowwClient.quote()` wraps `get_quote` but has no callers.

## Tech stack & dependencies

- **Language/runtime:** Python 3.10+ (uses PEP 604 `X | None` unions and `from __future__ import annotations`; README states 3.10+).
- **Runtime deps (pinned in `requirements.txt`):** `growwapi>=1.5.0` (broker SDK), `pandas>=2.1` (instrument master parsing/filtering), `requests>=2.31` (CSV download), `rich>=13.7` (TUI), `python-dotenv>=1.0` (`.env`), `pyotp>=2.9` (TOTP auth), `flask>=3.0` (dashboard; Werkzeug's `make_server` used directly for threaded serving).
- **Front-end:** dependency-free vanilla HTML/CSS/JS (no build step, no npm).
- **No test framework** (no pytest); the only test is the hand-rolled `smoke_test.py`.

## Build / run / test (actual commands)

```powershell
# Install deps
python -m pip install -r requirements.txt

# Configure credentials: copy .env.example -> .env, fill ONE auth path
#   GROWW_ACCESS_TOKEN=...                         (token)
#   GROWW_API_KEY=... + GROWW_TOTP_SECRET=...      (TOTP, README-recommended)
#   GROWW_API_KEY=... + GROWW_API_SECRET=...       (key+secret)

# Run (default: live Rich table, Nifty 200, 3s refresh)
python -m nifty_pricing_mirror.cli

# Common flags
python -m nifty_pricing_mirror.cli --index nifty50 --interval 5
python -m nifty_pricing_mirror.cli --once
python -m nifty_pricing_mirror.cli --serve 9000 --host 0.0.0.0      # browser dashboard
python -m nifty_pricing_mirror.cli --csv-out out\latest.csv --csv-history out\history.csv
python -m nifty_pricing_mirror.cli --symbols-file my_basket.txt     # custom universe

# PowerShell wrapper (exposes a subset of flags)
.\run.ps1                 # live loop
.\run.ps1 -Once
.\run.ps1 -InstallDeps    # pip install then run

# Offline smoke test (downloads real instrument CSV; uses FakeClient synthetic LTPs — NO Groww auth)
python smoke_test.py
```

The offline `smoke_test.py` is the practical way to verify instrument resolution + rendering without credentials: it loads the real instrument master, resolves the default universe, then runs `PricingEngine` against a `FakeClient` that cycles deterministic premium/discount/flat tweaks to exercise every stance.

## Status, completeness & notable gaps

Working and complete: auth (3 paths), instrument download/cache/resolution, batched throttled LTP, basis math, Rich live TUI, Flask dashboard with sortable/filterable table and top-10 movers, both CSV exporters, an offline smoke test, and a PowerShell launcher. The project is a tight, single-author build — all 26 commits land on 2026-05-02/03 with clean conventional-commit messages.

Gaps / rough edges:
- **No `LICENSE`** — repo is public with no license file.
- **Mislabelled table title:** `display.py` hardcodes "Nifty 50 - Spot vs Futures" even when tracking Nifty 200 (or a custom file). The `__init__.py` docstring is similarly stuck on "Nifty 50."
- **No automated test suite** (no pytest), only `smoke_test.py`.
- **Duplicated bias logic** in `display.py` and `server.py` (two copies, divergent strings).
- **Dead code:** `GrowwClient.quote()` / `get_quote` is unused.
- **`run.ps1` exposes only a subset** of CLI flags (no `--index`, `--csv-out`, `--csv-history`, or `--serve`), so the dashboard and CSV side-channels are CLI-only.
- **Naive annualisation** (`365/dte`, dte floored at 1) and stance based purely on raw LTP — no carry/rate/dividend model, no bid/ask, lot/tick unused.
- **Hardcoded constituents** — index lists are baked into `universe.py` (Nifty 50 = 50, Nifty 200 = 200 symbols) and will drift as NSE rebalances; `--symbols-file` is the escape hatch.

## README vs. code

- **"Nifty 50" labelling vs. Nifty 200 default.** The README's title block and the package both default to Nifty 200, yet `display.py`'s table title and `__init__.py`'s docstring still say "Nifty 50." Code-confirmed stale labels.
- **"2 calls per refresh."** The README ("How it works", step 3) and the `groww_client.py` comment both say 2 calls/refresh "for the Nifty 50 universe." Accurate for Nifty 50 only; the **default Nifty 200** makes 8 (`ceil(200/50)=4` chunks × 2 segments). The README's own default is Nifty 200, so the headline figure undercounts the default case.
- **Credential-path ordering.** The README table labels paths A (token), B (key+secret), C (key+TOTP) and calls C "recommended." The code's actual try-order is token → key+TOTP → key+secret, i.e. TOTP is attempted before key+secret — a different precedence than the table's A/B/C lettering implies (functionally fine since only one set should be populated, but worth noting).
- **Everything else matches:** CLI flags, defaults (`--interval` 3.0, `--serve` default port 8080, `--host` 127.0.0.1), atomic CSV behaviour, 12h cache, instrument-master URL, the three output surfaces, exit-on-no-pairs behaviour, and the basis/annualised/stance/bias definitions are all faithfully described.

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\Nifty Pricing Mirror. Source of truth: https://github.com/AmeyaBorkar/Nifty-Pricing-Mirror*
