# LexDrift

> A FastAPI + Next.js system that quantifies *how* public companies change their language across SEC filings (10-K / 10-Q / 8-K). For each filing it locates the previous filing of the same form type, extracts the same named sections (Risk Factors, MD&A, Business, etc.), and measures the semantic shift section-by-section using sentence-transformer embeddings (cosine distance), token-set Jaccard distance, sentence-level alignment, Loughran-McDonald + FinBERT sentiment deltas, and a stack of "research" signals (obfuscation, entropy, kinematics, contagion, latent space). Drift that is statistically anomalous for a given company raises alerts, surfaced through a Next.js dashboard, screener, company drill-down, and alert/watchlist pages.

| Field | Value |
|---|---|
| Repository | https://github.com/AmeyaBorkar/lexdrift |
| Visibility | Public |
| Category | finance |
| Primary language(s) | Python (backend, ~14,059 LOC in `src/lexdrift`), TypeScript/React (frontend, ~2,532 LOC) |
| Local path | `C:\Users\ameya\Documents\FiananceProject\lexdrift` |
| Default branch | `main` |
| Lines of code (computed) | ~19,107 across source files (Python in `src`/`scripts`/`tests`/`alembic` + frontend `.ts`/`.tsx`/`.css`); `src/lexdrift` alone is 14,059 LOC |
| Source files (computed) | 94 source files (77 `.py` incl. tests/scripts/alembic; 16 frontend `.ts`/`.tsx`; 1 `.css`). `src/lexdrift` package = 54 `.py` files |
| Key dependencies | Backend: FastAPI, uvicorn, SQLAlchemy 2.0 (async, aiosqlite), Alembic, httpx, BeautifulSoup4/lxml, sentence-transformers, pandas/numpy/scipy, Celery[redis], pydantic-settings. Lazily-imported (NOT in `pyproject.toml`): transformers (FinBERT), keybert, scikit-learn, networkx, umap-learn, torch, groq. Frontend: Next.js 16.2.2, React 19, Tailwind v4, Recharts, lucide-react, next-themes |
| License | MIT (declared in `pyproject.toml` and README; no standalone `LICENSE` file present) |
| Last commit | `823522b` — "Untrack transient SQLite journals and local Claude settings" (2026-04-19) |

---

## What it actually is

LexDrift is a "semantic drift analyzer" for SEC filings. The thesis (grounded in the Cohen-Malloy-Nguyen "Lazy Prices" and Loughran-McDonald literature cited in the README) is that *changes* in disclosure language are leading indicators, so the product compares each filing to its predecessor rather than reading filings in isolation.

Concretely, the working tree contains:

- A **FastAPI backend** (`src/lexdrift`) with 28 routed endpoints (the `@app.get("/health")` plus 27 `@router.*` decorators across five routers) that ingest filings from SEC EDGAR, parse them into named sections, run a multi-layer NLP drift analysis, persist results to a SQLAlchemy database, and generate alerts.
- A **Next.js frontend** (`frontend/`) — App Router, 5 pages (dashboard, screener, company/[ticker], alerts, watchlist) plus layout/UI components and a fully-typed API client.
- A **Celery worker layer** (`src/lexdrift/workers`) with sync DB sessions, a Redis broker, a beat schedule (EDGAR poll every 30 min, daily pipeline at 06:00 UTC), and a daily pipeline task.
- A **training package** (`src/lexdrift/training`) for self-supervised embedding fine-tuning and torch/sklearn risk + boilerplate classifiers.
- Alembic migrations, a Dockerfile + docker-compose, and a checked-in `lexdrift.db` (~78 MB SQLite) plus downloaded filing HTML under `data/filings/<CIK>/`, indicating the system has actually been run end-to-end.

Notable reality check: the README's headline claim that embeddings use `all-MiniLM-L6-v2` is **false** for the current code — both `config.py` and `.env.example` default to **`BAAI/bge-small-en-v1.5`** (384-dim, which is why `validated_bytes_to_embedding` hard-codes `expected_dim=384`). The README also omits the FinBERT sentiment path, the Groq LLM narrative layer, and the `intelligence`/`cross-filing`/`narrative`/`reasoning` modules that exist in code.

---

## Architecture & how it's structured

Pipeline: **EDGAR → ingest (download + parse to sections) → embed → drift-detect (multi-layer) → score/alert → persist → API → frontend.**

```
SEC EDGAR (data.sec.gov, www.sec.gov, efts.sec.gov)
   │   company_tickers.json (ticker↔CIK), submissions/CIK*.json, Archives/.../*.htm
   ▼
edgar/client.py  ── rate-limited async httpx (Semaphore = SEC_RATE_LIMIT=10), 429 backoff
edgar/tickers.py ── ticker↔CIK index, cached to data/company_tickers.json
edgar/filings.py ── parse submissions JSON → filing list; build doc URL; download HTML
edgar/parser.py  ── iXBRL extraction → regex section detection (3 passes) → title-based
   ▼  POST /filings/ingest/{ticker}
   ▼  raw HTML saved to data/filings/<CIK>/<accession>.html ; Sections rows created
nlp/drift.py  ── orchestrator: tokenize → jaccard → word-change → sentiment Δ →
   │              encode_text + cosine_distance → compare_sentences → risk score →
   │              boilerplate classify → compare_keyphrases   (graceful degradation)
   │   uses: embeddings.py, sentences.py, sentiment.py, risk.py, phrases.py,
   │         boilerplate.py, tokenizer.py
   ▼  POST /filings/{id}/analyze   (api/drift.py)
   ▼  DriftScore + SentenceChange + KeyPhrase rows; obfuscation & entropy computed
nlp/anomaly.py ── per-company z-score vs that company's own history → alert levels
   ▼
Alert rows (drift_anomaly | critical_risk_language | phrase_change | obfuscation_detected)
   ▼  GET /alerts, /drift/screener, /companies/{ticker}/drift, /research/*
frontend (Next.js)  ── dashboard, screener, company drill-down, alerts, watchlist
```

Annotated source tree (build/dep dirs excluded):

```
lexdrift/
├── pyproject.toml            # hatchling build, deps, ruff/pytest config (MIT, py>=3.11)
├── alembic.ini, alembic/     # 2 migrations: initial_schema + add_sentence_changes_table
├── Dockerfile                # Python 3.11 image
├── docker-compose.yml        # app + workers + redis + postgres
├── lexdrift.db (~78MB)       # checked-in SQLite (real run data); .db-journal also present
├── data/
│   ├── filings/<CIK>/...     # downloaded raw filing HTML (AAPL=320193 etc. by CIK)
│   ├── default_phrases.json  # priority watchlist phrases
│   └── Loughran-McDonald_MasterDictionary_1993-2024.csv  (expected; loaded by sentiment.py)
├── src/lexdrift/
│   ├── main.py               # FastAPI app, lifespan (create_all + TF-IDF bootstrap), CORS *, 5 routers
│   ├── config.py             # pydantic-settings; embedding_model = BAAI/bge-small-en-v1.5
│   ├── api/  companies.py filings.py drift.py alerts.py research.py
│   ├── edgar/  client.py tickers.py filings.py parser.py
│   ├── nlp/  (18 modules) drift sentences embeddings sentiment risk phrases anomaly
│   │         obfuscation entropy velocity contagion latent_space boilerplate diff
│   │         tokenizer narrative reasoning intelligence cross_filing patterns
│   ├── workers/  celery_app pipeline ingest analyze monitor cache
│   ├── training/  finetune data_quality risk_classifier boilerplate_classifier backtest
│   └── db/  models.py engine.py session.py
├── frontend/src/
│   ├── app/  page.tsx (dashboard) screener/ company/[ticker]/ alerts/ watchlist/
│   │         layout.tsx globals.css
│   ├── components/  layout/{sidebar,header} ui/{badge,card,data-table,skeleton} theme-provider
│   └── lib/  api.ts (typed client) utils.ts
├── scripts/  run_all.py  backfill.py
└── tests/  84 test functions across test_api / test_edgar / test_nlp / integration
```

---

## Code walkthrough — backend (FastAPI) module by module

### Entry point — `main.py`
Creates the `FastAPI` app (`title="LexDrift"`, `version="0.1.0"`). The `lifespan` async context manager runs `Base.metadata.create_all` on startup (dev convenience; "use Alembic in production") and bootstraps the TF-IDF corpus from existing DB sections via `bootstrap_corpus_from_db()`. On shutdown it closes the EDGAR client and disposes the engine. CORS is wide open (`allow_origins=["*"]`). Five routers are mounted: companies, filings, drift, alerts, research; plus a bare `GET /health`.

### Config — `config.py`
A pydantic-settings `Settings` reading `.env`. Key defaults: `database_url="sqlite+aiosqlite:///./lexdrift.db"`, `sec_rate_limit=10`, `embedding_model="BAAI/bge-small-en-v1.5"`, `finbert_model="ProsusAI/finbert"`, `use_finbert_sentiment=True`, `embedding_chunk_size=2000`/`overlap=400`, Groq LLM (`llm_model="llama-3.3-70b-versatile"`, `llm_enabled=True`, empty key by default), and alert tuning (`drift_threshold=0.15`, `sentiment_stddev_multiplier=2.0`).

### EDGAR layer — `edgar/`
- **`client.py`** — `EdgarClient`, an async `httpx` wrapper. A module-level `asyncio.Semaphore(settings.sec_rate_limit)` enforces SEC's rate cap; `get()` retries 3× with exponential backoff on HTTP 429 and on request errors. Sends the configured `User-Agent`. Singleton `edgar_client`.
- **`tickers.py`** — downloads `company_tickers.json` from `www.sec.gov`, caches it to `data/company_tickers.json`, and builds in-memory `_by_ticker` / `_by_cik` indexes. Provides `lookup_ticker`, `lookup_cik`, `search_companies` (substring over ticker + name), and `pad_cik` (zero-pad to 10).
- **`filings.py`** — `get_filing_metadata` (submissions JSON), `parse_filing_list` (filters by `FORM_TYPES = {10-K, 10-Q, 8-K, 10-K/A, 10-Q/A}` and date range from the `recent` arrays), `build_document_url`, `download_filing`.
- **`parser.py`** — the heavy lifter. `parse_filing()` tries **iXBRL extraction** first (`_try_ixbrl_extraction` reads `ix:nonNumeric` / `ix:nonfraction` tags and maps `*TextBlock` names via `IXBRL_SECTION_MAP`). If that yields nothing, it falls back to `clean_html` (strips scripts/styles, normalizes whitespace, strips XBRL namespace preamble and leading Table-of-Contents blocks) then `extract_sections`. Section extraction uses **three passes** of regex over `SECTION_PATTERNS_10K/10Q/8K`: (1) `Item N.` heading patterns excluding the detected TOC zone, with cross-reference-stub detection (`_is_cross_reference` drops "See Item 2", "Not applicable", incorporated-by-reference, etc.) and resolution (`_try_resolve_cross_reference` chases "see Note 15"); (2) more aggressive ALL-CAPS / no-punctuation patterns when any 10-K section has <100 words; (3) title-based patterns ("RISK FACTORS.", "LEGAL PROCEEDINGS.") for filings (GE/HON/MCD) that drop the "Item 1A" prefix. Returns `{section_type: text}`, or `{"full_text": text}` if nothing matches.

### NLP layer — `nlp/` (18 modules)
- **`drift.py`** — `compute_drift(prev_text, curr_text, prev_embedding, curr_embedding)` is the orchestrator. Core metrics (tokenize, `jaccard_distance`, `word_change_counts`, embeddings + `cosine_distance`) propagate errors; auxiliary signals (sentiment, sentence comparison, risk scoring, boilerplate, keyphrases) each degrade gracefully with logged warnings and empty fallbacks. Returns a dict with cosine/jaccard distance, added/removed words, sentiment prev/curr/delta, serialized embeddings, `sentence_changes` (risk-scored), and `keyphrase_changes`. `validated_bytes_to_embedding` checks byte length %4 and `expected_dim=384`.
- **`embeddings.py`** — thread-safe lazy `SentenceTransformer(settings.embedding_model)`. `encode_text` splits long text into overlapping chunks (`_chunk_text`, 2000/400) and **mean-pools** chunk embeddings (so full sections are represented, not truncated). `embedding_to_bytes`/`bytes_to_embedding` use raw float32 `tobytes`. `cosine_similarity` / `cosine_distance` (= 1 − sim).
- **`sentences.py`** — sentence-level alignment. Splits both sections, batch-encodes (cap `MAX_SENTENCES=300`, keep first+last half), builds an NxN cosine similarity matrix, then greedy best-match: ≥0.85 = unchanged, 0.55–0.85 = "changed" (same topic, different meaning), unmatched = added/removed. A second pass over unmatched pairs (sim ≥0.25) flags **likely replacements** (e.g., "workforce reduction" → "organizational realignment").
- **`sentiment.py`** — Loughran-McDonald 5-category dictionary scoring (negative/positive/uncertainty/litigious/constraining) with **negation awareness** (5-token backward window, scope breakers, flip negative↔positive). When `use_finbert_sentiment` is True it lazily loads the `ProsusAI/finbert` HF `transformers` pipeline for positive/negative and fills the other three from the dictionary; falls back to the dictionary if FinBERT fails to load.
- **`risk.py`** — per-sentence risk severity (critical/high/medium/low/boilerplate) from L-M sentiment + danger-keyword proximity, or a trained MLP (`models/risk_classifier.pt`) if present. `score_changes` annotates the sentence-comparison structure and adds a `risk_summary`.
- **`phrases.py`** — keyphrase discovery via TF-IDF (corpus bootstrapped at startup) plus optional **KeyBERT** (lazily loaded, reusing the embedding model). `check_watchlist_phrases` tracks curated phrases from `data/default_phrases.json`; `compare_keyphrases` returns appeared/disappeared/intensified/diminished + semantic lists.
- **`anomaly.py`** — `detect_anomaly` computes a **company-specific z-score** (needs ≥3 history points) and optional sector z-score, mapping to normal/elevated/high/extreme; falls back to absolute thresholds (>0.3 / >0.2) when history is thin. `detect_trends` looks for drift acceleration, spikes, elevated baselines, and rising negative/uncertainty sentiment.
- **`obfuscation.py`** — fuses info-density drop, specificity (concrete referents vs hedge words), readability shift (Gunning-Fog / Coleman-Liau), and euphemism detection into an `ObfuscationScore`.
- **`entropy.py`** — Shannon entropy, cross/conditional entropy, KL divergence, novelty score, vocab overlap.
- **`velocity.py`** — semantic kinematics: velocity/acceleration/jerk/momentum of the drift time series, with phase classification (stable / drifting / accelerating / decelerating / volatile / regime_change).
- **`contagion.py`** — builds an inter-company risk-similarity graph with `networkx`, computes systemic-risk metrics (centrality, communities, spectral gap, risk hubs).
- **`latent_space.py`** — projects all section embeddings into 2D/3D via `umap-learn` (lazy) or `sklearn` PCA fallback; KMeans / KernelDensity for clustering and danger zones.
- **`boilerplate.py`** — cross-company dedup + optional trained binary classifier (`models/boilerplate_classifier.pt`).
- **`diff.py`** — sentence-level `difflib.unified_diff` and `diff_stats` (SequenceMatcher opcodes).
- **`tokenizer.py`** — **regex-based, no spaCy at runtime**: `_WORD_RE` tokenizer and an abbreviation-aware `sentence_split` (protects abbreviations, multi-dot abbreviations, decimals before splitting on `[.!?]`). spaCy is a declared dependency but is never imported.
- **`narrative.py` / `reasoning.py` / `intelligence.py` / `cross_filing.py`** — higher-level synthesis: `reasoning.py` calls the **Groq** Llama-3.3-70B API (lazy `from groq import Groq`) to produce narratives, falling back to templates; `intelligence.py` builds a structured per-company assessment using a sync SQLAlchemy session; `cross_filing.py` computes divergence and risk propagation across companies. These power the `/research` endpoints not documented in the README.

### API routers — `api/`
- **`companies.py`** (`/companies`) — `GET ""` (search), `GET /{ticker}` (EDGAR lookup + DB presence), `GET /{ticker}/filings` (EDGAR filing list annotated with DB id/status).
- **`filings.py`** (`/filings`) — `POST /ingest/{ticker}` validates ticker/form_type, ensures a `Company` row, downloads + saves HTML to disk, creates a `Filing` (`status="parsed"`, `raw_text=None` to keep DB small), parses sections into `Section` rows. `GET /{id}` (metadata + section list), `GET /{id}/sections/{type}` (full text).
- **`drift.py`** (no prefix; tags drift) — the core. `POST /filings/{filing_id}/analyze` finds the previous same-form filing, iterates matching sections, calls `compute_drift`, persists `DriftScore` + `SentenceChange` + `KeyPhrase`, runs obfuscation/entropy inline, then anomaly + trend detection over historical drift, then generates four alert types, all in a single transaction (rollback on any failure). Also `GET /drift/{id}/sentences`, `GET /companies/{ticker}/drift` (timeline), `GET /filings/{id}/diff`, `GET /drift/screener` (rank companies by latest drift), `GET /companies/{ticker}/phrases`.
- **`alerts.py`** (no prefix) — watchlists (`POST/GET /watchlists`, `POST/DELETE /watchlists/{id}/companies[...]`) and alerts (`GET /alerts` with watchlist/unread filters, `PUT /alerts/{id}/read`).
- **`research.py`** (`/research`) — `POST /obfuscation/{id}`, `POST /entropy/{id}`, `GET /kinematics/{ticker}`, `GET /overview/{ticker}`, `GET /contagion/{ticker}`, `GET /latent-space`, `GET /market-intelligence`, `GET /cross-filing/{ticker}`, `GET /intelligence/{ticker}` (last two undocumented in README). The contagion/latent-space/intelligence routes are explicitly noted in-code as expensive and "should be cached" in production.

### Data / DB — `db/`
- **`models.py`** — 11 declarative tables: `Company`, `Filing`, `Section` (with `embedding` LargeBinary BLOB), `DriftScore`, `Watchlist`, `WatchlistCompany`, `Alert` (JSON `metadata`), `KeyPhrase`, `SentenceChange`. (README says "9 tables" — undercount; there are 11 mapped classes.) Proper indexes/unique constraints throughout.
- **`engine.py`** — async engine builder that conditionally applies pool settings only for non-SQLite backends.
- **`session.py`** — FastAPI `get_db` async-session dependency.

### Workers — `workers/`
Celery app (`celery_app.py`) using Redis broker/backend, JSON serialization, task routing to ingest/analyze/monitor queues, and a beat schedule (poll EDGAR every 1800 s, daily pipeline via `crontab(hour=6)`). `pipeline.py` is the daily orchestrator using **synchronous** SQLAlchemy (`create_engine` with `+aiosqlite` stripped / `+asyncpg`→`+psycopg2`): check watched companies for new filings → ingest → analyze ingested → rebuild TF-IDF corpus → generate intelligence reports → decide retraining (`should_retrain`: ≥100 filings or >30 days) → emit an alert digest. `data/price_feed.py` exists under `data/` for price data.

### Training — `training/`
`finetune.py` (contrastive embedding fine-tuning on self-supervised 5-tier text pairs), `data_quality.py`, `risk_classifier.py` (torch MLP 384→128→4, sklearn metrics), `boilerplate_classifier.py`, `backtest.py`. These import torch/sklearn directly.

---

## Code walkthrough — frontend (Next.js) pages/components

Next.js 16.2.2 App Router, React 19, Tailwind v4, all-client-component pages. `lib/api.ts` is a typed fetch client keyed off `NEXT_PUBLIC_API_URL` (default `http://localhost:8000`) with an `ApiError` class and 20+ functions mirroring the backend.

- **`app/layout.tsx`** — root layout: Inter font, `ThemeProvider` (next-themes), persistent `Sidebar` + `Header`, `<main>` content slot. Metadata title "LexDrift - SEC Filing Language Drift Analysis".
- **`app/page.tsx`** (Dashboard) — debounced ticker search (250 ms) → company route; loads alerts + screener (`getScreener("risk_factors","cosine_distance",10)`) via `Promise.allSettled`; renders quick-stat cards (companies tracked, filings analyzed, unread alerts), a "Top Movers by Drift" table (severity color by cosine ≥0.5/0.3/0.15), and recent alerts. Has explicit empty + loading skeleton states.
- **`app/company/[ticker]/page.tsx`** — company drill-down. Parallel-loads `getCompany`, `getOverview`, `getKinematics`, `getCompanyFilings`, `getPhrases`. Renders overview cards (latest drift, alerts, drift velocity, total filings), a section-selectable **Drift Timeline** (5 section types), a Filing History table with per-row **Ingest** (`ingestFilings(ticker,"10-K",5)`) and **Analyze / Re-analyze** (`analyzeFile(id, force)`) buttons + status badges, and New vs Recurring key-phrase panels.
- **`app/company/[ticker]/drift-chart.tsx`** — Recharts line chart over the drift timeline.
- **`app/screener/page.tsx`** — sortable screener table (`getScreener`) across section types; sort fields cosine_distance / filing_date / name / ticker; skeleton rows while loading; rows route to the company page.
- **`app/alerts/page.tsx`** — alert feed with filter tabs (all / unread / critical / high), severity icon/color config, timestamp formatting, and `markAlertRead`.
- **`app/watchlist/page.tsx`** — watchlist management (create watchlist, add/remove tickers).
- **`components/`** — `layout/sidebar.tsx` (nav), `layout/header.tsx`, `theme-provider.tsx`, and `ui/` primitives (`badge` with severity variants, `card`, `data-table`, `skeleton`). `lib/utils.ts` is the `cn` clsx + tailwind-merge helper.

---

## The drift-detection method — exactly how "sounding different" is quantified in code

Drift is computed **between a filing and the most recent prior filing of the same form type**, per matched section, and is not a single number but a bundle of complementary signals:

1. **Section-level semantic drift (primary).** `encode_text` embeds each section with `BAAI/bge-small-en-v1.5` using chunked mean-pooling (2000-char windows, 400 overlap). The headline metric is `cosine_distance = 1 − cosine_similarity(prev_embedding, curr_embedding)` (`nlp/embeddings.py`, `nlp/drift.py`). Embeddings are cached as float32 BLOBs on the `Section` row and reused on later analyses.
2. **Lexical drift.** `jaccard_distance` over token *sets* and raw added/removed word counts (`nlp/drift.py`).
3. **Sentence-level alignment (`nlp/sentences.py`).** Both sections are split into sentences, embedded, and matched via an NxN cosine matrix + greedy best-match. Thresholds: ≥0.85 unchanged, 0.55–0.85 "changed" (same topic / different meaning), unmatched = added or removed. A "likely replacement" second pass links unmatched pairs with sim ≥0.25 — this is how euphemistic swaps are surfaced. These become `SentenceChange` rows.
4. **Sentiment delta.** Loughran-McDonald 5-category scores (negation-aware) ± FinBERT positive/negative; the per-category delta (curr − prev) is stored on `DriftScore.sentiment_delta`.
5. **Anomaly judgment (`nlp/anomaly.py`).** A raw cosine distance is meaningless across 8,000 companies, so drift is converted to a **company-specific z-score** against that company's own section history (≥3 points), mapped to elevated/high/extreme. This drives `drift_anomaly` alerts (not the static `drift_threshold`, which is largely vestigial in the analyze path).
6. **Research overlays.** Obfuscation score, KL-divergence/novelty (entropy), and kinematic velocity/acceleration/phase add orthogonal "is this change suspicious / accelerating?" signals.

Alerts created in `POST /filings/{id}/analyze`: `drift_anomaly` (when a section's z-score is anomalous), `critical_risk_language` (sentence risk scoring finds critical changes), `phrase_change` (priority watchlist phrase appeared/disappeared), and `obfuscation_detected` (overall obfuscation score > 0.5).

---

## Data sources & integrations (EDGAR/SEC, embedding model)

- **SEC EDGAR** is the sole filing source. Endpoints used: `https://www.sec.gov/files/company_tickers.json` (ticker↔CIK), `https://data.sec.gov/submissions/CIK{cik}.json` (filing index), `https://www.sec.gov/Archives/edgar/data/{cik}/{accession}/{doc}` (filing HTML). `efts.sec.gov/LATEST` is defined as `EFTS_URL` on the client but not exercised by the read paths. Access is rate-limited (Semaphore of 10) and identifies via a configurable `User-Agent`. There is no paid data provider; everything is the free EDGAR API. Raw HTML is persisted to `data/filings/<CIK>/<accession>.html`.
- **Embedding model:** `sentence-transformers` running **`BAAI/bge-small-en-v1.5`** (384-dim), reused by KeyBERT. (README incorrectly says `all-MiniLM-L6-v2`.)
- **Sentiment models:** FinBERT (`ProsusAI/finbert`) via HF `transformers` for pos/neg, plus the **Loughran-McDonald master dictionary** CSV (`data/Loughran-McDonald_MasterDictionary_1993-2024.csv`) the user must supply.
- **LLM:** optional **Groq** API (Llama 3.3 70B) for narrative generation in `nlp/reasoning.py`; disabled gracefully when no `GROQ_API_KEY` is set (falls back to templates).
- **Infra integrations:** Redis (Celery broker/backend), SQLite (dev) / PostgreSQL (prod via asyncpg/psycopg2).

---

## API surface — endpoints table

28 routes total (1 app-level + 27 router-level). The README claims "29 routes."

| Method | Path | Router | Purpose |
|---|---|---|---|
| GET | `/health` | main | Liveness check |
| GET | `/companies?q=` | companies | Search companies (ticker/name substring) |
| GET | `/companies/{ticker}` | companies | EDGAR lookup + DB presence |
| GET | `/companies/{ticker}/filings` | companies | EDGAR filing list annotated w/ DB status |
| GET | `/companies/{ticker}/drift` | drift | Drift score timeline (optional section filter) |
| GET | `/companies/{ticker}/phrases` | drift | Key-phrase tracking timeline |
| POST | `/filings/ingest/{ticker}` | filings | Download + parse + store filings (limit ≤20) |
| GET | `/filings/{filing_id}` | filings | Filing metadata + section list |
| GET | `/filings/{filing_id}/sections/{section_type}` | filings | Full section text |
| POST | `/filings/{filing_id}/analyze?force=` | drift | Run full drift analysis + alerts |
| GET | `/filings/{filing_id}/diff?vs=&section_type=` | drift | Unified sentence-level diff |
| GET | `/drift/screener` | drift | Rank companies by latest drift |
| GET | `/drift/{drift_score_id}/sentences` | drift | Sentence-level changes for a drift score |
| POST | `/watchlists` | alerts | Create watchlist |
| GET | `/watchlists` | alerts | List watchlists |
| POST | `/watchlists/{watchlist_id}/companies` | alerts | Add company to watchlist |
| DELETE | `/watchlists/{watchlist_id}/companies/{ticker}` | alerts | Remove company |
| GET | `/alerts` | alerts | List alerts (watchlist/unread filters) |
| PUT | `/alerts/{alert_id}/read` | alerts | Mark alert read |
| POST | `/research/obfuscation/{filing_id}` | research | Obfuscation analysis per section |
| POST | `/research/entropy/{filing_id}` | research | Entropy / KL / novelty per section |
| GET | `/research/kinematics/{ticker}` | research | Velocity/acceleration/phase |
| GET | `/research/overview/{ticker}` | research | Combined drift+anomaly+kinematics+trends |
| GET | `/research/contagion/{ticker}` | research | Risk-contagion graph metrics |
| GET | `/research/latent-space` | research | PCA/UMAP projection of all embeddings |
| GET | `/research/market-intelligence` | research | Cross-filing market report (+ LLM narrative) |
| GET | `/research/cross-filing/{ticker}` | research | Divergence + risk-propagation signals |
| GET | `/research/intelligence/{ticker}` | research | Full per-company intelligence (+ LLM narrative) |

The frontend client (`lib/api.ts`) wires up roughly 25 of these (it does not call contagion, latent-space, market-intelligence, cross-filing, or intelligence).

---

## Tech stack & dependencies (backend + frontend)

**Backend (declared in `pyproject.toml`, requires Python ≥3.11):** FastAPI ≥0.115, uvicorn[standard], httpx, SQLAlchemy[asyncio] 2.0, aiosqlite, Alembic, beautifulsoup4, lxml, **spacy** (declared but unused at runtime), **sentence-transformers** ≥3.0, pandas, numpy, scipy, celery[redis], pydantic-settings. Dev extras: pytest, pytest-asyncio, pytest-cov, ruff. Prod extra: asyncpg. Ruff line-length 100, pytest `asyncio_mode=auto`.

**Backend (imported in code but NOT in `pyproject.toml`) — all lazily imported inside functions:** `transformers` (FinBERT), `keybert`, `scikit-learn` (PCA/KMeans/KernelDensity/MLP metrics/NearestNeighbors), `networkx`, `umap-learn`, `torch` (training + risk classifier), `groq` (LLM). This is a real packaging gap: a fresh `pip install -e .` will not pull these, and several features (FinBERT sentiment when enabled, KeyBERT phrases, contagion, latent-space, training, LLM narratives) will fail to import until they are installed manually.

**Frontend (`frontend/package.json`):** Next.js **16.2.2** (README says "Next.js 14"), React 19.2, Tailwind CSS v4 (`@tailwindcss/postcss`), Recharts 3, lucide-react, next-themes, clsx + tailwind-merge, TypeScript 5, ESLint 9.

**Infra:** Redis (Celery), SQLite/PostgreSQL, Docker + docker-compose (app + workers + beat + Redis + Postgres).

---

## Build / run / test (actual commands)

From the README quick-start and verified file layout:

```bash
# Backend
pip install -e ".[dev]"
python -m spacy download en_core_web_sm        # declared but tokenizer is regex-based
# Place Loughran-McDonald CSV at data/Loughran-McDonald_MasterDictionary_1993-2024.csv
uvicorn lexdrift.main:app --reload             # API on :8000, Swagger at /docs

# Frontend
cd frontend && npm install && npm run dev       # UI on :3000  (next dev)

# Ingest + analyze
curl -X POST "http://localhost:8000/filings/ingest/TSLA?form_type=10-K&limit=3"
curl -X POST "http://localhost:8000/filings/1/analyze"

# One-shot bulk run / backfill
python scripts/run_all.py
python scripts/backfill.py

# Workers
celery -A lexdrift.workers.celery_app worker -l info
celery -A lexdrift.workers.celery_app beat -l info

# Tests (84 test functions)
pytest

# Docker
docker-compose up
```

Caveat: because keybert/transformers/sklearn/networkx/umap/torch/groq are not in `pyproject.toml`, a clean install of just `.[dev]` is insufficient to exercise FinBERT sentiment, KeyBERT, the research/training endpoints, or LLM narratives without extra `pip install`s.

---

## Status, completeness & notable gaps

**Working / substantial.** This is a genuinely large, end-to-end project, not a stub: 14k LOC of backend, a complete typed Next.js UI, real migrations, a Celery pipeline, and — tellingly — a checked-in ~78 MB `lexdrift.db` plus downloaded filing HTML for ~9 CIKs, meaning ingestion and analysis have actually run. The drift pipeline is thoughtfully engineered (graceful degradation per auxiliary signal, single-transaction analyze with rollback, embedding caching, memory caps on sentence comparison, 3-pass section parser handling iXBRL and cross-references).

**Gaps and rough edges:**
- **Undeclared runtime dependencies** (keybert, transformers, sklearn, networkx, umap-learn, torch, groq) — the single biggest reproducibility hazard.
- **`spacy` declared but never imported** at runtime; the tokenizer is a self-contained regex module. The `python -m spacy download` step is therefore unnecessary for the core path.
- **`lexdrift.db` (78 MB) and `lexdrift.db-journal` are committed** into the tree (the latest commit message is literally about untracking transient journals/settings), inflating the repo.
- **Sector-relative anomaly detection is stubbed** — `detect_anomaly` accepts `sector_history` but the analyze endpoint always passes `sector_history=None` ("sector data not yet available").
- **No auth**, CORS is `*`, and expensive research endpoints (contagion, latent-space) are explicitly un-cached ("should be cached in production").
- **README drift is significant** (see next section) — the docs lag the code on model choice, route count, table count, file/LOC counts, Next.js version, and omit entire feature areas (FinBERT, Groq LLM, intelligence/cross-filing/narrative).
- A `pipeline.py` quirk: it analyzes filings with `status == "ingested"`, but the ingest API sets new filings to `status="parsed"` — so the daily pipeline's analyze step may not pick up API-ingested filings without a status reconciliation.

---

## README vs. code

| Claim in README | Reality in code |
|---|---|
| Embeddings use `all-MiniLM-L6-v2` | **`BAAI/bge-small-en-v1.5`** in `config.py` and `.env.example` (384-dim, matching the hard-coded `expected_dim=384`) |
| "29 API routes" | **28** routes (1 `@app` + 27 `@router`); README's own endpoint listing also omits `/research/market-intelligence`, `/research/cross-filing/{ticker}`, `/research/intelligence/{ticker}` |
| "9 SQLAlchemy tables" | **11** mapped tables (`models.py`) |
| "Python files: 65 / Lines of code: 9,700+" | **77 `.py` files** in source (54 in `src/lexdrift`); `src/lexdrift` alone is **14,059 LOC**; total source ~19,107 LOC |
| "15 NLP modules" | **18** non-`__init__` modules in `nlp/` (the README's structure list itself omits narrative/reasoning/intelligence/cross_filing/patterns) |
| "Next.js 14" | **Next.js 16.2.2**, React 19, Tailwind v4 |
| "82 tests (all passing)" | **84** `def test_` functions found (pass/fail not verified here) |
| Sentiment = Loughran-McDonald only | Also a **FinBERT** path (`ProsusAI/finbert`), on by default (`use_finbert_sentiment=True`) — unmentioned in README |
| No mention of LLM | A **Groq** Llama-3.3-70B narrative layer exists (`nlp/reasoning.py`), enabled by default, with template fallback |
| Tokenizer described as "abbreviation-aware sentence splitting"; spaCy in stack | Accurate on behavior, but it is a **pure-regex** tokenizer; **spaCy is never imported** despite being a declared dependency and a download step |
| Dependencies list (pyproject) | Omits **keybert, scikit-learn, networkx, umap-learn, torch, transformers, groq** that the code imports (lazily) |
| "MIT License" | Declared in `pyproject.toml`/README, but **no standalone `LICENSE` file** in the tree |

The README's high-level architecture, 3-layer drift model, and academic framing are accurate in spirit; the discrepancies are in concrete numbers, the embedding model name, the dependency manifest, and undocumented (newer) feature areas.

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\FiananceProject\lexdrift. Source of truth: https://github.com/AmeyaBorkar/lexdrift*
