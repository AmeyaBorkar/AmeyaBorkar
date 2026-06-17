# Engram

> Engram (`engrampy` on PyPI) is a hierarchical memory layer for LLM agents and assistants. It stores raw events, optionally consolidates clusters of related events into mid-level summaries and consolidated abstractions, retrieves coarse-to-fine across that hierarchy on a hybrid dense+lexical stack, applies a principled exponential decay where surprise/use strengthen and redundancy/contradiction weaken memories, tracks temporal validity ("as of when?"), and co-surfaces contradictions rather than silently picking one. The SQLite backend with an in-process numpy vector index is the only storage implementation that ships; Postgres/pgvector is a roadmap item, not on disk.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/Engram |
| **Visibility** | Public |
| **Category** | ml-research |
| **Primary language(s)** | Python (100% of source); SQL for storage migrations |
| **Local path** | `C:\Users\ameya\Documents\Engram` |
| **Default branch** | `main` |
| **Lines of code (computed)** | `src/engram/` = **18,536** lines across **78** `.py` files; tests = **23,905** lines / 107 files; benchmarks = 6,247 lines; scripts = 6,838 lines. Storage migrations = 720 lines of SQL across 12 files. (Excludes `.git`, `dist`, caches.) |
| **Source files (computed)** | 78 Python modules under `src/engram/`; 9 `.txt` prompt templates; 12 `.sql` migrations |
| **PyPI package** | **`engrampy`**, version **0.3.0** (`pyproject.toml`). Import name stays `import engram`. Built wheel/sdist present: `dist/engrampy-0.3.0-py3-none-any.whl`, `dist/engrampy-0.3.0.tar.gz` |
| **Key dependencies** | Runtime core: `pydantic>=2.7,<3`, `numpy>=1.26,<3` (only two). Everything else is an optional extra (see Tech stack). |
| **License** | MIT (© 2026 Ameya Borkar) |
| **Last commit** | `b5d9f5f` 2026-05-25 — "feat(bench): --locomo-split to target a single LoCoMo split (L2 prep)". Working tree has ~13 untracked changes (forensic report, audit dir, extra benchmark run JSONs) — does not alter the shipped library. |

---

## What it actually is

Engram is a **mature, single-primitive Python library** centered on one class — `engram.memory.Memory` (1,818 lines) — that composes five sub-engines: decay, consolidation, retrieval, reconciliation, and storage. It is published to PyPI as `engrampy` (the names `engram` and `engram-memory` were squatted/claimed by unrelated parties; the long comment block at the top of `pyproject.toml` documents this).

The README's headline framing has shifted: it now leads with *coarse-to-fine hierarchical retrieval + temporal reasoning + contradiction co-surfacing*, and treats **consolidation as an optional capability, off by default**. The code matches this — consolidation only runs if you construct `Memory(chat=...)` and explicitly call `.consolidate()`; with `chat=None` the consolidator is `None` and Engram behaves as a decaying hierarchical vector store.

The three-layer memory model (raw events → mid summaries → consolidated abstractions) is real and lives in the `Level` enum and the consolidation/promotion pipeline, but the actual hierarchy has **six levels**, not three (the prompt's "three-layer" description is the conceptual core; the code adds preference/topic/global routing levels on top).

The codebase shows heavy iteration: pervasive "audit M-xx / H-xx" comments reference a multi-cluster security+correctness audit, and the package is `Development Status :: 4 - Beta`.

---

## Architecture & how it's structured (three-layer memory; annotated tree)

The conceptual three-layer model, as it actually appears in code:

```
  Consolidated abstractions   Level.ABSTRACTION   ← promoted from stable summaries
            ▲                                        (promote(), off by default)
            │  promotion (corroboration + weight gate)
  Mid-level summaries         Level.SUMMARY       ← one per event cluster, produced by
            ▲                                        consolidate() (clustering + LLM abstraction)
            │  consolidation (optional): cluster events → LLM generalization per cluster
  Raw event log               Event (ItemKind.EVENT)  ← observe(); never modified
```
Plus three extra routing levels layered in later: `Level.PREFERENCE`, `Level.TOPIC`, `Level.GLOBAL` (the per-tenant aggregate "user-state" singleton).

Annotated source tree (`src/engram/`):

```
src/engram/
├── __init__.py             Public API re-exports + __version__ from installed metadata
├── memory.py               THE Memory class — orchestrates every sub-engine (1,818 lines)
├── schemas.py              Pydantic v2 models: Event, MemoryItem, Procedure, Embedding,
│                             Conflict, DecayState, Cluster, RetrievalResult, + enums
│                             (Level, ItemKind, Outcome, Verdict, Resolution, ConflictStatus)
├── ids.py                  UUID minting (new_id)
├── _vec_math.py            normalize() — L2 unit-norm so cosine == dot product
├── _time.py                utcnow() — tz-aware UTC clock used everywhere
├── _preference.py          is_preference() heuristic
├── _prompt_util.py         inline()/strip_code_fence() — prompt hardening helpers
├── _security/
│   └── prompt_injection.py looks_like_injection() — Unicode confusables, RTL, base64, etc.
├── _otel.py                Optional OpenTelemetry spans/metrics (no-op without [otel])
├── decay/
│   ├── _math.py            PURE decay formula + DecayParams (100%-coverage module)
│   ├── _engine.py          DecayEngine: record() (signal apply) + tick() (sweep)
│   └── _metrics.py         DecayMetrics / KindCounters
├── consolidation/
│   ├── _clustering.py      HDBSCAN or numpy agglomerative single-link union-find
│   ├── _abstraction.py     LLM "extract the general pattern" → strict JSON schema
│   ├── _contradiction.py   LLM judge: AGREE / CONTRADICT / UNRELATED
│   ├── _engine.py          ConsolidationEngine: consolidate(), aconsolidate(), promote()
│   └── prompts/            abstract_v1.txt, judge_v1.txt
├── reconcile/
│   ├── _engine.py          Reconciler: resolve a Conflict, invalidate the loser
│   ├── _merge.py           MERGE resolution (LLM synthesizes a unified item)
│   └── prompts/merge_v1.txt
├── retrieve/
│   ├── _engine.py          HierarchicalRetriever: coarse-to-fine + drill-down
│   ├── _bm25.py            Pure-python BM25 lexical index + reciprocal_rank_fusion
│   ├── _mmr.py             Maximal Marginal Relevance diversification
│   ├── _reranker.py        Reranker protocol + FakeReranker
│   ├── _bge_reranker.py    BGEReranker cross-encoder (sentence-transformers, lazy)
│   ├── _hyde.py            HyDE query→hypothetical-answer transform
│   ├── _multi_query.py     Query paraphrase expansion + RRF fusion
│   ├── _decompose.py       Multi-hop sub-query decomposition
│   ├── _react.py           Iterative ReAct judge for retrieve_iterative
│   ├── _temporal.py        Regex + LLM "as of" temporal anchor extraction
│   └── prompts/            hyde / multi_query / decompose / react / temporal_anchor _v1.txt
├── providers/
│   ├── _protocols.py       EmbeddingProvider / ChatProvider Protocols (sync+async)
│   ├── openai.py           OpenAIEmbedder + OpenAIChat (+ any OpenAI-compatible base_url)
│   ├── anthropic.py        AnthropicChat (Claude)
│   ├── commandcode.py      CommandCodeChat (Kimi/DeepSeek via /alpha/generate, httpx)
│   ├── local.py            LocalEmbedder (sentence-transformers, bge/e5/stella)
│   ├── _fake.py            FakeEmbedder / FakeChat (deterministic, used by all tests)
│   ├── _cache.py, _disk_cache.py, _batcher.py, _retry.py, _redactor.py, _message.py
├── storage/
│   ├── _protocol.py        Storage Protocol (the backend contract)
│   ├── sqlite.py           SqliteStorage — the ONLY shipped backend (1,988 lines)
│   ├── _vector_index.py    In-process numpy (n,d) matrix cache, sharded by (kind,model)
│   ├── _serialize.py       vector packing, metadata JSON, ISO timestamps
│   ├── _inspector.py       StorageStats / stats()
│   └── migrations/         0001..0012 .sql (schema, decay, procedures, conflicts,
│                             multi-tenant, preference, multi-granularity, perf indexes)
├── integrations/
│   ├── _agent.py           EngramAgent — framework-agnostic agent loop wrapper
│   ├── langgraph.py        EngramRetrieveNode / EngramObserveNode (lazy import)
│   ├── llamaindex.py       EngramLlamaIndexMemory (BaseMemory adapter, lazy import)
│   └── _context.py, _verify.py
└── bench/                  engram-bench CLI + suite/provider/runner harness
```

---

## Code walkthrough (the important parts) — module by module

**`memory.py` — `Memory`.** The constructor wires storage + embedder + (optional) chat into a `DecayEngine`, a `ConsolidationEngine` (only if a chat provider is present), a `HierarchicalRetriever`, and a `Reconciler`. A separate `consolidate_chat` can be passed to use a stronger model for the irreversible abstraction work while the cheap retrieval/answering path runs on `chat`. Every public method has an `a*` async twin that routes the sync body through `asyncio.to_thread` (true native-async only for `aconsolidate`, which fans out per-cluster LLM calls with an `asyncio.Semaphore`). The class also carries tenant scoping (`tenant_id`), but the docstrings are explicit: tenant is persisted on writes but **read-side enforcement is not implemented** — every retrieve ignores `tenant_id` and returns rows from any tenant (deferred to the v0.4 Postgres+RLS backend).

**`decay/_math.py` — the principled decay.** Pure, I/O-free, the module the project requires 100% coverage on. See formula below.

**`decay/_engine.py` — `DecayEngine`.** `record()` reads the row's `DecayState`, computes `dt` since `last_decayed_at`, applies the formula with the new signal, writes back atomically, and routes cold transitions through `mark_cold` (which invalidates the vector + BM25 index caches). `tick()` is the periodic sweep over every hot item applying pure decay (no signals), streamed in `batch_size` chunks so a multi-million-item store ticks without loading everything into memory.

**`consolidation/_clustering.py` — redundancy/consolidation grouping.** Two backends behind `ClusterParams.method`: HDBSCAN (density-based, optional `[consolidation]` extra) and a pure-numpy agglomerative single-link union-find fallback. `method="auto"` (default) picks HDBSCAN only when installed *and* N ≥ 50, else agglomerative. The agglomerative path vectorizes edge discovery with `np.triu(sims >= cohesion_threshold, k=1)` + `np.argwhere`. Default `min_cluster_size=3`, `cohesion_threshold=0.6`.

**`consolidation/_engine.py` — `ConsolidationEngine`.** Streams unconsolidated events (those with no provenance link yet) in deterministic `(created_at, id)` order, clusters each chunk, asks the chat provider for one generalization per cluster (`extract_abstraction`), embeds it, and atomically lands a `MemoryItem` + cluster + embedding + one provenance link per supporting event. The `promote()` pass lifts stable `Level.SUMMARY` items to `Level.ABSTRACTION` when corroboration/weight/no-open-conflict gates pass (off by default).

**`retrieve/_engine.py` — `HierarchicalRetriever`.** Coarse-to-fine: pulls `k * candidate_multiplier` candidates from the generalization levels first; per-hit, if confidence ≥ `confidence_threshold` it emits the abstraction, otherwise it drills into supporting events and re-scores them. Empty upper layer falls through to events (pure-vector-store fallback). Hybrid dense+lexical (BM25 RRF), MMR diversification, recency boost, and optional cross-encoder rerank all live here.

**`storage/sqlite.py` — `SqliteStorage`.** WAL mode, `synchronous=NORMAL`, foreign keys on, per-thread connections. Backs search methods with the in-process `VectorIndex` numpy matrix.

---

## The memory model in code — three layers, decay, strengthening, consolidation/clustering, retrieval

### Three layers (the `Level` enum, `schemas.py`)
- **Raw events** — `Event` (frozen Pydantic model, `ItemKind.EVENT`). "A raw observation. Lands first; is never modified."
- **Mid summaries** — `MemoryItem(level=Level.SUMMARY)`, one per event cluster, produced by `consolidate()`.
- **Consolidated abstractions** — `MemoryItem(level=Level.ABSTRACTION)`, promoted from stable summaries by `promote()`.
- Plus `PREFERENCE`, `TOPIC`, `GLOBAL` routing levels.

### The decay function (the "principled decay") — exact formula
From `decay/_math.py`, the canonical per-step update:

```
w_{t+1} = clamp01( w_t * alpha^dt  +  beta*r  +  gamma*c  -  delta*x )
```

- `alpha` = per-second decay base derived from a **half-life**: `alpha = 0.5 ** (1 / half_life_seconds)` (so `alpha ** half_life_seconds == 0.5`), `lru_cache`d.
- `dt` = elapsed seconds since last update; `r` / `c` / `x` = integer counts of **r**einforcement / **c**orroboration / contradiction (x) signals.
- `clamp01` maps the result into `[0,1]` (NaN → 0.0, failing loudly via the prune path).

**Default `DecayParams`:** `half_life_seconds = 30 days`, `beta = 0.10` (reinforcement), `gamma = 0.05` (corroboration — half as strong), `delta = 0.20` (contradiction — one contradiction overrides two reinforcements), `threshold = 0.05`. An item is **cold** when `weight < threshold` (strict; `threshold=0` means nothing is ever cold). This is the "recency decays, surprise/use strengthen, redundancy/contradiction weaken" mechanism, expressed as a closed-form recurrence rather than ML.

### Strengthening (surprise / use)
- **Use** drives reinforcement: retrieval is reinforcement. `HierarchicalRetriever` fires `reinforce` on every surfaced item when `reinforce_on_use` is on (the default), and `retrieve_procedures` / `retrieve_preferences` reinforce on read. This closes the "consulted → stays warm" loop.
- `Memory.reinforce` / `corroborate` / `contradict` are the explicit signal surface, each mapping to a positive/negative term above.
- For procedures, outcomes route through decay: `SUCCESS`/`PARTIAL` → reinforce, `FAILURE` → contradict, `UNKNOWN` → no-op (`_OUTCOME_SIGNAL` in `memory.py`).
- Note: "surprise" is realized via the decay weighting + the consolidation `confidence` (LLM-reported) that seeds an abstraction's initial weight; there is **no separate learned surprise/novelty model** — it is the signal-count arithmetic.

### Consolidation / clustering for redundancy
Clustering (above) groups redundant/related events; each cluster collapses into one abstraction, so redundancy is compressed into a single higher-level memory whose supporting events keep decaying underneath. `aconsolidate` parallelizes the per-cluster LLM calls. Contradiction detection (`_contradiction.py`) runs an LLM judge over vector-recalled candidates and records `CONTRADICT` verdicts as first-class `Conflict` rows.

### Retrieval
`retrieve(query, k, *, prefer="auto"|"specific"|"general", ...)` — coarse-to-fine with drill-down, hybrid dense+BM25 RRF, MMR, recency boost, optional HyDE / multi-query / decompose / iterative-ReAct / temporal-anchor / surface-conflicts / cross-encoder rerank. Temporal: `as_of=<datetime>` returns historically-correct state; invalidated items are excluded by default but reappear with an `as_of` before their invalidation.

---

## Storage & embeddings backend — what it actually uses

**Storage = SQLite only.** `SqliteStorage` (`storage/sqlite.py`, 1,988 lines) is the single shipped `Storage` implementation. WAL journal mode, `PRAGMA synchronous = NORMAL`, `foreign_keys = ON`, `temp_store = MEMORY`, `busy_timeout = 5000`, `cache_size = -65536` (64 MiB), `mmap_size = 256 MiB`, per-thread connections (`check_same_thread` enforced). Schema is applied via 12 numbered SQL migrations (`0001_initial.sql` … `0012_tenant_id_length_cap.sql`).

**Vector store = in-process numpy, not an external vector DB.** `storage/_vector_index.py` caches an `(n, d)` float matrix per `(item_kind, model)` shard, lazily built and invalidated by a dirty flag on writes; top-k is a numpy matmul against unit-norm vectors (cosine == dot product). There is **no FAISS, no sqlite-vec, no pgvector, no Chroma** in the shipped storage path — the module's own docstring calls pgvector/sqlite-vec "roadmap items, not shipped." (ChromaDB appears only as a *benchmark baseline* under the `[bench]` extra and in `benchmarks/baselines/`, never as Engram's own backend.) Lexical search is a pure-python BM25 index (`retrieve/_bm25.py`).

**Embeddings — providers actually integrated (verified by reading the adapters):**
- **OpenAI** (`providers/openai.py`, `[openai]` extra): `OpenAIEmbedder` (default `text-embedding-3-small`, 1536-dim) and `OpenAIChat` (default `gpt-4o-mini`). Supports any **OpenAI-compatible endpoint** via `base_url` (Moonshot/Kimi, OpenRouter, Together, vLLM, LM Studio).
- **Anthropic / Claude** (`providers/anthropic.py`, `[anthropic]` extra): `AnthropicChat`, **default model `claude-haiku-4-5-20251001`**. Chat-only (no Anthropic embedder — Anthropic has no embeddings API). Handles Claude's top-level `system` arg and typed content blocks.
- **Command Code** (`providers/commandcode.py`, `[commandcode]` extra, httpx): `CommandCodeChat` against an undocumented `POST /alpha/generate`, default model `kimi-k2.5` (Kimi K2.x / DeepSeek V4, newline-delimited JSON streaming).
- **Local / sentence-transformers** (`providers/local.py`, `[bench]` extra): `LocalEmbedder`, **default `BAAI/bge-large-en-v1.5`** (1024-dim), GPU auto-detect, asymmetric query prompts for stella/e5, L2-normalized at encode time. Plus `BGEReranker` cross-encoder (default `BAAI/bge-reranker-v2-m3`) behind `[reranker]`.
- **Fake** (`providers/_fake.py`): `FakeEmbedder` (SHA-256 → unit-norm, default dim 128) and `FakeChat` — deterministic, network-free, used by the entire test suite.

The provider contract is two `Protocol`s (`EmbeddingProvider`, `ChatProvider`) with sync+async methods plus a `manifest_hash()` for reproducible benchmark runs.

---

## Public API — the surface a user of `engrampy` calls

Top-level exports (`engram.__init__`): `Memory`, `SqliteStorage`, `Storage`, `StorageStats`, `stats`, `DecayParams`, `HierarchicalRetriever`, `RetrieveParams`, `RetrievePrefer`, `Reranker`, `FakeReranker`, the schema models/enums (`Event`, `MemoryItem`, `Procedure`, `ProcedureMatch`, `Embedding`, `Cluster`, `Conflict`, `ConflictStatus`, `DecayState`, `RetrievalResult`, `Resolution`, `Verdict`, `Outcome`, `Source`, `ProvenanceLink`, `Level`, `ItemKind`), plus `new_id`, `normalize`, `is_preference`. `__version__` is read from installed `engrampy` metadata.

Core `Memory` surface (selected real signatures):

```python
Memory(*, storage, embedder, chat=None, consolidate_chat=None,
       decay_params=None, prune_policy="cold", consolidation_params=None,
       retrieve_params=None, reranker=None, clock=None, tenant_id=None)

# write
observe(content: str | Event) -> Event
observe_many(contents) -> list[Event]                  # one batched embed call

# retrieve (coarse-to-fine + every retrieval mode as kwargs)
retrieve(query, k=None, *, prefer=None, confidence_threshold=None, drill_k=None,
         include_cold=None, reinforce=None, reranker=None, as_of=None,
         hyde=None, multi_query_n=None, decompose=None, temporal=None,
         surface_conflicts=None, bm25_weight=None, mmr_lambda=None,
         recency_lambda=None, ...) -> list[RetrievalResult]
retrieve_iterative(query, k=10, *, max_steps=3, ...) -> list[RetrievalResult]

# decay signals + sweep
reinforce(item_id, kind=EVENT, *, count=1, now=None) -> DecayState
corroborate(item_id, kind=MEMORY_ITEM, *, count=1) -> DecayState
contradict(item_id, kind=MEMORY_ITEM, *, count=1) -> DecayState
tick(*, now=None) -> TickResult
is_cold(item_id, kind=EVENT) -> bool
metrics() -> DecayMetrics

# consolidation / promotion
consolidate(*, max_events=None) -> ConsolidationResult     # raises if chat is None
promote(*, now=None) -> PromotionResult

# procedural memory
record_procedure(situation, action, *, outcome=UNKNOWN, metadata=None) -> Procedure
retrieve_procedures(situation, k=5, *, outcomes=None, include_cold=False) -> list[ProcedureMatch]
update_outcome(procedure_id, outcome, *, now=None) -> Procedure

# contradiction / temporal
reconcile(conflict_id, *, resolution, manual_winner_id=None) -> Conflict
list_conflicts(*, status=None, memory_item_id=None, limit=100) -> list[Conflict]

# specialized layers
record_preference(content, *, source=None, weight=1.0) -> tuple[Event, MemoryItem]
retrieve_preferences(query, k=5, ...) -> list[RetrievalResult]
record_topic(content, supporting_event_ids, *, weight=1.0) -> MemoryItem
set_user_state(content, ...) / get_user_state() -> MemoryItem | None

# every method above has an a*-prefixed async twin (aobserve, aretrieve, aconsolidate, ...)
```

Procedure retrieval ranks by `similarity * weight * outcome_boost` (SUCCESS=1.0, PARTIAL=0.8, UNKNOWN=0.7, FAILURE=0.6 — failures stay surfaced as "this didn't work" lessons).

Integrations: `EngramAgent` (framework-agnostic agent loop with `chat()`/`achat()`), `EngramRetrieveNode`/`EngramObserveNode` (LangGraph), `EngramLlamaIndexMemory` (LlamaIndex `BaseMemory`-shaped), `format_context`.

CLI: `engram-bench` (entry point `engram.bench.__main__:main`).

---

## Tech stack & dependencies

- **Language:** Python ≥ 3.10 (classifiers through 3.13), typed (`py.typed`, mypy `strict`).
- **Build backend:** Hatchling (`hatchling>=1.24`); wheel packages `src/engram`.
- **Runtime core (only two):** `pydantic>=2.7,<3`, `numpy>=1.26,<3`.
- **Optional extras:** `openai`, `anthropic`, `commandcode` (httpx), `commandcode-direct` (requests+dotenv), `reranker` (sentence-transformers, BGE cross-encoder), `otel` (OpenTelemetry), `langgraph`, `llamaindex` (llama-index-core), `consolidation` (hdbscan), `bench` (chromadb, sentence-transformers, python-dotenv), `docs` (mkdocs-material + mkdocstrings), and an inlined `dev` extra (pytest, pytest-cov, hypothesis, ruff, mypy, …). Note: `postgres`/`duckdb`/`sqlite-vec` extras were **removed** because no in-tree consumer referenced them.
- **Lint/type:** ruff (line-length 100, broad rule set incl. bandit/async), mypy strict on `src/engram`.

---

## Build / install / test (actual commands; PyPI packaging)

```bash
# install from PyPI (import name stays `engram`)
pip install engrampy
pip install 'engrampy[openai]'        # + OpenAI adapters
pip install 'engrampy[anthropic]'     # + Claude
pip install 'engrampy[consolidation]' # + HDBSCAN density clustering
pip install 'engrampy[bench,reranker]'

# from source
pip install -e '.[dev]'

# tests (config in pyproject [tool.pytest.ini_options])
pytest                  # testpaths=tests, pythonpath=[src,.], -m 'not slow' by default
pytest -m slow          # opt-in performance suite
mypy                    # strict, files = src/engram
ruff check .

# benchmarks
engram-bench run <suite> ...   # e.g. LoCoMo / LongMemEval harness
```

Packaging: hatchling auto-includes the `.txt` prompts, `.sql` migrations, and `py.typed` inside the wheel. sdist includes `/src`, `/tests`, and the top-level docs. Built artifacts for 0.3.0 already sit in `dist/`.

---

## Tests

- **107 test files**, **23,905 lines** under `tests/`. Raw `def test_` count = **1,353** (1,301 unique names); the README badge/STATUS table records a specific pytest run as **"1267 passed, 1 skipped, 17 deselected (~63s)"** — the discrepancy is parametrization + the default `-m 'not slow'` deselection, not missing tests.
- Coverage is broad and adversarial: dedicated suites for the decay math/engine/properties/replay/perf, consolidation (clustering, abstraction, contradiction, promotion, prompt-injection), storage (sqlite, migrations 0002–0006, fuzz, invariants, conflicts, vector index), every retrieval mode (bm25/rrf, hyde, mmr, decompose, multi-query, iterative, temporal anchor, recency), every provider (openai, anthropic, commandcode, local-models, batcher, cache, retry, redactor), integrations (agent, langgraph, llamaindex, verify), tenancy, async surface, OTel, and a prompt-injection corpus.
- Uses **Hypothesis** property testing (`.hypothesis/` present) and `pytest-cov` (`.coverage` artifact present).
- CI: `.github/workflows/ci.yml`.

---

## Status, completeness & notable gaps

- **Strengths:** the library is unusually complete for a personal project — six retrieval enhancement modes, an async surface, OTel hooks, four real provider integrations, deterministic replay guarantees, a benchmark harness (LoCoMo, LongMemEval), and a heavy test/property suite. The decay math is isolated, pure, and held to 100% coverage.
- **Gap — single storage backend:** only SQLite ships. Postgres + pgvector + row-level-security is repeatedly cited (in `_protocol.py`, `_vector_index.py`, `schemas.py`, README roadmap) as a v0.4 item but is **not on disk**.
- **Gap — tenant isolation is write-only:** `tenant_id` is persisted but **not enforced on reads** — every retrieve/search returns rows across tenants. The schema docstring is explicit; today's isolation requires separate SQLite files per tenant.
- **Gap — vector index not validity-aware:** temporal `as_of` filtering is applied via an over-fetch + SQL pass on candidates, not in the in-memory index.
- **Gap — promotion is an N+1 walk:** `promote()` documents (audit H-62) that bulk-promote SQL is still TODO; off by default.
- **Naming reality:** the project is import-`engram` / install-`engrampy` due to PyPI name squatting on both `engram` and `engram-memory`; PEP 541 reclaim requests are pending.

---

## README vs. code

- **PyPI package & version — consistent.** README and `pyproject.toml` both say `pip install engrampy`, version 0.3.0; description strings match exactly.
- **Consolidation framing — consistent and notably honest.** README explicitly demotes consolidation to "optional, off by default," which the code confirms (no chat provider ⇒ no consolidator). The prompt's "CONSOLIDATES" headline overstates relative to the *current* README and code, where consolidation is one capability among coarse-to-fine retrieval, temporal reasoning, and contradiction co-surfacing.
- **"Three layers" — directionally true, simplified.** README's diagram shows exactly three layers (events → summaries → abstractions); the code's `Level` enum has **six** values (adds preference/topic/global). The three-layer story is the conceptual core, not the full enum.
- **Test count — minor mismatch.** README badge says "1267 passing"; raw `def test_` count is 1,353. README's own STATUS table reconciles this ("1267 passed, 1 skipped, 17 deselected") — it's a recorded run with `-m 'not slow'`, so no real contradiction.
- **Embedding/LLM providers — accurate.** README's code samples use `OpenAIEmbedder`/`OpenAIChat` and `FakeEmbedder`/`FakeChat`; the adapters exist as described, and the OpenAI adapter genuinely supports any OpenAI-compatible `base_url`. Anthropic/Claude and a Kimi/DeepSeek (Command Code) chat adapter also exist (README mentions OpenAI-compatible endpoints for the latter).
- **Storage claims — accurate.** README roadmap labels SQLite as shipped (v0.1) and Postgres as v0.4; the code matches (SQLite only, in-process numpy vector index, no external vector DB in the library path). ChromaDB is only a benchmark baseline, never Engram's store — README does not claim otherwise.
- **Decay description — accurate.** README's "recency, reinforcement, corroboration, contradiction" decay matches the `w*alpha^dt + beta*r + gamma*c - delta*x` formula in `decay/_math.py` exactly.

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\Engram. Source of truth: https://github.com/AmeyaBorkar/Engram*
