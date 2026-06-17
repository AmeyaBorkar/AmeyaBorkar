# Personal Health Assistant

> A full-stack **AI nutrition & fitness coaching** web app — a Flask REST API backend plus a React (Vite) single-page frontend. Users register a health profile, set a weight goal, and log meals by name; the backend calls a **Cerebras-hosted Llama-4-Scout LLM** to (a) estimate calories/macros from a free-text food description, (b) give per-meal coaching feedback, (c) compute calorie/macro targets, (d) chat with full user context, and (e) generate weekly summaries. It is **not** a symptom checker, disease-prediction ML model, or medical diagnosis tool — there is no trained model and no dataset. All "intelligence" is LLM prompt engineering against Cerebras.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/PersonalHealthAssistant |
| **Visibility** | Public |
| **Category** | apps |
| **Primary language(s)** | Python (Flask backend), JavaScript/JSX (React frontend) |
| **Local path** | `C:\Users\ameya\.repo-cache\PersonalHealthAssistant` |
| **Default branch** | `main` |
| **Lines of code (computed)** | Python: **2,674** across 14 `.py` files; Frontend JS/JSX: **1,147** across 11 files (`frontend/src`); plus 394 lines of `index.css` |
| **Source files (computed)** | 14 Python files; 11 JS/JSX files (9 `.jsx` + 2 `.js`) under `frontend/src` |
| **Key dependencies** | Backend: `Flask 3.0.0`, `Flask-SQLAlchemy 3.1.1`, `Flask-JWT-Extended 4.6.0`, `Flask-CORS 4.0.0`, `bcrypt 4.1.2`, `cerebras-cloud-sdk 1.0.0`, `python-dotenv 1.0.0`. Frontend: `react 19.2`, `react-dom 19.2`, `react-router-dom 7.13`, `vite 7.3` |
| **License** | MIT (Copyright (c) 2026 Ameya Borkar) |
| **Last commit** | `2ee84f1` — *"feat: complete Personal Health Assistant overhaul"* — 2026-03-02 |

---

## What it actually is

A **two-tier web application**:

1. **Backend** — a Flask JSON REST API (URL prefix `/api/v1`) with JWT auth, SQLAlchemy ORM over SQLite (MySQL-compatible), and a single LLM integration module (`cerebras_service.py`). It exposes endpoints for auth, user profile, meals, goals, water/achievements tracking, AI chat, and a dashboard.
2. **Frontend** — a React 19 + Vite SPA (`frontend/`) with client-side routing (React Router 7), a thin `fetch` API client, JWT stored in `localStorage`, and pages for Dashboard, Meals, AI Chat, Goals, and Profile.

The health "logic" is entirely **LLM prompt-driven**, not ML. There is no dataset, no scikit-learn/PyTorch model, no symptom→disease mapping. The grep across the repo for `openai|anthropic|google.generativeai|groq|transformers|langchain|cohere|ollama` returns **only Cerebras** — the sole AI provider is `cerebras.cloud.sdk.Cerebras`, called with the OpenAI-style `client.chat.completions.create(...)` interface, against model `llama-4-scout-17b-16e-instruct` (configurable via env). Note: deterministic BMR/TDEE/macro math also exists in `utils.py` as a fallback path.

---

## Architecture & how it's structured

```
PersonalHealthAssistant/
├── run.py                 # Entry point — starts Flask dev server on PORT (default 8000)
├── app.py                 # App factory: extensions, CORS, JWT error handlers, db.create_all()
├── config.py              # Env-driven config + ALL LLM prompt templates + achievement defs (427 LOC)
├── database.py            # 9 SQLAlchemy models (298 LOC) — UUID PKs, to_dict() serializers
├── cerebras_service.py    # LLM integration — 8 functions wrapping Cerebras chat completions (599 LOC)
├── utils.py               # Responses, bcrypt, LLM-JSON parsing, BMR/TDEE/macro math, validation (274 LOC)
├── routes/
│   ├── __init__.py        # register_blueprints() — wires 7 blueprints into the app
│   ├── auth.py            # register / login / refresh / logout (JWT, bcrypt, refresh tokens)
│   ├── user.py            # GET/PUT profile, GET stats
│   ├── meals.py           # log meal (AI nutrition + feedback), today, week, delete
│   ├── ai.py              # chat, history (paginated), weekly-summary
│   ├── goals.py           # set goal (LLM target calc w/ deterministic fallback), current
│   ├── tracking.py        # water log/today, achievements (read-only)
│   └── analytics.py       # dashboard aggregate
├── frontend/              # React + Vite SPA
│   ├── vite.config.js     # dev proxy: /api + /health → http://localhost:8000
│   └── src/
│       ├── main.jsx, App.jsx        # router + sidebar layout + ProtectedRoute
│       ├── AuthContext.jsx          # auth state (localStorage-backed)
│       ├── api.js                   # fetch-based API client (Bearer token, auto-logout on 401)
│       ├── index.css                # 394 lines, dark glassmorphism theme
│       └── pages/  Login, Register, Dashboard, Meals, Chat, Goals, Profile
├── requirements.txt, .env.example, LICENSE, README.md
```

**Control flow (meal logging — the showcase path):**
`Meals.jsx` → `api.logMeal()` → `POST /api/v1/meals/log` (JWT-guarded) → `cs.extract_nutrition_from_food()` (LLM call #1, returns JSON macros) → sums today's meals → `cs.generate_meal_feedback()` (LLM call #2, returns coaching text) → persists a `Meal` row (with `ai_feedback`) → increments `UserStats.total_meals_logged` → returns `meal.to_dict()`.

**Data flow:** browser holds JWT in `localStorage`; every request carries `Authorization: Bearer <token>`; backend resolves `get_jwt_identity()` → user UUID → SQLAlchemy queries; LLM context is assembled per-request from DB (profile, active goal, today's meals, water, stats, full chat history).

---

## Code walkthrough (the important parts)

### `app.py` (94 LOC) — application factory
`create_app()` configures `SECRET_KEY`, JWT secrets/expiries, `SQLALCHEMY_DATABASE_URI`; initializes `db`, `JWTManager`, and CORS (restricted to localhost dev origins for `/api/*`). Registers all blueprints, defines `/` and `/health`, 404/500 handlers (500 does `db.session.rollback()`), and JWT error callbacks returning the standard error envelope. Calls `db.create_all()` inside `app_context()` at import time, then instantiates `app = create_app()` for `run.py` to import.

### `config.py` (427 LOC) — config + prompt library
The real heart of the "AI" lives here as **8 prompt templates**: `NUTRITION_EXTRACTION_PROMPT`, `BASE_SYSTEM_PROMPT` (large context block with profile/goal/today's progress/recent meals/stats/conversation summary), `MEAL_ANALYSIS_PROMPT`, `WEEKLY_SUMMARY_PROMPT`, `RECIPE_GENERATION_PROMPT`, `SUPPLEMENT_ADVICE_PROMPT`, `GOAL_CALCULATION_PROMPT`, `CHAT_SUMMARIZATION_PROMPT`, and `WORKOUT_SUGGESTION_PROMPT`. Also defines `CEREBRAS_CONFIG` (api_key, primary/summary model + token/temperature settings), JWT expiries, an `ACHIEVEMENTS` dict (10 badge definitions), validation bounds, and enum lists (meal types, genders, activity levels, goal types).

### `cerebras_service.py` (599 LOC) — LLM integration
Initializes one module-level `client = Cerebras(api_key=...)`. Eight functions, each formatting a prompt and calling `client.chat.completions.create(model=..., messages=[...], max_completion_tokens=..., temperature=..., stream=False)`:
- `extract_nutrition_from_food()` — system "respond with valid JSON only", temp 0.3, parsed via `extract_json_from_llm_response`.
- `generate_meal_feedback()` — 2–3 sentence coaching, temp 0.7, max 300 tokens.
- `chat_with_ai()` — builds the big system prompt, appends the last `MAX_CHAT_HISTORY_FOR_CONTEXT` (=10) messages verbatim, summarizes older history via `summarize_chat_history()`, returns `{ai_response, tokens_used}`. On any exception returns a graceful fallback string.
- `summarize_chat_history()` — uses the `summary_model`.
- `generate_weekly_summary()` — 4–6 paragraph motivational report.
- `calculate_goal_targets()` — LLM computes TDEE/BMR/calories/macros as JSON (Mifflin-St Jeor in the prompt), temp 0.3.
- `generate_recipe()` — temp 0.8 for creativity. **No route calls this.**
- `generate_supplement_advice()` — evidence-based supplement advice. **No route calls this.**

Every function is wrapped in try/except returning `None` (or a fallback) on failure — there is no retry logic.

### `database.py` (298 LOC) — 9 models
`User`, `Goal`, `Meal`, `ChatHistory`, `WaterLog`, `Achievement`, `UserStats`, `Workout`, `RefreshToken`. All use `String(36)` UUID primary keys (`generate_uuid`), `created_at`/`updated_at` timestamps, cascade-delete relationships off `User`, and a `to_dict()` serializer. `ChatHistory.context_data` stores JSON as text "for SQLite compat". `Achievement` has a unique `(user_id, badge_name)` constraint.

### `utils.py` (274 LOC) — helpers
Standardized `success_response`/`error_response`/`paginated_response` envelopes (all carry `success`, `message`, ISO `timestamp`). bcrypt `hash_password`/`verify_password`. **`extract_json_from_llm_response`** — robust LLM-JSON extraction (tries raw `json.loads`, then ```json fences, then a `{...}` regex). Deterministic health math: `calculate_bmi`, `calculate_bmr` (Mifflin-St Jeor), `calculate_tdee` (activity multipliers), `calculate_macro_targets`, `calculate_calorie_adjustment` (-500 loss / +300 gain), `calculate_weeks_to_goal`. Email/password validators, HTML-tag sanitizer, and `check_achievements_earned` (**defined but never called anywhere**).

### Routes
- **`auth.py`** — `register` validates fields, email, password strength (≥8 chars, letter+digit), uniqueness; hashes password; creates `User` + seeds `UserStats` with computed BMI/TDEE; issues access + refresh tokens and persists a `RefreshToken`. `login` verifies bcrypt and issues tokens. `refresh` mints a new access token. `logout` marks refresh tokens `is_revoked=True` (note: token validity isn't actually re-checked on use — see gaps).
- **`meals.py`** — described above; also `today` (totals + goal_progress %), `week` (per-day breakdown + averages), and `DELETE /<meal_id>` (decrements `total_meals_logged`).
- **`ai.py`** — `chat` assembles full DB context, calls `chat_with_ai`, persists both user and assistant `ChatHistory` rows; `history` is paginated; `weekly-summary` aggregates the last 7 days, computes adherence %, top foods (`collections.Counter`), then calls the LLM.
- **`goals.py`** — `set` tries the LLM (`calculate_goal_targets`); **if it returns None, falls back to deterministic `utils` math**; deactivates prior active goals; creates the new `Goal`. `current` returns progress %/days remaining (`on_track` is hardcoded `True`).
- **`tracking.py`** — water logging with progress %, and a **read-only** achievements endpoint.
- **`analytics.py`** — single `dashboard` aggregate (today's totals, water, goal progress, streak, recent achievements, a hardcoded `motivational_quote`).

### Frontend (`frontend/src`)
- **`api.js`** — `ApiClient` class wrapping `fetch` against `/api/v1`; injects `Bearer` token; on a 401 it clears the token and redirects to `/login`. Exposes typed methods mirroring every backend endpoint.
- **`App.jsx`** — `BrowserRouter` + `AuthProvider`; `ProtectedRoute` redirects unauthenticated users to `/login`; sidebar `Layout` with emoji nav (Dashboard, Meals, AI Chat, Goals, Profile).
- **`AuthContext.jsx`** — auth state persisted to `localStorage`.
- **`pages/Chat.jsx`** — loads history, sends messages with a category selector (general/diet/workout/supplements), optimistic UI, auto-scroll.
- Other pages: `Login`, `Register`, `Dashboard`, `Meals`, `Goals`, `Profile`.

---

## Core capability — exactly how the health logic works (LLM, not ML)

**Provider/SDK (verified from code):** `cerebras-cloud-sdk==1.0.0`, imported as `from cerebras.cloud.sdk import Cerebras` in `cerebras_service.py`. The client is `Cerebras(api_key=config.CEREBRAS_CONFIG['api_key'])` and all calls use the OpenAI-compatible `client.chat.completions.create(...)` shape.

**Model:** `llama-4-scout-17b-16e-instruct` (Meta's Llama 4 Scout, served by Cerebras) for the primary model **and** the summary model. Both are overridable via `CEREBRAS_PRIMARY_MODEL` / `CEREBRAS_SUMMARY_MODEL`. Primary: 8000 max tokens, temp 0.7. Summary: 2000 tokens, temp 0.5.

**No ML / no dataset / no medical inference.** There is no trained classifier, no symptom checker, no disease prediction. "Nutrition extraction" is the LLM returning estimated macros as JSON from a food name + quantity (system prompt forces JSON; `extract_json_from_llm_response` parses it). Confidence is just a string the model self-reports. Goal target calculation is *delegated to the LLM* (BMR/TDEE/macros), with the deterministic Mifflin-St Jeor implementation in `utils.py` used only as a fallback when the LLM call fails.

**Context engineering:** the chat path injects a large `BASE_SYSTEM_PROMPT` populated with the user's profile, active goal, today's calories/macros/water vs. targets, last 5 meals, streak/BMI/TDEE stats, and a running conversation summary. The full chat history is fetched per request; only the most recent 10 messages go in verbatim, older ones are LLM-summarized to save context — a simple but real long-conversation strategy.

---

## Interface — how a user interacts

**Web (browser SPA).** There is no CLI and no desktop GUI. The user flow: Register/Login (React forms) → Set a goal → Log meals by name (AI extracts nutrition + gives feedback) → view Dashboard → Chat with the AI. The Vite dev server (port 5173) proxies `/api` and `/health` to the Flask backend (port 8000), so in dev both run side-by-side. JWT access token lives in `localStorage`; expired/invalid tokens trigger an automatic redirect to `/login`.

The backend is a pure JSON API and could equally be driven by curl/Postman; the README documents the endpoint table.

---

## Data & persistence

- **Database:** SQLAlchemy ORM. Default `DATABASE_URL = sqlite:///health.db` (auto-created via `db.create_all()` at startup, zero setup). MySQL is supported by swapping the connection string (`mysql+pymysql://...`) — models are written to be cross-compatible (UUID strings, JSON-as-text).
- **Tables (9):** users, goals, meals, chat_history, water_logs, achievements, user_stats, workouts, refresh_tokens.
- **Auth/secrets:** passwords bcrypt-hashed; JWT access (1h) + refresh (30d) tokens; refresh tokens persisted in `refresh_tokens` with an `is_revoked` flag. `CEREBRAS_API_KEY` and Flask/JWT secrets come from `.env` (see `.env.example`).
- **AI persistence:** every chat turn (user + assistant) is stored in `chat_history` with `tokens_used`; each meal stores its LLM `ai_feedback` text inline.

---

## Tech stack & dependencies

**Backend:** Python / Flask 3, Flask-SQLAlchemy, Flask-JWT-Extended, Flask-CORS, bcrypt, python-dotenv, cerebras-cloud-sdk. (`requests` is pinned in `requirements.txt` but not actually imported anywhere in the Python sources.)

**Frontend:** React 19, React DOM 19, React Router 7, Vite 7, ESLint 9. Plain `fetch` for HTTP (no axios), vanilla CSS (dark glassmorphism, 394 lines) — no Tailwind, no component library.

**Database:** SQLite (default) or MySQL.

---

## Build / run (actual commands)

**Backend:**
```bash
pip install -r requirements.txt
cp .env.example .env          # then set CEREBRAS_API_KEY (and secrets)
python run.py                 # serves on http://localhost:8000 (debug=True, host 0.0.0.0)
```
SQLite DB `health.db` is created automatically on first run.

**Frontend:**
```bash
cd frontend
npm install
npm run dev                   # Vite dev server at http://localhost:5173, proxies /api → :8000
```
Production frontend build: `npm run build` (→ `dist/`), preview via `npm run preview`. There is **no** production WSGI config (Gunicorn/uWSGI) — `run.py` uses Flask's dev server with `debug=True`.

---

## Status, completeness & notable gaps

**Working end-to-end:** auth (register/login/refresh/logout), profile CRUD, goal setting (LLM + fallback), meal logging with AI nutrition + feedback, today/week meal views, AI chat with persisted history + summarization, weekly summary, water tracking, dashboard. These are wired front-to-back.

**Gaps / dead code found in the source:**
- **Achievements are never awarded.** `check_achievements_earned()` (utils) and the `ACHIEVEMENTS` config exist, but nothing calls the checker and no code ever instantiates an `Achievement(...)` row. The `/tracking/achievements` and dashboard endpoints will always return an empty badge list.
- **Streak is always 0.** `UserStats.current_streak_days` is read in several places but **never incremented** anywhere — so streaks, the streak-based achievements, and the displayed streak are non-functional.
- **Workouts are vestigial.** There's a `Workout` model and a `WORKOUT_SUGGESTION_PROMPT`, but **no workout route, no `generate_workout` service function, and no code writes a `Workout` row**. `total_workouts` is never updated. The frontend chat has a "Workout" category, but it just feeds the generic chat prompt.
- **Unused LLM features.** `generate_recipe()` and `generate_supplement_advice()` are fully implemented in the service but have **no API route** — unreachable from the app.
- **Logout doesn't truly invalidate access tokens.** Refresh tokens are flagged `is_revoked`, but JWT validity isn't re-checked against the DB on protected requests, so a stolen access token remains valid until expiry.
- **`on_track` is hardcoded `True`** and the dashboard `motivational_quote` is a fixed string.
- **No tests, no CI, no Dockerfile, no migrations** (relies on `create_all()`).
- `requests` is in `requirements.txt` but unused.

**Maturity:** a polished, coherent portfolio/demo app with a real LLM integration and clean blueprint structure, but several advertised "engagement" features (achievements, streaks, workouts) are scaffolded only and several AI capabilities are not exposed.

---

## README vs. code

The README is largely accurate but **oversells gamification features that aren't functional**:

| README claim | Reality in code |
|---|---|
| "🏆 Achievements — Earn badges for consistency and milestones" | No badge is ever created; `check_achievements_earned` is never called. Endpoint always returns empty. **Non-functional.** |
| "📊 Dashboard … streak tracking" | `current_streak_days` is never incremented; always 0. **Non-functional.** |
| AI Nutrition Analysis / Chat / Meal feedback / Goal macros / Weekly summaries / Water tracking | All accurate and implemented. ✅ |
| "Built with Flask … React (Vite)" + endpoint table | Matches the code exactly. ✅ |
| Tech stack lists "Cerebras LLM SDK" | Correct — `cerebras-cloud-sdk`, model `llama-4-scout-17b-16e-instruct`. ✅ |
| Python "3.9+" badge | Plausible; nothing in code requires newer, but unverified — no `python_requires` declared. |

Additional notes the README omits: the app also contains **unused** recipe-generation and supplement-advice LLM functions, an unused `Workout` model + workout prompt, and `requests` is a dead dependency. The README's port (8000 backend, 5173 frontend) and SQLite-by-default claims are correct.

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\.repo-cache\PersonalHealthAssistant. Source of truth: https://github.com/AmeyaBorkar/PersonalHealthAssistant*
