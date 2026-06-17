# Tesseract

> A FastAPI cargo X-ray analysis service that sends customs scan images to a hosted vision-language model (Qwen2.5-VL-7B-Instruct, via the HuggingFace Inference API) with a "customs analyst" prompt, then parses the free-text reply into a structured risk score, detected-object list, and officer recommendations. It is inference-only — it calls a remote model, runs no local weights, and contains no fine-tuning code. Ships with a single-page vanilla-JS dashboard served by the same app.

| Field | Value |
|---|---|
| Repository | https://github.com/AmeyaBorkar/Tesseract |
| Visibility | Public |
| Category | ml-research |
| Primary language(s) | Python (backend), JavaScript / HTML / CSS (frontend) |
| Local path | `C:\Users\ameya\Documents\Tesseract` |
| Default branch | `main` |
| Lines of code (computed) | Python: 1,135 across 17 files (8 are empty/1-line `__init__.py`; ~1,134 real). Frontend: `app.js` 526, `index.html` 418, `styles.css` 1,263. Total ≈ 3,342 lines. |
| Source files (computed) | 17 `.py` files + 3 frontend files (`index.html`, `app.js`, `styles.css`) = 20 source files |
| Key dependencies | `fastapi==0.115.0`, `uvicorn[standard]==0.30.6`, `huggingface-hub>=1.7.0`, `Pillow==10.4.0`, `pydantic-settings==2.5.2`, `python-dotenv==1.0.1`, `python-multipart==0.0.9` |
| License | README says **MIT**, but there is **no `LICENSE` file** in the repo (discrepancy) |
| Last commit | `6b0aa30` — "docs: update READMEs with frontend, guardrails, and full configuration reference" (2026-03-28). 8 commits total. |

## What it actually is

Tesseract is a small but cleanly-structured web service for assisting customs / border-security officers. An officer uploads a cargo scan or X-ray image (single image, or two images for comparison); the backend validates and preprocesses it, base64-encodes it into a data URL, and sends it together with a long natural-language "customs analyst" prompt to a hosted Qwen2.5-VL-7B-Instruct model. The model returns free-form text; a regex/JSON parser (`risk_scorer.py`) extracts a 0.0–1.0 risk score, a LOW/MEDIUM/HIGH/CRITICAL level, a list of detected objects with confidence and a category (normal/suspicious/restricted/prohibited), plus a generated recommendation string. The result is returned as structured JSON and rendered in a dark "security-ops" dashboard with an animated risk gauge.

Key reality check on the "model" angle:
- There is **no local model, no GPU inference, no transformers `from_pretrained`, no vLLM, and no quantization** in this codebase. It is a thin client over HuggingFace's hosted **`InferenceClient.chat_completion`** API (`backend/app/services/vlm_service.py:100`).
- There is **no fine-tuning, no training loop, no dataset loaders, and no evaluation harness**. The only image asset is a single screenshot in `TestImages/` (git-ignored).
- The "X-ray vision-language model" is therefore really "a general VLM (Qwen2.5-VL) prompted to act as a customs analyst," not a model trained on X-ray data.

The repo also includes a problem-statement PDF (`Problem Statement - CIIBS.pdf`) suggesting this was built for a hackathon/challenge (CIIBS — likely a customs/border security challenge).

## Architecture & how it's structured

Flow: **image upload → FastAPI route → validation → Pillow preprocessing → base64 data URL → HuggingFace VLM chat-completion → regex/JSON parse → structured JSON → dashboard render.**

```
Tesseract/
├── README.md                       # Top-level docs (claims, structure, env reference)
├── Problem Statement - CIIBS.pdf   # Challenge brief (not code)
├── TestImages/                     # 1 screenshot, git-ignored
├── frontend/                       # Vanilla SPA, served by FastAPI (no build step)
│   ├── index.html  (418 lines)     # Sidebar + Dashboard/Analyze/Compare views
│   ├── app.js      (526 lines)     # Fetch calls, dropzones, gauge animation, toasts
│   └── styles.css  (1,263 lines)   # Dark security-ops theme
└── backend/
    ├── requirements.txt            # 7 pinned deps
    ├── .env / .env.example         # Config (.env is git-ignored)
    ├── README.md                   # Backend-specific quick start
    └── app/
        ├── main.py                 # FastAPI app factory, lifespan, middleware stack,
        │                           #   exception handler, frontend serving + /static mount
        ├── config.py               # Pydantic-settings Settings (model id, thresholds, limits)
        ├── dependencies.py         # verify_api_key — optional X-API-Key auth
        ├── middleware.py           # SecurityHeaders + RequestTracing + RateLimit (in-memory)
        ├── routes/
        │   ├── health.py           # GET  /api/v1/health
        │   ├── analysis.py         # POST /api/v1/analyze   (1 image)
        │   └── comparison.py       # POST /api/v1/compare   (2 images)
        ├── services/
        │   ├── vlm_service.py      # HuggingFace InferenceClient + the two prompts
        │   ├── image_processor.py  # Pillow resize/contrast/sharpen (+ unused X-ray variant)
        │   └── risk_scorer.py      # Free-text → structured response parser
        ├── models/
        │   └── schemas.py          # Pydantic response models + enums
        └── utils/
            └── image_utils.py      # Extension/magic-byte/size validation, base64 helpers
```

Middleware is registered in `main.py:67-88` (FastAPI applies them bottom-to-top): CORS → RateLimit → RequestTracing → SecurityHeaders. Routes are mounted under `API_PREFIX = "/api/v1"`. The frontend is served at `/` via `FileResponse`, with `/static` mounted for CSS/JS.

## The VLM in code — exact model, loading/serving, quantization, fine-tuning vs inference

- **Exact model:** `Qwen/Qwen2.5-VL-7B-Instruct` — the default of `settings.MODEL_ID` (`config.py:15`), overridable via the `MODEL_ID` env var. The README's claim of Qwen2.5-VL-7B is accurate.
- **How it's loaded/served:** It is **not loaded locally at all.** `vlm_service.py:84` constructs a `huggingface_hub.InferenceClient(token=settings.HF_API_TOKEN, timeout=settings.VLM_TIMEOUT)` and calls `self.client.chat_completion(model=self.model_id, messages=..., max_tokens=1500)` (`vlm_service.py:100-105`). This routes through HuggingFace's hosted Inference API / providers. No weights are downloaded; `transformers`, `torch`, and `vllm` are **absent from `requirements.txt`.**
- **Quantization:** None. There is no bitsandbytes, GPTQ, AWQ, dtype, or device-map configuration anywhere — quantization (if any) is whatever the remote provider applies.
- **Fine-tuning vs inference:** **Inference-only.** No training scripts, no LoRA/PEFT, no datasets, no optimizers. The "specialization" for cargo X-ray comes entirely from prompt engineering, not weights.
- **Image transport:** The PIL image is JPEG-encoded and base64-embedded as a `data:image/jpeg;base64,...` URL (`vlm_service.py:90-93`), passed in the OpenAI-style multimodal `messages` content array (`{"type": "image_url", "image_url": {"url": ...}}`).
- **Concurrency:** The blocking `chat_completion` call is wrapped in `asyncio.to_thread` (`vlm_service.py:130`, `:164`) so it doesn't block the event loop. `InferenceTimeoutError` is caught and surfaced as a friendly `RuntimeError`.
- **Lifecycle:** A module-level singleton `vlm_service` is `initialize()`-d in the FastAPI `lifespan` startup (`main.py:39`); `initialize()` raises if `HF_API_TOKEN` is unset.

## Prompting & reasoning — how the contraband-detection prompt is constructed

Two hard-coded system prompts live as module constants in `vlm_service.py`:

- **`ANALYSIS_PROMPT` (lines 16-46):** Frames the model as "an expert customs and border security cargo image analyst" and demands a fixed 6-part structure: (1) Summary, (2) `Risk Score: X.XX` (0.0–1.0), (3) Risk Level (LOW/MEDIUM/HIGH/CRITICAL), (4) Detected Objects as a JSON array of `{name, confidence, description}`, (5) a detailed Explanation covering concealment indicators (unusual density, hidden compartments, double walls), shape/pattern anomalies, and manifest-mismatch reasoning, and (6) Recommendations. It instructs the model to err toward flagging and to still analyze non-X-ray images for security-relevant content.
- **`COMPARISON_PROMPT` (lines 49-69):** Frames a two-image comparison task — differences, missing items, density/shape changes indicating tampering, concealment indicators — with a 4-part output (Summary, `Risk Score`, Differences Found bullets, Explanation).

The prompt is sent as a **single user message** whose `content` array interleaves the prompt text and one or two `image_url` entries (`vlm_service.py:119-127` for analyze, `:152-161` for compare). There is no separate "system" role message — the analyst framing is folded into the user turn. `max_tokens` defaults to 1500.

**Reasoning is post-hoc parsed, not enforced.** Because the model returns free text, `risk_scorer.py` does the heavy lifting:
- `parse_risk_score` (lines 27-62) tries 6 regex patterns (`risk score: 0.75`, `7/10`, `75/100`, `0.7/1.0`, etc.), normalizes to 0–1, and falls back to level-keyword-derived scores (LOW 0.15 / MEDIUM 0.50 / HIGH 0.75 / CRITICAL 0.95).
- `parse_detected_objects` (lines 94-147) first attempts to `json.loads` the first `[...]` array; on failure it falls back to bullet/numbered-list regex extraction.
- `classify_category` (lines 65-91) is a **keyword lookup, not model output** — it maps words like "gun/explosive/narcotic" → `prohibited`, "knife/chemical/lithium" → `restricted`, "hidden/concealed/anomal" → `suspicious`, else `normal`.
- `build_analysis_response` (lines 150-183) reconciles score vs level against thresholds (`config.py:23-25`: 0.3/0.6/0.8) and **synthesizes the `recommendations` string in Python** from the final risk level (e.g. "IMMEDIATE REVIEW RECOMMENDED…") — the model's own recommendations text is not used for that field. The full raw VLM text is preserved in `explanation`.

Net effect: the score/level/objects shown to the officer are a deterministic Python interpretation of the model's prose, which makes the pipeline brittle if the model deviates from the requested format.

## API surface — FastAPI endpoints

All under prefix `/api/v1`. Analyze and compare depend on `verify_api_key` (enforced only when `API_KEY` is configured).

| Method | Path | Handler | Auth | Request | Response model |
|---|---|---|---|---|---|
| GET | `/api/v1/health` | `health.health_check` | none | — | `HealthResponse` (status healthy/degraded, app_name, version, model_id) |
| POST | `/api/v1/analyze` | `analysis.analyze_image` | optional `X-API-Key` | `multipart/form-data` field `file` | `AnalysisResponse` (risk_score, risk_level, summary, detected_objects[], explanation, recommendations) |
| POST | `/api/v1/compare` | `comparison.compare_images` | optional `X-API-Key` | `multipart/form-data` fields `file1`, `file2` | `ComparisonResponse` (risk_score, risk_level, differences_found, summary, differences[], explanation) |
| GET | `/` | `serve_frontend` | none | — | Frontend `index.html` (or JSON fallback if missing) |
| — | `/static/*` | StaticFiles mount | none | — | CSS/JS assets |
| GET | `/docs`, `/redoc` | FastAPI built-ins | none | — | Only enabled when `DEBUG=True` (`main.py:63-64`) |

Upload validation (in both POST routes): extension allowlist (jpg/jpeg/png/bmp/webp/tiff), non-empty check, ≤20 MB size, and magic-byte content sniffing (`image_utils.py:31-41`). Error responses are documented for 400/401/403/429/500.

## Frontend

Yes — a self-contained single-page app under `frontend/`, served by FastAPI, **no build step / no npm**, matching the README's "zero build dependencies" claim.

- `index.html` (418 lines): sidebar with three views (Dashboard, Analyze Scan, Compare Scans), an optional API-key input in the topbar, SVG risk-gauge markup, dropzones, and result blocks.
- `app.js` (526 lines): vanilla JS — `apiFetch` wrapper that injects the `X-API-Key` header from the topbar input; drag-and-drop + click-to-browse dropzones with client-side type/size validation; `runAnalysis`/`runComparison` posting `FormData` to the API; `animateGauge` driving an SVG `stroke-dashoffset` gauge color-coded by threshold; detected-object cards with confidence bars; toast notifications; and a `checkHealth` poll every 60s that updates the status dot and model-id display.
- `styles.css` (1,263 lines): dark security-ops theme with CSS variables for risk colors (`--risk-low/medium/high/critical`), responsive mobile sidebar.

## Datasets & evaluation

**None present.** No dataset directory, no labeled X-ray corpus, no metrics, no eval/benchmark scripts, no ground-truth, no test suite. The only image is `TestImages/Screenshot 2026-03-23 235613.png` (a single git-ignored screenshot, not a dataset). There are no automated tests of any kind in the repo. Accuracy claims cannot be substantiated from the code.

## Tech stack & dependencies

- **Backend:** FastAPI 0.115.0 on Uvicorn (standard), Python 3.10+ (uses `str | None` unions and `list[...]` generics). Pydantic v2 + `pydantic-settings` for config and response models. `python-multipart` for uploads, `python-dotenv` for `.env`.
- **VLM client:** `huggingface-hub>=1.7.0` (`InferenceClient.chat_completion`).
- **Image processing:** Pillow 10.4.0 — resize-to-1920, contrast ×1.3, `SHARPEN` (`image_processor.py`); decompression-bomb guard `Image.MAX_IMAGE_PIXELS = 50_000_000`.
- **Frontend:** plain HTML/CSS/JS, Google Fonts (Inter) via CDN, no framework.
- **Hardening (real, in code):** in-memory sliding-window rate limiter per IP (`middleware.py`), security headers (HSTS, X-Frame-Options, nosniff, Referrer-Policy, Permissions-Policy), request-ID tracing, global exception handler that hides internals, optional API-key auth, configurable CORS.

Note: the rate limiter and request-tracking buckets are in-process dicts (`middleware.py:71`) — they are per-worker and not shared across multiple Uvicorn workers or restarts.

## Build / run (actual commands)

```bash
git clone https://github.com/AmeyaBorkar/Tesseract.git
cd Tesseract/backend
pip install -r requirements.txt
cp .env.example .env          # then set HF_API_TOKEN=hf_...
uvicorn app.main:app --reload --port 8000
```

Then open http://localhost:8000 for the dashboard. Set `DEBUG=True` in `.env` to expose `/docs`. Startup will raise if `HF_API_TOKEN` is missing (`vlm_service.initialize`). Configurable via env vars: `MODEL_ID`, `MAX_IMAGE_SIZE` (1920), `MAX_FILE_SIZE` (20 MB), `API_KEY`, `ALLOWED_ORIGINS`, `RATE_LIMIT_REQUESTS`/`_WINDOW` (30/60), `VLM_TIMEOUT` (120s), `LOG_LEVEL`.

## Status, completeness & notable gaps

Working / complete:
- End-to-end happy path is coherent: upload → validate → preprocess → VLM → parse → JSON → dashboard.
- Genuinely thorough input validation and a sensible production-hardening middleware stack.
- Polished, dependency-free frontend.

Gaps & caveats:
- **Inference-only, remote-model** — no local model, no fine-tuning, no quantization, no training; the "X-ray VLM" is a general model steered by a prompt.
- **No tests, no datasets, no evaluation** — zero quantitative validation of detection quality.
- **Parser fragility** — structured output is reconstructed from prose with regex; `recommendations` and `category` are Python-derived, not model-derived. Confidence is hard-defaulted to 0.5 when absent (`risk_scorer.py:110`, `:136`).
- **Dead code:** `enhance_xray_image` (`image_processor.py:34`) and several helpers (`bytes_to_base64`, `get_image_info`) are defined but never called; the analyze pipeline uses the milder `preprocess_image` instead.
- **No `LICENSE` file** despite README saying MIT.
- In-memory rate limiting won't scale across multiple workers.
- A `.claude/` directory and a committed `.env` reference exist locally (`.env` itself is git-ignored).

## README vs. code

Mostly accurate, with a few flags:

- **Model:** README "Qwen2.5-VL-7B-Instruct via HuggingFace Inference API" — **accurate** (`config.py:15`, `vlm_service.py:84-105`).
- **Endpoints, env vars, guardrails tables:** **accurate** and match the code (defaults, header names, thresholds all line up).
- **"zero build dependencies" frontend:** **accurate.**
- **License = MIT:** **discrepancy** — no `LICENSE` file exists in the tree.
- **README example responses:** illustrative JSON, fine; field names match `schemas.py`.
- **README never overclaims fine-tuning or training** — it correctly frames this as Inference-API usage. The main thing the README doesn't make explicit is that risk level, category, and recommendations are computed by deterministic Python post-processing rather than emitted directly by the model.
- Project-structure tree in README matches the actual layout (it omits `dependencies.py` in one place but lists it elsewhere; otherwise correct).

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\Tesseract. Source of truth: https://github.com/AmeyaBorkar/Tesseract*
