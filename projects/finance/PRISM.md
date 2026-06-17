# PRISM

> An autonomous equity-research agent. You give it a company name; it resolves the ticker, then turns control over to an LLM running a ReAct tool-use loop. The model decides — on its own, iteration by iteration — which of 13 research tools to call (live price data, news, technicals, sentiment, web search, article reading, SEC filings, ML predictions, stock comparison), observes each result, follows leads, and finally calls a `submit_report` tool that returns a structured bullish/bearish/neutral thesis. The LLM is reached through the **OpenAI Python SDK pointed at an OpenAI-compatible endpoint** (Groq by default, also Cerebras / Gemini / Anthropic-via-base-url). Four ML models (TensorFlow/Keras LSTM, XGBoost, Prophet, statsmodels SARIMAX) are *tools the agent may choose to run*, not a fixed pipeline.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/PRISM |
| **Visibility** | Public |
| **Category** | finance |
| **Primary language(s)** | Python (58 source files); plus 1 HTML/JS single-page UI and 4 YAML config files |
| **Local path** | `C:\Users\ameya\Documents\StockPredictor` |
| **Default branch** | `main` |
| **Lines of code (computed)** | 5,120 lines of Python across `src/` + `tests/`; ~641 lines HTML/CSS/JS UI; ~143 lines YAML config (total Python in `src/` only ≈ 5,090) |
| **Source files (computed)** | 58 `.py` files (of which the 3 `tests/*` files are empty `__init__.py` stubs → 55 non-trivial Python modules) + 1 `index.html` + 4 YAML |
| **Key dependencies** | `openai`, `yfinance`, `tensorflow`, `xgboost`, `prophet`, `statsmodels`, `scikit-learn`, `ta`, `vaderSentiment`, `feedparser`, `ddgs`, `trafilatura`, `httpx`, `fastapi`, `uvicorn`, `typer`, `rich`, `pydantic`/`pydantic-settings`, `jinja2`, `pandas`, `numpy` |
| **License** | MIT (asserted by README badge only — no `LICENSE` file present in the working tree) |
| **Last commit** | `a41d494` — "chore: remove cached bytecode from tracking" — 2026-03-31 |

---

## What it actually is

PRISM (the package and CLI are named `prism`; the repo is `PRISM`; the local folder is `StockPredictor`) is an **agentic stock-analysis system**. Its defining feature is that the research flow is *not scripted*. After a single deterministic step — resolving the user's input ("Apple", "AAPL") to a ticker via `yfinance` — the program constructs a system prompt casting the model as a senior equity-research analyst and enters a loop that hands the LLM a set of function-calling tools. The LLM chooses what to call and when, reads each tool's text output back as a `tool` message, and keeps going until it either calls `submit_report` or stops emitting tool calls.

Concretely it is:

- A **ReAct / tool-use loop** in `src/prism/agent/analyst.py` (the core engine, 408 lines), driven by `openai.OpenAI(...).chat.completions.create(..., tools=...)`.
- A **tool registry + executor** in `src/prism/agent/tools.py` (1,069 lines — the largest file, ~21% of all Python). It defines 13 tools and a `ToolExecutor` that backs each with a real data source.
- **Four ML forecasting models** (`src/prism/ml/models/`), exposed to the agent through a single `run_prediction_model` tool, plus a weighted ensemble.
- **Two front-ends**: a Typer CLI (`python -m prism analyze ...`) and a FastAPI + Server-Sent-Events web UI that live-streams the agent's tool calls to a single-page chat interface.

There is also a **second, older, largely-dead architecture** still in the tree: a static "data aggregator → analysis → Jinja-templated prompt" pipeline (`src/prism/data/aggregator.py`, `src/prism/data/registry.py`, `src/prism/data/sources/*`, `src/prism/agent/prompts.py` with `AnalysisContext`). None of it is wired into the live agent path — see *Status & gaps*.

---

## Architecture & how it's structured

### Control flow (the live path)

```
CLI: python -m prism analyze "Apple"          Web: GET /api/analyze?company=Apple
            │                                              │
            ▼                                              ▼
  cli/commands/analyze.py                        web/app.py  (background thread,
            │                                     pushes events to an asyncio.Queue
            ▼                                     drained as text/event-stream SSE)
  pipeline/orchestrator.py  ── resolve_ticker(query)  [yfinance]  ← only static step
            │
            ▼
  agent/analyst.py :: AgentAnalyst.run(ticker, company_name, horizon_days, on_event)
            │
            │  build system prompt + seed user message
            ▼
   ┌───────────────────────── ReAct loop (max 25 iterations) ─────────────────────────┐
   │  client.chat.completions.create(model, messages, tools, max_tokens)               │
   │        │                                                                          │
   │        ├─ message.content  → printed ("thinking") / emit("thinking")             │
   │        ├─ no tool_calls?   → done (synthesis from free text)                      │
   │        └─ for each tool_call:                                                     │
   │              name == submit_report? → parse JSON args → AISynthesis → RETURN      │
   │              else: ToolExecutor.execute(name, json.loads(args)) → text result     │
   │                    append {"role":"tool", tool_call_id, content} → loop           │
   └──────────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  orchestrator._build_report(...)   ← pulls cached price/company/news from the
            │                          ToolExecutor's in-memory session state
            ▼
  reporting/generator.py  → Markdown / HTML / JSON  → reports/<TICKER>_<date>.md
```

Key design facts derived from the code:

- The agent's working memory is the OpenAI-style `messages` list. Tool results are returned as plain text strings (not JSON), which the LLM reads in subsequent turns.
- The `ToolExecutor` **caches fetched data in memory across the session** (`self._price_data`, `self._company`, `self._news`, `self._events`, `self._price_df`). This is how, e.g., `compute_technical_indicators` can require that `get_stock_price` was called first, and how the final report is assembled (`orchestrator._build_report` reaches into `executor._price_data[ticker]` etc.).
- Loop termination has three exits: `submit_report` called; model returns no tool calls; or `MAX_ITERATIONS = 25` reached (in which case it injects a "call submit_report now" user message and tries once more, then falls back to an "incomplete" report).

### Annotated source tree (live vs. vestigial marked)

```
src/prism/
├── __main__.py            # `python -m prism` → load_dotenv() → cli.app.main()   [LIVE]
├── agent/
│   ├── analyst.py         # ReAct loop, provider client builder, retry, synthesis [LIVE — core]
│   ├── tools.py           # 13 TOOL_DEFINITIONS + ToolExecutor (real data sources) [LIVE — core]
│   └── prompts.py         # Jinja AnalysisContext prompt builder                  [DEAD — old static mode]
├── analysis/
│   ├── technical.py       # `ta`-based SMA/RSI/MACD/Bollinger/ATR/vol            [LIVE via tool]
│   ├── sentiment.py       # VADER compound scoring                               [LIVE via tool]
│   ├── pattern_matcher.py # cosine-similarity historical pattern search          [LIVE via tool]
│   └── historical.py      # event→price-impact correlation                       [LIVE via tool]
├── ml/
│   ├── base.py            # BasePredictionModel ABC
│   ├── registry.py        # builds enabled models from config                    [LIVE via tool]
│   ├── ensemble.py        # weighted-average / voting combiner                   [LIVE via tool]
│   ├── preprocessing.py   # FeatureEngineer: ~30 features, scaling, sequences    [LIVE]
│   ├── models/
│   │   ├── lstm_model.py      # Keras LSTM (trains on the fly)                    [LIVE]
│   │   ├── xgboost_model.py   # XGBRegressor                                      [LIVE]
│   │   ├── prophet_model.py   # Prophet                                          [LIVE]
│   │   └── arima_model.py     # statsmodels SARIMAX                              [LIVE]
│   └── training/__init__.py   # empty
├── data/
│   ├── sources/price_history.py  # YFinancePriceSource + resolve_ticker()   [resolve_ticker LIVE; class DEAD]
│   ├── sources/{company_profile,financial_data,news_free,news_premium}.py [DEAD — old DataSource pipeline]
│   ├── aggregator.py / registry.py / base.py                              [DEAD — never instantiated]
│   └── cache/{manager,backends}.py  # SQLite-ish cache                    [used ONLY by `prism cache` CLI cmd]
├── pipeline/
│   ├── orchestrator.py    # resolve ticker → run agent → build report          [LIVE]
│   └── context.py         # AnalysisContext helpers                            [DEAD]
├── reporting/
│   ├── generator.py       # Markdown/HTML/JSON renderer                         [LIVE]
│   ├── charts.py          # matplotlib/plotly chart helpers                     [DEAD — not imported]
│   └── renderers/__init__.py  # empty
├── cli/
│   ├── app.py             # Typer app: analyze, version, cache                  [LIVE]
│   ├── commands/analyze.py
│   └── display.py         # rich console + helpers
├── web/
│   ├── app.py             # FastAPI: GET / (HTML), GET /api/analyze (SSE)       [LIVE]
│   └── static/index.html  # vanilla-JS chat UI, dark mode, live tool stream     [LIVE]
└── core/
    ├── config.py          # pydantic AppConfig, YAML+env layering               [LIVE]
    ├── models.py          # all pydantic domain models / enums                  [LIVE]
    ├── exceptions.py, logging.py
```

---

## Code walkthrough (the important parts)

### `core/config.py` — configuration
Loads four YAML files from `config/` (`default`, `models`, `sources`, `prompts`) and overlays environment-sourced API keys via `pydantic_settings.BaseSettings` reading `.env`. Produces an `AppConfig` with sub-models: `APIKeys` (anthropic/cerebras/groq/alpha_vantage/newsapi/finnhub), `LLMConfig` (provider, model, max_tokens, base_url), `CacheConfig`, `DataConfig`, `EnsembleConfig` (default weights lstm .30 / xgboost .30 / prophet .25 / arima .15), and `ml_models` (raw dict from `models.yaml`). Note a discrepancy *within the code*: `LLMConfig`'s Python defaults are Cerebras + `qwen-3-235b-...` + `max_tokens=16000`, but `config/default.yaml` (which wins) sets provider `groq`, model `llama-3.3-70b-versatile`, `max_tokens=8000`.

### `core/models.py` — domain contracts
Pydantic v2 models for everything: `PriceData` (with `.to_dataframe()`, `current_price`, `high_52w`/`low_52w` computed from the trailing 252 rows), `CompanyInfo`, `NewsItem`, `HistoricalEvent`, `TechnicalIndicators`, `MLPrediction`, `EnsemblePrediction`, `HistoricalPattern`, `AISynthesis` (the structured report), and `AnalystReport` (final container). `Direction` (bullish/bearish/neutral) and `Confidence` (high/medium/low) are string enums.

### `agent/analyst.py` — the engine
`AgentAnalyst.__init__` builds the provider client and converts `TOOL_DEFINITIONS` (authored in Anthropic schema shape: `name`/`description`/`input_schema`) into OpenAI function-calling shape via `_convert_tools_to_openai`. `_build_client` branches on `config.llm.provider`:
- `groq` → `OpenAI(base_url="https://api.groq.com/openai/v1", api_key=GROQ_API_KEY)`
- `cerebras` → `OpenAI(base_url=llm.base_url, api_key=CEREBRAS_API_KEY)`
- `gemini` → `OpenAI(base_url="https://generativelanguage.googleapis.com/v1beta/openai/", api_key=...)` (note: reuses `groq_api_key` as the key — a latent bug for real Gemini use)
- anything else (`anthropic`, `openai`, ...) → generic `OpenAI(base_url=llm.base_url, api_key=<first non-empty key>)`

So **every provider is reached through the `openai` SDK** — there is no use of the `anthropic` SDK at runtime despite it being a declared dependency. `run()` builds the system prompt, seeds a user message asking for a thorough `{horizon_days}`-day analysis, and loops up to 25 times. `_call_with_retry` retries up to 4 times with linear backoff (8s × attempt) when the error string looks like a 429/rate-limit/queue. Helper functions coerce the model's free-form `submit_report` arguments into a typed `AISynthesis` (`_synthesis_from_dict`), tolerating string-encoded lists from weaker models via `_ensure_list` (json → `ast.literal_eval` → wrap).

### `agent/tools.py` — registry + executor
`TOOL_DEFINITIONS` is a list of 13 tool schemas. `ToolExecutor.execute()` dispatches `name` → `self._tool_<name>(**input)`, coercing string ints first (`_coerce_types`), catching exceptions into `"Error executing ...: e"` strings, and logging each call as an `AgentAction`. Each tool returns a **human-readable text block** the LLM reads back. Highlights:
- `_tool_get_stock_price` — `yf.Ticker(...).history(...)`, computes day change, 7/30/90/YTD returns, SMA-20 relation, volume vs 20-day avg, last-5-day table. Caches the `PriceData` + raw DataFrame.
- `_tool_get_company_profile` — `yf.Ticker.info`: sector/industry/market cap/employees + P/E, fwd P/E, P/B, revenue, margins, ROE, D/E, FCF, dividend yield + analyst targets/recommendation.
- `_tool_search_news` — three sources merged & deduped: **Google News RSS** via `feedparser`, **yfinance `.news`**, and **NewsAPI** (only if `NEWSAPI_KEY` set). Caches articles per ticker for later sentiment.
- `_tool_get_financial_events` — yfinance earnings (with surprise %), splits, dividends, analyst recommendations; enriches with 5-day price impact via `analysis/historical.py`.
- `_tool_compute_technical_indicators` / `_tool_analyze_news_sentiment` / `_tool_find_historical_patterns` — thin wrappers over the `analysis/` modules; all require prior tool calls to have populated session state.
- `_tool_run_prediction_model` — builds a DataFrame from cached price data (requires ≥100 rows), then either runs one named model or the full ensemble. **ML models train on the fly at call time** (no pre-saved weights are loaded in this path).
- `_tool_compare_stocks` — relative return/volatility/correlation vs a second ticker (SPY/sector ETF/peer).
- `_tool_search_web` — `from ddgs import DDGS` → `DDGS().text(query, max_results=8)`.
- `_tool_read_webpage` — **Jina Reader** (`https://r.jina.ai/<url>`, free, no key) as primary extractor, falling back to `trafilatura`; resolves `news.google.com` redirect URLs first.
- `_tool_get_sec_filings` — SEC EDGAR: looks up CIK from `company_tickers.json`, then pulls recent filings from `data.sec.gov/submissions/CIK<10-digit>.json`, filtering to 10-K/10-Q/8-K/etc.
- `_tool_submit_report` — a no-op stub; the agent loop intercepts this tool name *before* calling the executor and uses its JSON arguments as the final report.

### `analysis/` modules
`technical.py` uses the `ta` library for SMA/RSI/MACD/Bollinger/ATR + annualized 20-day volatility. `sentiment.py` is a module-level VADER `SentimentIntensityAnalyzer`; scores title+summary, classifies at ±0.05. `pattern_matcher.py` L2-normalizes the last N daily-return vector and slides it over history with a dot-product (cosine) similarity, threshold 0.3, reporting the cumulative return in the window *after* each match. `historical.py` measures each event's 5-trading-day forward price impact.

### `ml/` package
`registry.ModelRegistry` instantiates only models marked `enabled` in `models.yaml`. `ensemble.EnsemblePredictor` runs each model, then combines: `weighted_average` (default) blends predicted % change by config weight and modulates confidence by inter-model agreement; `voting` does a plain direction majority; `stacking` falls through to weighted average. See *ML models* below for per-model detail.

### `reporting/generator.py`
`ReportGenerator.render(report, format)` → Markdown (hand-built section strings), JSON (`model_dump_json`), or HTML (the markdown wrapped in a `<pre>` inside a styled page — not actually rendered as rich HTML). Because the live orchestrator passes `ml_prediction=None`, `sentiment=None`, and `historical_patterns=[]` into the report object (the agent's *narrative* about them lives inside `AISynthesis`), the Markdown's ML-table / sentiment / pattern sections are typically skipped; the report is driven by the `AISynthesis` text blocks plus the cached price/company/news/technical data.

### CLI & Web
`cli/app.py` is a Typer app with `analyze`, `version`, and `cache {stats|clear}`. `cli/commands/analyze.py` loads config, runs the orchestrator, renders, and writes `reports/<TICKER>_<YYYY-MM-DD>.<ext>` (default markdown, also echoed to terminal via `rich.markdown`). `web/app.py` is FastAPI: `GET /` serves `static/index.html`; `GET /api/analyze?company=...` runs the (synchronous) agent in a daemon thread, pushing `on_event(type, data)` callbacks onto an `asyncio.Queue` that is drained as a `text/event-stream` (SSE). Event types: `status`, `thinking`, `tool_start`, `tool_end`, `report`, `error`, `done`. The browser uses `fetch()` + manual line-parsing of the stream (not the `EventSource` API) and renders tool cards, a "thinking" block, and the final markdown report with a bullish/bearish/neutral badge.

---

## The agent: ReAct loop, LLM provider/model, tool registry — exactly how it works

**Provider / SDK (verified):** the only LLM SDK used at runtime is **`openai`** (`from openai import OpenAI`). It is always pointed at an OpenAI-compatible chat-completions endpoint. The **default provider is Groq** (`config/default.yaml`: `provider: groq`, `model: llama-3.3-70b-versatile`, `base_url: https://api.groq.com/openai/v1`). Cerebras, Gemini (via Google's OpenAI-compat endpoint), and "anthropic/openai/other via base_url" are also supported through the same SDK. `.env.example` and the README frame Groq as the recommended free default. The `anthropic>=0.40.0` dependency is declared but **never imported in code** — Anthropic is meant to be used through its OpenAI-compatible URL, not the Anthropic SDK. Despite many comments/docstrings naming "Claude," no Claude/Anthropic-native call exists.

**Loop mechanics:** one `chat.completions.create(model, messages, tools=self.tools, max_tokens=...)` call per iteration. The model's `message.content` (if any) is surfaced as "thinking"; `message.tool_calls` drives action. Each tool call's `function.arguments` is `json.loads`-ed and executed; the string result is appended as a `{"role":"tool", "tool_call_id":..., "content":...}` message. The loop ends on `submit_report`, on an assistant turn with no tool calls, or at iteration 25 (with a forced final request). The system prompt instructs the model to "always call `get_stock_price` first," to "follow interesting leads," that a typical analysis is 8–15 tool calls, and that it **must** end by calling `submit_report` (text-only completion is treated as a degraded path).

**Tool registry (13 tools):**

| Tool | Backed by | Returns to the LLM |
|---|---|---|
| `get_stock_price` | yfinance | price, 52w range, multi-horizon returns, SMA-20 relation, last-5-day table |
| `get_company_profile` | yfinance `.info` | sector/industry/mcap + valuation/profitability ratios + analyst targets |
| `search_news` | Google News RSS + yfinance + NewsAPI(opt) | deduped recent headlines w/ source, date, summary |
| `get_financial_events` | yfinance | earnings (surprise %), splits, dividends, recs + 5-day price impact |
| `compute_technical_indicators` | `ta` lib | SMA 20/50/200, RSI-14, MACD, Bollinger, vol, ATR |
| `analyze_news_sentiment` | VADER | avg compound score + pos/neu/neg breakdown |
| `find_historical_patterns` | cosine similarity on returns | similar past windows + what happened next |
| `run_prediction_model` | LSTM/XGB/Prophet/ARIMA + ensemble | per-model + ensemble direction/confidence/Δ% |
| `compare_stocks` | yfinance | return/vol/correlation/spread vs a 2nd ticker |
| `search_web` | DuckDuckGo (`ddgs`) | top 8 web results |
| `read_webpage` | Jina Reader → trafilatura | cleaned article text (≤5000 chars) |
| `get_sec_filings` | SEC EDGAR JSON | recent 10-K/10-Q/8-K etc. |
| `submit_report` | (loop-intercepted) | ends the loop; args become the structured thesis |

(The README says "12 Research Tools" — that count excludes the `submit_report` control tool; there are **13 entries** in `TOOL_DEFINITIONS`.)

---

## ML models — what each predicts, inputs/outputs, training vs inference

All four implement `BasePredictionModel` (`train(df)`, `predict(df, horizon_days) -> MLPrediction`, `save/load`). In the agent path they are **trained on demand at predict time** (`if not self.is_trained: self.train(df)`); the `models/` dir + `save/load` exist but the live `run_prediction_model` tool never calls `registry.load_all`, so each analysis trains fresh. All produce an `MLPrediction` with `direction` (thresholded at ±1% change), `confidence` (0–1), `predicted_price`, `predicted_change_pct`, `horizon_days`.

- **LSTM (`tensorflow.keras`)** — `lstm_model.py`. Stacked LSTM (default units `[64, 32]`, dropout 0.2, Adam lr 0.001), MSE loss, EarlyStopping + ReduceLROnPlateau, up to 50 epochs, lookback window 60. Features come from `FeatureEngineer.create_features` (minmax-scaled). Predicts the next close price from the last 60-step sequence; confidence = `min(|Δ%|/10, 1)`.
- **XGBoost (`xgboost.XGBRegressor`)** — `xgboost_model.py`. 200 trees, max_depth 6, lr 0.1, `reg:squarederror`. Target is **next-day close** (`close.shift(-1)`), time-series 80/20 split, standard-scaled features (config picks 8 features: close, volume, returns, sma_20/50, rsi_14, macd, volatility_20). Confidence blends `|Δ%|/10` with top feature importance. Reports MAE/RMSE on train.
- **Prophet (`prophet`)** — `prophet_model.py`. Multiplicative seasonality, yearly+weekly, changepoint prior 0.05. Fits `(ds, y)` on close, forecasts `horizon_days` ahead; predicted price = `yhat` at horizon; confidence = `1 − (yhat_upper − yhat_lower)/current` (narrower interval → higher confidence).
- **ARIMA/SARIMAX (`statsmodels`)** — `arima_model.py`. `SARIMAX(order=(5,1,0), seasonal_order=(1,1,0,5))` on the last ≤500 closes, with a `(1,1,0)` fallback if fitting fails. Forecasts `horizon_days`; confidence from the 95% forecast interval width. Reports AIC/BIC.

**Ensemble** (`ensemble.py`): weighted-average of per-model Δ% by config weight, direction thresholded at ±1%, `agreement_ratio` = fraction of models agreeing with the consensus direction, and consensus confidence = weighted confidence × `(0.5 + 0.5·agreement)`. All of it feeds the agent only as a text observation, which the model is told to weigh with its own judgment.

---

## Data sources & integrations

| Source | Used for | Key required? |
|---|---|---|
| **yfinance** | prices, fundamentals, news, earnings/splits/dividends/recs, ticker resolution, comparisons | No |
| **Google News RSS** (`feedparser`) | primary free news search | No |
| **NewsAPI** | extra news coverage | Yes (`NEWSAPI_KEY`, optional) |
| **DuckDuckGo** (`ddgs`) | general web search | No |
| **Jina Reader** (`r.jina.ai`) | clean article/page extraction (falls back to `trafilatura`) | No |
| **SEC EDGAR** (`data.sec.gov`) | recent filings | No (uses a UA header) |
| **VADER** (local) | news sentiment | No |
| LLM provider (Groq default; Cerebras/Gemini/etc.) | the agent's reasoning | Yes (one provider key) |

Declared-but-unused integration keys: `ALPHA_VANTAGE_KEY`, `FINNHUB_KEY` appear in config/`.env.example` but are not consumed by any live tool. The `data/cache` SQLite cache and per-type TTLs in config exist but are only exercised by the `prism cache` CLI command, not by the agent (which caches in memory per session).

---

## Tech stack & dependencies

- **Agent/LLM:** `openai` SDK → OpenAI-compatible endpoints (Groq / Cerebras / Gemini / Anthropic-via-URL). `anthropic` is declared but unused.
- **ML:** `tensorflow` (Keras LSTM), `xgboost`, `prophet`, `statsmodels` (SARIMAX), `scikit-learn` (scalers/metrics).
- **Data/analysis:** `yfinance`, `feedparser`, `ddgs`, `trafilatura`, `httpx`, `beautifulsoup4`, `ta`, `vaderSentiment`, `pandas`, `numpy`.
- **Web/CLI/render:** `fastapi` + `uvicorn` (SSE), vanilla JS UI, `typer` + `rich`, `jinja2`, `matplotlib`/`plotly` (declared; charts module unused).
- **Config:** `pydantic` + `pydantic-settings`, `pyyaml`, `python-dotenv`. Python ≥3.11; build via `hatchling`; console script `prism = prism.cli.app:main`.

---

## Build / run / test (actual commands)

```bash
# install (editable)
pip install -e .
# optional extras
pip install -e ".[premium]"   # newsapi-python, alpha-vantage
pip install -e ".[dev]"       # pytest, ruff, etc.

# configure: one provider key (Groq is the default/free)
echo "GROQ_API_KEY=gsk_your_key_here" > .env

# CLI
python -m prism analyze "Apple"
python -m prism analyze "TSLA" --format html --output report.html
python -m prism analyze "NVIDIA" --horizon 60
python -m prism version
python -m prism cache stats        # or: cache clear
# (also installed as the `prism` console script)

# Web UI (note: there is NO convenience runner script in the repo)
python -c "from dotenv import load_dotenv; load_dotenv(); import uvicorn; from prism.web.app import app; uvicorn.run(app, host='0.0.0.0', port=8000)"
# open http://localhost:8000
```

**Tests:** `pyproject.toml` configures pytest (`testpaths = ["tests"]`, `asyncio_mode = "auto"`) but **there are zero actual tests** — `tests/`, `tests/unit/`, `tests/integration/` contain only empty `__init__.py` files. Running `pytest` collects nothing.

---

## Status, completeness & notable gaps

- **Working end-to-end:** the agentic CLI and web paths are complete and coherent — ticker resolution → ReAct loop → 13 tools → structured report → renderer. This is genuinely an autonomous tool-picking agent, not a fixed pipeline.
- **No tests.** Despite a configured pytest setup, no test bodies exist.
- **Dead "static pipeline" subsystem.** An earlier non-agentic architecture survives in the tree but is unreachable: `data/aggregator.py` (`DataAggregator`) is never instantiated; `data/registry.py`, `data/base.py`, and `data/sources/{company_profile,financial_data,news_free,news_premium}.py` (and `YFinancePriceSource`) are unused by the live path; `agent/prompts.py` + `AnalysisContext` + `pipeline/context.py` are only referenced by each other; `reporting/charts.py` and `reporting/renderers/` and `ml/training/` are empty or unimported. Roughly a quarter of the modules are vestigial.
- **ML never persists/reuses weights in the live path.** Models retrain on every `run_prediction_model` call (each analysis = a fresh LSTM/XGB/Prophet/ARIMA fit on ~1y of data), which is slow and non-deterministic; `save/load` and `registry.load_all` are unused at runtime.
- **Reporting under-fills structured sections.** The orchestrator passes `ml_prediction=None`/`sentiment=None`/`historical_patterns=[]` to the report, so those Markdown tables are usually empty; the substance lives in the LLM's `AISynthesis` prose.
- **Latent provider bug:** the `gemini` branch reuses `groq_api_key` as the key (no dedicated `GEMINI_API_KEY` field exists in `APIKeys`).
- **No `LICENSE` file** in the working tree despite the MIT badge.
- **Config self-inconsistency:** `LLMConfig` Python defaults (Cerebras/qwen/16000) differ from the YAML that actually loads (Groq/llama-3.3-70b/8000).

---

## README vs. code

- **"12 Research Tools"** → there are **13** entries in `TOOL_DEFINITIONS` (12 research tools + the `submit_report` control tool). The 12-research-tool framing is accurate; the literal count is 13.
- **Dependency name mismatch** → `pyproject.toml` declares `duckduckgo-search>=6.0.0`, but the code imports `from ddgs import DDGS` (the renamed package). As written, web search needs the `ddgs` package installed, not (only) `duckduckgo-search`.
- **"Claude/Groq/LLM Agent" wording and many `# Claude` comments** → real runtime uses the **`openai` SDK only**; the `anthropic` SDK is a declared dependency but never imported. Anthropic, if used, is reached via its OpenAI-compatible base URL.
- **README architecture diagram & feature list** are otherwise accurate to the live agent path (ReAct loop, on-the-fly ML, Jina Reader, Google News, SEC EDGAR, multi-provider, free-by-default).
- **README claims a web "runner"** only via an inline `python -c` one-liner — there is indeed no dedicated runner module/script in the repo; the one-liner is the actual way to launch.
- **README is silent on the dead static-pipeline subsystem and the empty test suite**, both of which are real and material.
- **`config/default.yaml`** matches the README's "Groq + llama-3.3-70b-versatile" example; the Python class defaults (Cerebras/qwen) do not, but YAML wins at load time.

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\StockPredictor. Source of truth: https://github.com/AmeyaBorkar/PRISM*
