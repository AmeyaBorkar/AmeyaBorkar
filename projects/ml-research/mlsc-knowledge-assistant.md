# mlsc-knowledge-assistant

> A grounded, citation-first **retrieval-augmented QA system** over the six-document MLSC (Microsoft Learn Student Community, VIT Pune) knowledge base, built as Ameya's submission to the MLSC AI/ML Domain Lead recruitment challenge. It is a small but complete RAG stack: structure-aware chunking, hybrid dense (bge-small, ONNX) + BM25 retrieval fused with Reciprocal Rank Fusion, **three abstention gates** (a calibrated cosine threshold, a schema-level `sufficient_context` flag, and an optional post-hoc faithfulness verifier), mechanical citation validation, and an **in-repo evaluation harness** covering retrieval, LLM-judged generation quality and abstention. It ships as a Typer CLI (`mlsc`), a FastAPI service with SSE and RFC 9457 errors, and a single-file monochrome web UI. The whole corpus is 18 chunks, and most design choices follow from that.

| | |
| --- | --- |
| **Repository** | https://github.com/AmeyaBorkar/mlsc-knowledge-assistant |
| **Visibility** | Public |
| **Category** | ml-research |
| **Primary language(s)** | Python (all logic); one self-contained HTML/CSS/JS page for the demo UI |
| **Local path** | `C:\Users\ameya\Documents\MLSC\mlsc-knowledge-assistant` |
| **Default branch** | `main` |
| **Lines of code (computed)** | `src/mlsc_assistant/` = **7,486** lines of Python (6,153 non-blank) + 619-line `web/index.html`; tests = **3,234** lines (2,685 unit + 463 integration + 86 conftest). Docs ≈ 2,246 lines of Markdown (README, 5 design docs, 2 committed reports); dev set 529 lines of YAML; config 140 lines. Excludes `.venv`, caches, `data/index`, `data/cache`, `evaluation/runs`. |
| **Source files (computed)** | 54 `.py` under `src/` (43 substantive modules + 11 one-line `__init__.py`) + 1 HTML; 17 `.py` under `tests/` (13 test modules, `conftest.py`, 3 empty `__init__`). 102 tracked files in total. **242 test functions → 249 collected cases** with parametrization. |
| **Package / host platform** | Python ≥ 3.12 package `mlsc-knowledge-assistant` 0.1.0 (hatchling), console script `mlsc`. Runs locally: `mlsc serve` starts uvicorn on `127.0.0.1:8000` (UI at `/`, API at `/v1`, OpenAPI at `/docs`). No PyPI release, no Dockerfile, no deployment config. |
| **Key dependencies** | `fastembed` (BAAI/bge-small-en-v1.5, 384-d, ONNX, no torch), `numpy`, `rank-bm25`, `py-rust-stemmers` (Snowball), `google-genai` (default LLM), `fastapi` + `uvicorn` + `sse-starlette`, `pydantic` / `pydantic-settings` / `pyyaml` / `python-dotenv`, `typer` + `rich`. Declared extras `anthropic`, `openai`, `groq`, `chroma`, `sbert`, `ragas` have **no adapter code behind them** (see README vs. code). `tenacity` is declared but never imported (retry is hand-written). Dev: `pytest`, `pytest-cov`, `ruff`, `mypy --strict`. |
| **License** | MIT © 2026 Ameya Borkar |
| **Last commit** | `64abf5c` — "Rework the UI: drop the status readout, raise contrast to AA, group the results" (2026-08-24 17:13 IST). First commit `c3edd24` "Plan the MLSC Knowledge Assistant: architecture, API contract and evaluation design" (2026-08-24 14:37 IST). **11 commits**, all in one afternoon (~2 h 36 min from first to last; the brief recommends 6 h). Working tree clean; CI green on the latest commit. |

---

## What it actually is (be concrete)

The challenge brief (`MLSC_AIML_RAG_Recruitment_Challenge.pdf`) asks for an assistant that answers natural-language questions from six supplied `.txt` files, cites source documents, handles multi-document and unanswerable questions, and includes an evaluation component reporting at least context precision, context recall, answer relevancy and faithfulness. It also forbids hard-coding answers.

What the code implements:

- **Corpus:** `data/knowledge_base/*.txt`, six files totalling 109 lines / 767 words (`about_mlsc`, `domains`, `leadership`, `membership`, `hackathons`, `code_of_conduct`). The chunker turns them into **18 chunks** (15 merged paragraphs, 2 list blocks, 1 paragraph), confirmed from the locally built `data/index/manifest.json` and `chunks.json`.
- **Index:** an `18 × 384` float32 NumPy matrix persisted as `vectors.npz`, plus a JSON chunk table and a manifest that holds SHA-256 checksums per document so staleness is detectable. No vector database.
- **Answering:** one Gemini call constrained to a JSON schema `{sufficient_context, answer, cited_chunk_ids, confidence}`. Refusal is a parsed boolean, not a string match, and cited ids are checked against what was actually retrieved.
- **Evaluation:** a 40-question hand-labelled dev set (14 direct, 6 multi-document, 5 reasoning, 3 ambiguous, 12 unanswerable split into 9 "near-miss" and 3 "off-domain") with gold chunk ids and reference answers. There are two runners: a key-free retrieval and threshold-sweep runner that CI gates on, and a quota-spending full runner that answers every question and LLM-judges the answers.
- **Key-free half:** indexing, `mlsc search`, `/v1/search`, `/v1/documents`, `/v1/health` and every retrieval metric run with no API key. Only answer generation and the judged metrics need `GOOGLE_API_KEY`.

## Architecture & how it's structured

Ports-and-adapters layout. `core/ports.py` declares `Protocol` interfaces (`Embedder`, `Chunker`, `VectorStore`, `Reranker`, `LLMProvider`, `StreamingLLMProvider`, `EmbeddingCache`), and `factories.py` is the only place config gets mapped to concrete classes.

```
mlsc-knowledge-assistant/
├─ pyproject.toml            hatchling; script `mlsc = mlsc_assistant.cli:app`; ruff + mypy(strict) + pytest config
├─ config.yaml               every tunable, with the measured calibration curve inline as comments
├─ .env.example              MLSC_LLM__PROVIDER, GOOGLE_API_KEY (+ unused ANTHROPIC/OPENAI/GROQ slots)
├─ .github/workflows/ci.yml  2 jobs: lint/types/unit tests; integration + retrieval quality gate
├─ data/knowledge_base/      the 6 supplied .txt files (verbatim) + README
├─ docs/                     ARCHITECTURE · API (HTTP contract) · DECISIONS (D1–D14) · EVALUATION · PLAN
├─ evaluation/
│  ├─ datasets/dev_set.yaml  40 questions, gold chunks/docs, reference answers, labelling rule
│  ├─ reports/               committed artefacts: retrieval-{dense,lexical,hybrid}-metrics.json,
│  │                         retrieval-hybrid.md (incl. gate-1 sweep), full-evaluation{.md,-metrics.json},
│  │                         abstention-phase4.log
│  └─ runs/                  gitignored per-run dirs (config.json, metrics.json, per_question.jsonl, report.md)
├─ src/mlsc_assistant/
│  ├─ config.py              pydantic-settings: defaults < config.yaml < MLSC_* env/.env; loads .env into os.environ
│  ├─ factories.py           composition root: make_chunker/embedder/store/retriever/provider/answerer
│  ├─ cli.py                 Typer app: index, search, info, serve (+ ask, eval registered from subpackages)
│  ├─ core/                  models.py (frozen dataclasses: Document, Chunk, ScoredChunk, RetrievalResult,
│  │                         Answer, Citation, IndexManifest, AbstentionReason) · ports.py · errors.py (typed,
│  │                         each carries HTTP status + problem+json slug)
│  ├─ ingestion/             loader.py (BOM-safe, sorted) · chunker.py (StructuralChunker) · pipeline.py (build_index)
│  ├─ embeddings/            fastembed_embedder.py (bge query prefix, L2-normalise) · cache.py (content-hash .npz cache)
│  ├─ stores/                numpy_store.py (matrix @ query, argpartition top-k, JSON/npz persistence)
│  ├─ retrieval/             dense.py · lexical.py (BM25 + stemming + low-IDF filter) · fusion.py (RRF)
│  │                         · diversify.py (per-doc cap with backfill) · retriever.py (HybridRetriever)
│  ├─ generation/            prompts.py (grounded-v1 system prompt + schema) · answerer.py (gates + citation
│  │                         binding) · verifier.py (gate 3) · cli_commands.py (`mlsc ask`)
│  │  └─ providers/          base.py (retry, error translation, JSON repair, RateLimiter) · gemini.py · registry.py
│  ├─ evaluation/            dataset.py · runner.py (retrieval + sweep) · full_runner.py · judge.py (cached judge)
│  │  │                      · report.py / full_report.py · cli_commands.py (`mlsc eval`)
│  │  └─ metrics/            retrieval.py · abstention.py · generation.py
│  ├─ api/                   app.py (factory, lifespan index load, trace-id middleware, problem+json handlers)
│  │  │                      · deps.py (AppState, lazy provider) · schemas.py (DTOs)
│  │  └─ routes/             ask.py (/ask, /ask/stream SSE) · search.py · documents.py · evaluation.py · health.py
│  └─ web/index.html         619-line monochrome SPA; uses /v1/health and /v1/ask/stream; zero external requests
└─ tests/
   ├─ unit/                  10 modules (api, chunker, config/factories, dataset, metrics ×2, generation,
   │                         loader, retrieval, store/pipeline)
   └─ integration/           real embedder, retrieval quality on the real corpus, live Gemini (key-gated)
```

Lines per subpackage: `evaluation` 2,242 · `generation` 1,403 · `api` 911 · top-level (`cli`, `config`, `factories`, `__init__`) 836 · `core` 639 · `retrieval` 531 · `ingestion` 461 · `embeddings` 260 · `stores` 203.

## Code walkthrough (the important parts)

**Chunking (`ingestion/chunker.py`).** `StructuralChunker` splits on lines while tracking character offsets against the original string, so citations can point to exact `char_range`s. It keeps a colon-terminated introductory sentence and the list after it together as one atomic `LIST_BLOCK`, so the domain list stays whole and "What technical domains exist?" has a single answering chunk. Short paragraphs are merged up to a 40-token floor (token estimate is `words × 1.3`). The merge works both ways: blocks accumulate forward, and a leftover is folded backward when a list interrupts. Blocks over 320 tokens split at sentence boundaries, which never happens on this corpus. The title line is dropped as a chunk and instead prepended to each chunk's `embed_text` (`"{title} - {chunk}"`). The displayed `text` stays clean. Chunk ids are deterministic (`leadership::c01`) so the dev set can reference them.

**Retrieval (`retrieval/`).**
- *Dense:* `embed_query` adds bge's asymmetric instruction prefix and vectors are L2-normalised, so `NumpyVectorStore.search` is a single `matrix @ query` with `argpartition` top-k.
- *Lexical:* `BM25Okapi` over the same `embed_text`. The tokenizer uses `[a-z0-9]+` so `Web3` stays one token, removes a small stopword list and applies Snowball stemming via `py-rust-stemmers`, with an `lru_cache` on stems. The notable addition is `discriminative_terms`: query terms that appear in more than 50% of chunks are dropped, and if no terms are left, BM25 returns nothing so hybrid falls back to dense. The docstring gives the reason: "What is MLSC?" reduced to `mlsc`, which is present in all 18 chunks, and BM25 noise was pushing the correct chunk out of the results.
- *Fusion:* plain RRF, `Σ 1/(k + rank)` with `k = 60`, over 15 candidates per arm. Ties break on dense score, then chunk id, which keeps the ordering deterministic.
- *Diversification:* `cap_per_document` allows at most 3 chunks per document. Skipped items backfill if the cap would return fewer than `top_k`, and relative order is preserved.
- `HybridRetriever.retrieve` returns `ScoredChunk`s that carry each arm's rank and score, the RRF score and the matched BM25 stems. That data drives the `--explain` output and the UI diagnostics. A `reranker` argument is accepted and stored but **never invoked**.

**Abstention and calibration (`generation/answerer.py`, `evaluation/metrics/abstention.py`).**
- *Gate 1, free and before any LLM call:* refuse when the best **raw cosine** (`RetrievalResult.top_dense_score`) is below `abstention.min_dense_score = 0.55`. The code compares against cosine on purpose: under hybrid, `score` is an RRF value around 0.03 and has no meaningful scale. Lexical-only retrieval has no cosine, so gate 1 defers to gate 2. An optional rank-1/rank-2 margin check exists but is disabled (`0.0`).
- *Calibration:* `runner.run_retrieval_evaluation` records each question's top cosine. `sweep_threshold` then recomputes abstention precision, recall, F1, hallucination rate and over-refusal over 29 thresholds from 0.300 to 1.000, plus separate recall for near-miss and off-domain questions. `best_operating_point` picks the highest-F1 threshold with **zero over-refusal**, which lands at 0.55 (it ties 0.45 and 0.50 on F1, and the higher threshold wins the tie). The sweep shows near-miss recall stays at 0 until 0.75, where 39% of answerable questions are already being refused. That result is the argument for gate 2.
- *Gate 2:* if the schema field `sufficient_context` is false, the model's own explanation is returned as the refusal. The prompt requires it to start with "The knowledge base…" and to say what the KB does cover. A `StructuredOutputError` (malformed JSON) also leads to abstaining, as `PROVIDER_UNAVAILABLE`.
- *Gate 3, optional (`--verify` / `verify_faithfulness`):* `verifier.verify_answer` asks the same provider to list unsupported claims, judged against the **full text of the cited chunks**. When the claim list is non-empty it overrides the boolean. If the verifier's own output fails to parse, it **fails open** (`supported=True`, reason "inconclusive").
- Every `Answer` carries a `diagnostics` dict with per-gate outcomes (`retrieval_gate`, `context_sufficiency`, `citation_binding`, `faithfulness_check`), retrieval scores, top cosine vs. threshold, tokens and timings.

**Citation binding.** `_bind_citations` keeps only cited ids that are in the retrieved context. It drops and reports unknown ids, marking binding as `repaired`, and de-duplicates. If no valid citation is left, the answer is refused with `UNFAITHFUL_ANSWER` instead of being shown without a source. A `Citation` holds a 240-character display snippet plus `char_range`. Anything that grades an answer (gate 3 and the faithfulness metric) resolves the **full chunk text** through `cited_passages()`. The code comments record that an earlier version passed the truncated snippet and pushed faithfulness down to 0.556.

**Provider layer (`generation/providers/`).** `GeminiProvider` is the only adapter. It uses native `response_schema` JSON output, sets `thinking_budget=0` (the comments report 1.18 s vs 39 s with the same verdict), a client-side `RateLimiter` (minimum interval, 15 rpm by default) and `with_retry`. The retry follows the server's advertised `retryDelay`/"retry in Ns", capped at 120 s, and otherwise uses exponential backoff with jitter. `translate_error` maps SDK errors to typed 429/503/504 domain errors. `parse_structured` cleans up fenced or prose-wrapped JSON and fails if required fields are missing. `registry.py` lists `anthropic`, `openai`, `groq` and `ollama` builders that lazy-import modules that do not exist, so selecting any of them raises a `ConfigurationError`.

**Evaluation harness (`evaluation/`).**
- *Retrieval metrics* (`metrics/retrieval.py`, no LLM): precision@k (divided by the number returned, not k), recall@k, R-precision, average precision, reciprocal rank, binary nDCG@k, document recall, all-docs-hit, and multi-document coverage over questions with more than one gold document (7 questions). All are macro-averaged and reported at k ∈ {3, 5, 6, 10}. The answer for `top_k` is retrieved once at the widest k and sliced for the rest.
- *Abstention metrics:* refusal is the positive class. Hallucination rate is the share of unanswerable questions that got answered; over-refusal is the share of answerable questions that got refused. Both are always reported together.
- *Generation metrics* (`metrics/generation.py`):
  - **Faithfulness:** the judge breaks the answer into atomic claims, checks each against the cited passages, and the score is supported/total. Zero extracted claims scores 0, not 1.
  - **Answer relevancy:** RAGAS-style. The judge writes 3 questions the answer would answer; these are embedded with bge and compared to the real question by mean cosine. A noncommittal answer scores 0.
  - **Answer correctness:** a judge verdict of correct, partially correct or incorrect, mapped to 1, 0.5 and 0.
  - Questions the system refused are **excluded** from all three, since an abstention is trivially faithful.
- *Judge* (`judge.py`): a thin wrapper around an `LLMProvider` plus a content-addressed `VerdictCache` (`evaluation/runs/judge-cache.json`) keyed on sha256 of task, model, prompt version and payload. In `mlsc eval --metrics all`, the judge is **built from the answering provider**, so it shares that provider's rate limiter.
- *Runners:* `full_runner` stops at the first `MLSCError` (for example quota exhaustion), marks the run incomplete and exits with code 2. Partial results are never scored. Each run writes `config.json`, `metrics.json`, `per_question.jsonl` and `report.md`. `dataset.validate_against_index` fails loudly when a gold chunk or document id is missing from the current index.

**API (`api/`).** `create_app()` loads the index, embedder and BM25 once in the lifespan, and the server refuses to start without an index. The provider is built lazily and cached, so `/v1/search` and the other key-free routes keep working without a key. Every response carries an `X-Trace-Id`. Errors are returned as `application/problem+json`. Abstention is a **200** with `answered: false`. `/v1/ask/stream` emits SSE events `retrieval → answer → citations → done`. The `answer` event arrives whole, because the structured call cannot be interpreted until it completes, and the endpoint runs retrieval twice (once for the early event and once inside `answer()`). `/v1/eval/runs` is a 202 background job held in an in-memory dict that runs **retrieval-only** evaluation. The web UI is mounted last at `/` through `StaticFiles`.

## Core features & public API/commands

| Capability | CLI | HTTP | Needs key |
| --- | --- | --- | --- |
| Build / check index (checksum staleness) | `mlsc index [--force --no-cache --show-chunks]` | — (`/v1/health` reports `index.stale`) | no |
| Retrieval only, with per-arm explain | `mlsc search "<q>" [-k -s --full --no-diversify]` | `POST /v1/search` | no |
| Grounded answer + citations + gates | `mlsc ask "<q>" [-k -s --verify --show-context -d --json]` | `POST /v1/ask` | yes |
| Streamed answer (sources first) | — | `POST /v1/ask/stream` (SSE) | yes |
| Browse KB / chunks with offsets | — | `GET /v1/documents`, `/{doc_id}`, `/{doc_id}/chunks` | no |
| Retrieval eval + gate-1 sweep + CI gate | `mlsc eval [--compare --fail-under metric=x -q]` | `POST/GET /v1/eval/runs[/{id}]` | no |
| Full eval (answer + judge) | `mlsc eval --metrics all [--limit N]` | — | yes |
| Show resolved config / index status | `mlsc info` | `GET /v1/health` | no |
| Serve UI + API + OpenAPI docs | `mlsc serve [--host --port --reload]` | `/`, `/v1`, `/docs` | no |

Abstention reasons (`AbstentionReason`): `no_relevant_context` (gate 1), `insufficient_context` (gate 2), `unfaithful_answer` (no valid citation, or gate 3), `provider_unavailable` (malformed structured output).

## Tech stack & tooling

- **Language/runtime:** Python ≥ 3.12, using PEP 695 generics (`def cap_per_document[T]`), `StrEnum` and frozen slotted dataclasses. The local `__pycache__` shows development ran on CPython 3.13; CI runs 3.12.
- **Retrieval:** fastembed ONNX `BAAI/bge-small-en-v1.5` (384-d; model cached in `data/cache/models`), `rank-bm25`, Snowball stemming, NumPy.
- **LLM:** Google Gemini via `google-genai`, default model `gemini-3.1-flash-lite` (pinned in `llm.models`), temperature 0, thinking disabled, 800 max output tokens.
- **Serving:** FastAPI, uvicorn, sse-starlette, Starlette `StaticFiles`.
- **Config:** pydantic-settings with a custom YAML source, layered as defaults, then `config.yaml`, then `MLSC_*` env vars (with `__` for nesting).
- **Quality:** ruff (a broad rule set: E, F, I, N, UP, B, SIM, C4, RET, ARG, PTH, RUF), `mypy --strict` with the pydantic plugin, pytest with `--strict-markers` and `integration`/`slow` markers, pytest-cov.
- **CI** (`.github/workflows/ci.yml`, Ubuntu, Python 3.12) runs two jobs:
  - *quality:* install → cache model → `mlsc index` → ruff check + format check → mypy → `pytest -m "not integration" --cov`.
  - *retrieval-gate:* `mlsc index` → `pytest -m integration` (the live-Gemini tests skip because CI has no key) → `mlsc eval --quiet --fail-under recall_at_k=0.90 --fail-under mrr=0.85 --fail-under ndcg_at_k=0.85`.
  - `gh run list` shows the latest five runs green except the Phase 7 commit. That failure was fixed in the next commit, "Fix CI: build the index before tests that need one".

## Build / dev / test (actual commands)

```bash
python -m venv .venv && source .venv/Scripts/activate   # Windows Git Bash
pip install -e ".[dev]"

mlsc index                          # build data/index (downloads ~130 MB ONNX model once); no key
mlsc search "judging criteria"      # retrieval with per-retriever explain; no key
mlsc eval                           # retrieval metrics + gate-1 sweep, writes evaluation/runs/<id>/; no key
mlsc eval --compare                 # dense vs lexical vs hybrid table
cp .env.example .env                # set GOOGLE_API_KEY
mlsc ask "What technical domains exist in MLSC?" --diagnostics
mlsc eval --metrics all             # answer + judge all 40 questions (~120 calls; verdicts cached)
mlsc serve                          # http://127.0.0.1:8000  (UI), /v1, /docs

ruff check . && ruff format --check .
mypy
pytest -m "not integration"         # unit tests (API tests skip unless an index is built)
pytest -m integration               # real embedder + real corpus; live Gemini tests need a key
```

## Tests

**242 test functions / 249 collected cases**, 3,234 lines in total.

- **Unit, 214 functions (221 cases):** `test_generation` 35, `test_api` 26, `test_eval_metrics` 26, `test_retrieval` 23, `test_generation_metrics` 21, `test_eval_dataset` 20, `test_chunker` 18, `test_config_and_factories` 17, `test_store_and_pipeline` 17, `test_loader` 11. They exercise:
  - BM25 tokenisation (`Web3` survives, inflections collapse), RRF properties and the diversification cap and backfill.
  - Each gate and citation binding, using stub providers.
  - Retry-delay parsing, error mapping and JSON salvage.
  - Metric arithmetic and judge caching.
  - The API contract. `test_api.py` is skipped unless a built index exists, and CI builds one first.
  - A UI test that asserts `index.html` contains no `http://`, `https://`, `//cdn` or `@import`.
  - `test_no_reference_answer_leaks_into_the_source_tree`, which strips docstrings with `ast` and checks that no reference answer appears verbatim in executable source. It checks only the first sentence of each answer, and only when that sentence is over 40 characters. A companion test proves the guard can fail.
- **Integration, 28:**
  - `test_retrieval_quality` (14): the brief's example ranks first, rare exact terms are found, lexical rescues a dense near-miss, near-miss unanswerables score as high as real hits, retrieval is deterministic.
  - `test_real_embedder` (7): normalisation, the query prefix, the high similarity floor, and that the cache makes a rebuild free.
  - `test_generation_live` (7): live Gemini. It refuses the Technical Head near-miss, refuses a world fact the corpus lacks, does not borrow an adjacent number, and cites multiple sources on a multi-document question. These skip without a key.

Nothing was run for this doc (read-only rule). The CI results on GitHub stand in as evidence.

### Evaluation results (verified against artefacts)

The dev set has 40 questions: 28 answerable and scored on retrieval, 12 unanswerable. Retrieval uses hybrid at k = 6, the 18-chunk index and `bge-small-en-v1.5`.

| Strategy (k=6) | Recall | Precision | R-Prec | MRR | nDCG | Doc recall | Multi-doc cov. |
| --- | --- | --- | --- | --- | --- | --- | --- |
| dense | 0.973 | 0.232 | 0.741 | 0.858 | 0.866 | 1.000 | 1.000 |
| lexical | 0.795 | 0.250 | 0.616 | 0.711 | 0.700 | 0.893 | 0.714 |
| **hybrid (shipped)** | 0.955 | 0.226 | 0.741 | **0.912** | **0.891** | 1.000 | 1.000 |

Source: `evaluation/reports/retrieval-{dense,lexical,hybrid}-metrics.json`. Every value matches `docs/EVALUATION.md`. Hybrid gains on ranking (MRR +0.054) and gives up 0.018 recall to dense, a trade-off the doc states openly.

| Abstention | Gate 1 only (threshold 0.55) | Gates 1 + 2 (full run) |
| --- | --- | --- |
| Hallucination rate | 0.83 | **0.000** |
| Abstention recall / precision / F1 | 0.17 / 1.00 / 0.29 | **1.000 / 0.923 / 0.960** |
| Over-refusal | 0.00 | 0.036 (1/28: q25) |
| Near-miss / off-domain caught | 0/9, 2/3 | **9/9, 3/3** |

| Generation (27 answered questions judged; `gemini-3.1-flash-lite` answering **and** judging) | Score |
| --- | --- |
| Faithfulness (claim-level, against cited chunks) | **0.981** (26.5/27) |
| Answer correctness | **0.944** (three 0.5 verdicts: q12, q16, q22) |
| Answer relevancy | **0.800** (ambiguous questions 0.684; terse question wording lowers the score) |

How these were checked:

- The gate-1 sweep matches the committed `retrieval-hybrid.md`.
- The full-run numbers match `full-evaluation-metrics.json` (run `20260824-110510-full-hybrid`).
- Every generation and abstention figure was **recomputed from the run's local `per_question.jsonl`**. The recompute shows 13 refusals: 2 by gate 1 (q38 and q39, both off-domain) and 11 by gate 2 (10 unanswerable questions plus the wrongly refused q25).
- Judge accounting adds up: 27 answered × 3 judge tasks = 81 = 69 calls + 12 cache hits.
- The separate Phase-4 log (`abstention-phase4.log`) reproduces the same abstention numbers.

## Status, completeness & notable gaps

- **Status: complete as a challenge submission.** CLI, API and UI all work. All four metric families the brief names are measured, plus abstention, and CI gates retrieval quality on every push. Done in a single afternoon (11 commits, Phases 0–7 of `docs/PLAN.md`). Phase 8 ("Polish") is not marked done and the "Definition of done" checklist is unchecked.
- **Evaluated only on a self-authored dev set.** The brief says MLSC will provide its own evaluation set. `dataset.from_records` exists to load it, but it is only exercised by tests and not wired into the CLI. EVALUATION.md notes that the dev set was written by the person who built the system.
- **Single LLM backend.** Only Gemini is implemented, and the same model answers and judges. EVALUATION.md flags the self-preference bias risk. The mitigation it suggests (switching the judge provider in config) is not wired up; see below.
- **Not done:** the RAGAS cross-check (no RAGAS code anywhere; EVALUATION.md blames free-tier quota), the planned position/verbosity bias re-judging, any reranker (the port exists, the retriever never calls it), the `fixed` chunker (config enum value only), and the `sbert`/`chroma` adapters.
- **Diversification is inert on this set.** By the project's own ablation, multi-doc coverage is 1.000 with the cap set to 3 and with it effectively off.
- **Dead or unwired code:** `provider_unavailable_answer()` is never called, although its docstring says the CLI and the harness use it. `GeminiProvider.stream()` is never called, because SSE sends the answer whole. `tenacity` is declared but not imported. Config keys `evaluation.judge.provider/model`, `retrieval.rerank.*` and `abstention.calibrated` are never read.
- **Small robustness notes:**
  - Gate 3 fails open when its own JSON is malformed.
  - `AskRequest.question` has a hard pydantic `max_length=1000`, so raising `api.max_question_length` above 1000 has no effect.
  - The HTTP eval job store is in memory only (documented).
  - With no API key, `/v1/ask` returns a **500** `configuration-error`, not the 503 the API doc promises.

## README vs. code

The docs are unusually candid about measured negative results: wrong Phase 2 predictions, inert diversification, the q25 over-refusal, the judge disagreeing with the human spot-check. Every **number** spot-checked matched its artefact. The discrepancies are in **capability claims**, mostly features that were planned and documented but never written:

1. **"Anthropic, OpenAI, Groq and Ollama are supported through the same interface"** (README) and **D8 "adapters for Gemini, Anthropic, OpenAI, Groq and Ollama"**: false. `generation/providers/` holds only `gemini.py`. The registry's other four builders import modules that do not exist and raise `ConfigurationError`. The registry docstring itself says they are "declared … ahead of being written".
2. **`mlsc calibrate`** (README CLI table, ARCHITECTURE §6, EVALUATION §4): there is no such command. The threshold sweep runs inside `mlsc eval` and needs **no** key, while the README table says `calibrate` needs one.
3. **"`evaluation.judge.provider` points the judge at a different backend in one line"** (EVALUATION.md, `judge.py` docstring, the generated report text): the config key is never read. `_run_full` always passes the answering provider to `Judge(...)`. As a result `config.yaml`'s `judge.model: gemini-2.5-flash` is dead, and the reported run's judge was in fact `gemini-3.1-flash-lite`, as `full-evaluation.md` records. Switching judges would need a code change.
4. **Repository layout "stores/ VectorStore adapters (numpy, chroma)"** and **"retrieval/ … reranking"**, plus D2 "Chroma as an adapter": only `numpy_store.py` exists, and no reranker exists. `factories.py` imports `stores.chroma_store` and `embeddings.sbert_embedder`, which are both missing. Selecting either one reports "not installed — run `pip install -e ".[chroma]"`", which would not fix it.
5. **D8 "Gemini 2.5 Flash is the default"**: the configured default is `gemini-3.1-flash-lite`, and `config.yaml` explains why (free-tier daily limits).
6. **README sample output cites `[leadership::c03]`** for the domain-lead responsibilities. The built index has only `leadership::c00`–`c02`, and the dev set labels that fact `leadership::c01`. The sample output is illustrative, not real.
7. **API.md drift:**
   - `POST /v1/eval/runs` is shown accepting `"metrics": ["retrieval","generation","abstention"]` and a `top_k`. In the code, `MetricFamily` has only `retrieval`, so other values get a 422, and `top_k` is ignored.
   - The status table lists validation failures as `400`. Schema validation actually returns **422**.
   - "No key configured" is listed as `503`. It is actually a **500** `configuration-error`.
   - The "generation metrics in Phase 7 will change that" note is stale. Phase 7 shipped for the CLI only.
8. **Minor:**
   - README says the judge is "cached **and seeded**". No seed is passed anywhere; only temperature 0.
   - "Metrics are implemented in-repo (~200 readable lines)": the three metric modules total 626 lines, including docstrings and judge prompts.
   - EVALUATION.md's methodology promises a shuffled-claim bias check (not implemented) and roughly ten hand-graded spot-checks (the results report four).
   - The Results preamble still says generation metrics "arrive in Phase 7", directly above the Phase 7 results.
   - The dev set's 12/40 unanswerable share falls just short of the stated "at least a third".
9. **Not reproducible from artefacts:** the ablation table (no contextual headers: recall 0.848; no low-IDF filter: 0.920; `rrf_k`/`candidate_k`/cap sweeps) has no committed report and no matching local run. Every local run directory has `prepend_doc_title: true`. The numbers are plausible, because each knob is a config value, but they cannot be verified.

Claims that **are** borne out by the code: no vector DB (NumPy `18×384`); retrieval and the CI gate run without a key; refusal is a parsed schema field; cited ids are validated mechanically; abstention is HTTP 200; RFC 9457 errors with trace ids; a UI that makes no external requests (enforced by a test); a no-hard-coded-answers test exists; 6 documents / 18 chunks; every retrieval, abstention and generation metric quoted in the README and EVALUATION.md.

---

*Doc generated 2026-10-12 by reading the source at C:\Users\ameya\Documents\MLSC\mlsc-knowledge-assistant. Source of truth: https://github.com/AmeyaBorkar/mlsc-knowledge-assistant*
