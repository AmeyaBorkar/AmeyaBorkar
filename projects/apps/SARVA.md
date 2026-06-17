# SARVA

> SARVA ("sab kuch, sabke liye" — everything, for everyone) is an ambitious, spec-heavy attempt at a pan-India life-utility platform organised around four lenses (Samay/time, Parivar/family, Adhikar/rights, Awaaz/voice). **As it exists in the working tree today it is a single Go microservice (`core-api`) doing phone-OTP authentication + user-profile CRUD, a Flutter auth-flow shell, and a large amount of design docs and local-dev infra.** None of the four pillars are implemented as features yet — they live in the spec. The "microservices", MQTT, pgvector, OSM, mTLS, Kafka, and the Python AI service described in the README are aspirational/infra-only and have **no application code** in this clone.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/SARVA (origin remote; repo dir is named `TravelIndia`, the pre-rename working name) |
| **Visibility** | Private |
| **Category** | apps |
| **Primary language(s)** | Go (backend), Dart/Flutter (mobile), SQL (schema) |
| **Local path** | `C:\Users\ameya\Documents\TravelIndia` |
| **Default branch** | `main` |
| **Lines of code (computed)** | Go: ~2,883 (non-test, non-generated) + 1,955 test + 747 sqlc-generated. Dart: ~2,798 (hand-written) + 1,562 generated (freezed/json/l10n). SQL: 608 (migrations + sqlc queries + dev init). |
| **Source files (computed)** | 23 Go (non-test, non-generated), 15 Go test files, 0 `*.pb.go`, 35 Dart (non-generated), 13 `.sql` |
| **Services (count + names)** | **1 implemented:** `core-api` (Go). README/CODEOWNERS reference 4 more by path only — `geo`, `realtime`, `ai`, plus mobile — but `services/` contains **only** `core-api` (+ a README). No `services/geo`, `services/realtime`, `services/ai` directories exist. |
| **Key dependencies** | Go: `go-chi/chi/v5`, `jackc/pgx/v5`, `golang-jwt/jwt/v5`, `golang-migrate/migrate/v4`, `caarlos0/env/v11`, `go-playground/validator/v10` (**exactly 6 direct runtime deps**). Dart: `flutter_riverpod`, `go_router`, `dio`, `flutter_secure_storage`, `freezed`, `json_serializable`. |
| **License** | Proprietary — all rights reserved (see `LICENSE`) |
| **Last commit** | `b89ab33` — *docs(sprint-01): F-011 prompt — root golangci-lint v1 to v2 migration* (2026-05-10) |

---

## What it actually is

SARVA is, in the repository as cloned, **a foundation-phase monorepo** whose largest concrete asset is one Go HTTP service plus a Flutter app skeleton. The README is explicit about this ("Foundation phase — engineering kickoff", v1 is "internal demo + iteration only"). The vast majority of the repo is **documentation and orchestration scaffolding**: a 84 KB master spec (`SARVA.md`), a 16-file split spec under `docs/md/`, ADRs, decision records, role charters, and "sub-agent prompt" files for a Claude-driven, file-based engineering workflow.

The one piece of working product is `services/core-api`, a `chi`-based Go service implementing:

- **Phone-number + OTP authentication** (E.164 phone, 6-digit code, argon2id-hashed, rate-limited via Postgres row counts).
- **JWT issuance** (RS256, access+refresh, refresh rotation + revocation list).
- **Self-service user profile** (`GET/PATCH/DELETE /v1/me`).
- **DPDP-style consent capture** on signup and an **append-only audit log** on every sensitive action.
- **Postgres Row-Level Security** on every user-data table, driven by per-request session GUCs.

The Flutter app (`mobile/`) implements the **client side of that exact auth flow** (phone entry → OTP entry → home/profile) and nothing else — `home_screen.dart` is a self-described placeholder.

The mobile app's package is `sarva_mobile`; the Go module is `github.com/sarva/core-api`. The repo directory name `TravelIndia` is the pre-incorporation working name (README: "renames to 'sarva' once entity registered").

---

## The four pillars in code — Samay / Parivar / Adhikar / Awaaz

The "four pillars" terminology in the prompt corresponds to the **Quartet** in the spec (`docs/md/04-quartet.md`): four *lenses*, not four services. The spec separately lists **30 numbered "pillars"** (feature groups) in `docs/md/05-pillars.md`. **None of the Quartet lenses or the 30 feature-pillars have implementing code.** The only implemented vertical is identity/auth, which the spec treats as cross-cutting plumbing rather than a pillar.

| Quartet lens | Devanagari | Spec meaning | Service / module that implements it | Status in code |
|---|---|---|---|---|
| **Samay** (time) | समय | "Will I waste hours waiting?" — queue/wait predictions, transit ETAs | *(planned: Geo + Realtime services)* | **Not implemented.** No code. `mosquitto.conf` comments reference a future "Wait Map" but no producer/consumer exists. |
| **Parivar** (family) | परिवार | "Is my family okay?" — family check-ins, care coordination | *(planned: Realtime service)* | **Not implemented.** Mentioned only in MQTT config comments ("Family check-ins"). |
| **Adhikar** (rights) | अधिकार | "Am I being cheated? What am I owed?" — schemes, fair-fare, eligibility | *(planned: future services)* | **Not implemented.** The closest real artifact is DPDP **consent records** (`user_consent`) — a user-rights *mechanism*, not the Adhikar feature set. |
| **Awaaz** (voice) | आवाज़ | "Will my problem be heard?" — Reddit-like civic platform | *(planned: Realtime + AI services)* | **Not implemented.** No posts/votes/feed code. |

**Bottom line:** the four pillars exist entirely in `docs/`. The shipping code is the auth/identity substrate that every pillar would eventually sit on.

---

## Architecture & how it's structured

Implemented vs. planned, by where the actual code lives:

```
TravelIndia/  (working name; module path is github.com/sarva/core-api)
├── services/
│   ├── README.md
│   └── core-api/                  ← the ONLY implemented service
│       ├── cmd/server/main.go     ← entrypoint: config→log→migrate→pool→JWT→chi router
│       ├── api/openapi.yaml        ← OpenAPI 3 spec (8 paths)
│       ├── db/
│       │   ├── embed.go            ← go:embed of migrations
│       │   └── migrations/         ← 0001_init, 0002_extensions_check, 0003_revoked_tokens (up+down)
│       ├── internal/
│       │   ├── auth/               ← jwt.go, middleware.go, otp.go, sms.go, handlers.go, errors
│       │   ├── users/              ← /v1/me handlers
│       │   ├── audit/              ← append-only audit writer
│       │   ├── config/             ← env→Config (caarlos0/env)
│       │   ├── db/                 ← pgx pool + RLS GUC tx plumbing
│       │   │   ├── queries/        ← *.sql (sqlc input): users, otp, tokens, audit
│       │   │   └── generated/      ← sqlc OUTPUT (generated, not hand-edited)
│       │   ├── httpx/              ← request-id, recoverer, access log, JSON/error helpers
│       │   ├── log/                ← slog setup
│       │   └── testdb/             ← testcontainers + ephemeral JWT keys for tests
│       ├── .secrets/               ← dev RSA JWT keypair (jwt_private.pem / jwt_public.pem)
│       ├── go.mod / go.sum         ← 6 direct runtime deps
│       └── Dockerfile, Makefile, sqlc.yaml
│
├── mobile/                         ← Flutter Android-first app (sarva_mobile)
│   └── lib/
│       ├── main.dart, app/         ← router (go_router), theme, MaterialApp
│       ├── core/api/               ← Dio client + auth/request-id/error interceptors
│       ├── core/storage/           ← flutter_secure_storage wrapper
│       ├── features/auth/          ← data (api/repo) + domain (freezed AuthState) + presentation (phone/otp screens)
│       ├── features/home/          ← placeholder home screen
│       ├── features/me/            ← profile screen
│       └── l10n/                   ← ARB + generated localizations: en / hi / mr
│
├── infra/
│   ├── docker-compose.yml          ← dev data plane: Postgres(+PostGIS/Timescale/pgvector), Redis, Meilisearch, Mosquitto, pgAdmin
│   ├── dev/                        ← per-service configs + postgres init/ (extensions, roles, databases)
│   └── terraform/                  ← AWS skeleton (modules: compute/database/edge/network/secrets/storage) — committed, NOT applied
│
├── docs/                           ← the bulk of the repo: spec (md/), adr/, decisions/, roles/, status/, sub-agent-prompts/
├── tools/                          ← dev helpers
├── go.work                         ← workspace: `use ./services/core-api` only
├── Makefile / make.ps1             ← root build/lint/test/security + dev-up/down targets
└── README.md, SARVA.md, CODEOWNERS, CONTRIBUTING.md, LICENSE
```

Request flow for the implemented service:

```
Flutter app (Dio + interceptors)
        │  HTTP/JSON, Bearer access token
        ▼
core-api  (chi router)
  middleware:  RequestID → Recoverer → AccessLog
  ┌──────────────────────────────────────────────┐
  │ /v1/auth/*  (otp issue/verify, refresh)       │  ← service-role tx
  │ /v1/auth/logout, /v1/me/*  (Authenticated)    │  ← user-role tx
  └──────────────────────────────────────────────┘
        │  every DB op runs inside a pgx tx that first does
        │  SELECT set_config('app.user_id',…) / ('app.role',…)
        ▼
PostgreSQL  — Row-Level Security policies key off app.user_id / app.role
              tables: users, otp_request, user_consent, audit_log, revoked_token
```

There is **no API gateway, no service mesh, no mTLS in code** — those are described in the README/spec but absent. Redis, Meilisearch, and Mosquitto run in docker-compose but `core-api` connects to **none** of them (it requires `REDIS_URL` in config but never opens a Redis client).

---

## Backend services — service by service

### core-api (the only service)

**Responsibility:** identity (phone OTP → JWT), user self-service, consent capture, audit logging. Module `github.com/sarva/core-api`, Go 1.25 (toolchain go1.26.3), `chi` router, `pgx` pool, `sqlc`-generated queries, `golang-migrate` for schema.

**Endpoints** (from `cmd/server/main.go` + `api/openapi.yaml`):

| Method & path | Auth | Behaviour |
|---|---|---|
| `GET /healthz` | none | liveness `{"status":"ok"}` |
| `GET /readyz` | none | pings DB; 503 if unreachable |
| `POST /v1/auth/otp/issue` | none | validates E.164, rate-limits (≤3/phone/hr, ≤10/IP/min via Postgres counts), generates 6-digit code (crypto/rand), argon2id-hashes, inserts `otp_request`, sends via SMS provider, audits. Returns **202** `{request_id, expires_at}`. |
| `POST /v1/auth/otp/verify` | none | checks consumed/expired/attempts, constant-time argon2id compare; on match find-or-create user, on new user write `tos`+`privacy` consent rows; issues access+refresh JWTs. Returns **200** `{tokens, user, is_new}`. |
| `POST /v1/auth/refresh` | none (refresh token in body) | verifies refresh JWT, checks `revoked_token`, **rotates** (revokes old jti, reason `rotation`), issues new pair. |
| `POST /v1/auth/logout` | Bearer access | revokes supplied refresh jti (reason `logout`); idempotent 204. |
| `GET /v1/me` | Bearer access | returns caller's user row (RLS-scoped) + audit. |
| `PATCH /v1/me` | Bearer access | updates `display_name` / `locale` (validated). |
| `DELETE /v1/me` | Bearer access | soft-delete (sets status/`deleted_at`), returns 202. |
| `GET /debug/last-otp?phone=` | none | **dev-only**, gated by `OTP_DEV_DEBUG_ENDPOINT` + non-prod; returns the stub-sent code. |

**Deps:** exactly **6 direct runtime** modules in `go.mod` (`chi`, `pgx`, `golang-jwt`, `golang-migrate`, `caarlos0/env`, `validator`). Test-only directs: `testify`, `google/uuid`, `golang.org/x/crypto` (argon2 lives here and is used in non-test OTP code too). The module comment states adding a 7th direct dep "requires an ADR and CTO sign-off."

**SMS providers** (`internal/auth/sms.go`): `SMSStub` (dev — logs the code + caches last-per-phone) and `SMSMSG91` (skeleton for msg91.com whose `Send` returns `"not implemented in v1"`). Provider chosen via `SMS_PROVIDER` env (`stub` | `msg91`).

### Planned-but-absent services

`services/geo`, `services/realtime`, `services/ai` are referenced in `CODEOWNERS`, `go.work` comments, and README but have **no directories or code**. `go.work` only `use`s `./services/core-api`.

---

## Data layer — Postgres schema, RLS, pgvector, migrations

**Migrations** (`db/migrations/`, golang-migrate, embedded via `go:embed`):

- **0001_init** — core schema + RLS + grants.
- **0002_extensions_check** — idempotent `CREATE EXTENSION IF NOT EXISTS` for `pgcrypto`, `pg_trgm`, `unaccent`, **`postgis`**, **`vector`** (pgvector). Comments admit postgis/vector are "core-api-adjacent (geo / vector services own them long-term)" — installed defensively but **no table in the schema uses a `vector` or `geometry` column**.
- **0003_revoked_tokens** — `revoked_token` table for JWT revocation.

**Tables** (all with `ENABLE ROW LEVEL SECURITY`):

| Table | Purpose | Notable columns |
|---|---|---|
| `users` | identity; phone is primary key of identity | `phone_e164` (CHECK E.164 regex, UNIQUE), `locale` (CHECK ∈ 13 Indian-language codes), `status` (active/suspended/deleted), `consent_id`, soft-delete `deleted_at` |
| `otp_request` | OTP issuance/verification audit | `code_hash`, `attempts`/`max_attempts`, `expires_at`, `consumed_at`, `consumed_by_user`, `ip`, `request_id` |
| `user_consent` | DPDP consent records | `consent_type`, `granted`, `version`, `granted_at`, `withdrawn_at` |
| `audit_log` | append-only audit | `bigserial` PK, `actor_role`, `action`, `result` (CHECK success/failure/blocked), `payload_summary jsonb` |
| `revoked_token` | refresh-token revocation | `jti` PK, `reason` (CHECK logout/rotation/admin/suspicion), `expires_at` |

**Row-Level Security** — the security centrepiece. Two helper SQL functions normalize session GUCs: `app_current_user_id()` (`NULLIF(current_setting('app.user_id',true),'')::uuid`) and `app_current_role()`. Policies:

- `users`: self SELECT/UPDATE where `id = app_current_user_id()`; INSERT/DELETE only for `app.role = 'service'`.
- `otp_request`: `FOR ALL` **service-only**.
- `user_consent`: self-SELECT; service-only INSERT/UPDATE.
- `audit_log`: INSERT allowed for service **or** a user inserting their own `actor_user_id`; self-SELECT; explicit `USING(false)` policies blocking **UPDATE and DELETE for everyone** (append-only invariant, belt-and-braces with missing GRANTs).
- `revoked_token`: `FOR ALL` service-only.

The per-service role `core_api` is granted least-privilege: SELECT/INSERT/UPDATE/DELETE on user tables, but only SELECT+INSERT on `audit_log`.

**How RLS is enforced at runtime** (`internal/db/db.go`): each request handler calls `db.WithUser(ctx, userID, role)` which `BeginTx` then `SELECT set_config('app.user_id',$1,true)` / `('app.role',$2,true)` (transaction-local). All sqlc queries run on that tx via `db.Queries(ctx)`. `SET LOCAL`/`set_config(...,true)` is used deliberately so GUCs vanish on commit and never leak across pooled connection reuse. Auth/OTP write paths use `role='service'` (no user exists yet at signup); `/v1/me` paths use `role='user'` so RLS scopes rows to the caller — application code does **no** `WHERE id = $current_user` filtering, leaning entirely on the policy.

**sqlc**: queries in `internal/db/queries/{users,otp,tokens,audit}.sql` compile to `internal/db/generated/*` (747 LOC, **generated — do not edit**). Config in root `sqlc.yaml` + service `sqlc.yaml`.

---

## Messaging & realtime — MQTT

**MQTT is infra-only; there is zero MQTT client code.** `infra/docker-compose.yml` runs `eclipse-mosquitto:2.0.20` with `infra/dev/mosquitto/mosquitto.conf` exposing TCP 1883 + WebSockets 9001, `allow_anonymous true` (dev). Config comments describe intended use ("Wait Map crowdsource pings, Family check-ins, Awaaz live feed") and future production mTLS with per-device certs from "the Realtime service." No Go or Dart file references MQTT/mosquitto, defines topics, or connects. WebSockets and Kafka (README "durable event log") likewise have **no code**.

---

## Security — mTLS, auth, secrets

**Authentication (real):**
- **OTP**: 6-digit, `crypto/rand` only (explicitly not `math/rand`), **argon2id**-hashed into a self-contained PHC string (`$argon2id$v=19$m=16384,t=1,p=2$salt$hash`); deliberately light work factor (t=1, 16 MB, p=2) since security comes from rate-limit + 5-min TTL + 3-attempt lockout, with constant-time compare (`subtle.ConstantTimeCompare`).
- **JWT**: **RS256, 2048-bit RSA** (ADR-0006). Typed `Claims` add `role` (user/service) and `typ` (access/refresh); `Verify` asserts alg, issuer, audience, exp/nbf, and `typ` (an access token cannot pass refresh verification). Access TTL 15m (not revocable, expires naturally); refresh TTL 720h with **rotation + DB-backed revocation list** (chosen over Redis to stay within the 6-dep budget — ADR-0007). Keypair sanity-checked at boot (`priv.PublicKey.Equal(pub)`).
- **Middleware**: `Authenticated` extracts `Bearer`, verifies access token, stashes `Identity{UserID, Role}` on context. Access tokens are **not** revocation-checked per request (only refresh tokens are, in the refresh handler).

**Secrets:** dev RSA keypair committed at `services/core-api/.secrets/{jwt_private,jwt_public}.pem` (dev only; key paths come from env). Prod design (per config comments) injects via AWS Secrets Manager → ECS env. CI runs gitleaks, gosec, govulncheck, Trivy (`.github/workflows/security.yml`, `.githooks/`).

**mTLS:** the README/spec/mosquitto comments claim internal mTLS, but **no TLS/x509 code exists in the service** (the only `x509`/`tls` reference is in `internal/testdb/jwt_keys.go`, which generates test RSA keys, not transport TLS). With one service there is no internal service-to-service traffic to secure yet.

**DPDP / privacy:** consent rows written on signup; append-only audit log on every sensitive action (`audit.Action*` constants: `auth.otp.issue/verify`, `auth.token.refresh`, `auth.logout`, `user.read/update/delete`, `consent.grant/withdraw`); minimal PII (phone is the only required identity field).

---

## Geo — OpenStreetMap usage

**None in code.** README lists "OSM + MapLibre + Valhalla — no Google Maps API" and the spec's NAVIGATE pillar leans on OpenStreetMap, but a search across Go and Dart finds **no** references to OSM, MapLibre, Valhalla, or map tiles. `postgis` is installed by migration 0002 but unused by the schema. Geo is entirely a future-service concern (`services/geo` doesn't exist).

---

## Flutter app — structure, screens, state, backend wiring

**Package** `sarva_mobile`, Flutter ≥3.22 / Dart ^3.4 (founder on Flutter 3.38.9). Android-first.

**Structure** (feature-first): `app/` (router, theme), `core/` (api client, storage, `Result` type), `features/{auth,home,me}/` each split into `data` / `domain` / `presentation`, `l10n/` (en/hi/mr ARB + generated).

**Screens / routing** (`go_router`): `/` root gate → `/auth/phone` → `/auth/otp` → `/home` ↔ `/me`. A single `redirect` consults `AuthState` and forces the user to the screen matching their lifecycle stage (`AuthUnknown` / `AuthSignedOut` / `AuthAwaitingOtp` / `AuthSignedIn`). `home_screen.dart` is an explicit placeholder ("T5 replaces it with the real Pune-seeded list of POIs").

**State management**: **Riverpod** (ADR-0009). `AuthRepository extends Notifier<AuthState>` is the boundary between UI, the `AuthApi` HTTP layer, and `SecureStorage`. `AuthState` is a `freezed` sealed union.

**Backend wiring**: `dio` client (`core/api/api_client.dart`) with base URL `http://10.0.2.2:8080` (Android-emulator loopback; override via `--dart-define=SARVA_API_BASE_URL`). Three interceptors: `RequestIdInterceptor`, `AuthInterceptor` (attaches Bearer, auto-refreshes on 401, forces sign-out on refresh failure), and a debug `LogInterceptor`. Refresh tokens persisted in `flutter_secure_storage`; access tokens held in memory in a shared `AuthTokens` holder. The app talks **only** to `core-api`'s auth + `/v1/me` endpoints.

**Tests**: 10 Dart test files under `mobile/test` + `mobile/integration_test`; `mocktail` + `very_good_analysis`.

---

## Tech stack & dependencies

- **Backend:** Go 1.25 (toolchain 1.26.3), `chi/v5`, `pgx/v5`, `golang-jwt/v5`, `golang-migrate/v4`, `caarlos0/env/v11`, `validator/v10`; `sqlc` (compile-time SQL, no ORM); `slog`; argon2id via `golang.org/x/crypto`. **6 direct runtime deps, by policy.**
- **Mobile:** Flutter/Dart — `flutter_riverpod`, `go_router`, `dio`, `flutter_secure_storage`, `shared_preferences`, `freezed`/`json_serializable`, `intl`. Locked dep list (ADR-0010); adding one needs a successor ADR.
- **Data plane (docker-compose, dev):** PostgreSQL 16 via `timescale/timescaledb-ha:pg16.6-ts2.17.2-oss` (bundles PostGIS + pgvector + TimescaleDB), Redis 7.4, Meilisearch 1.11, Eclipse Mosquitto 2.0, pgAdmin 8. **Only Postgres is actually consumed by code.**
- **Infra-as-code:** Terraform AWS skeleton (`infra/terraform/`, modules compute/database/edge/network/secrets/storage) — committed, **not applied**.
- **CI:** GitHub Actions — `pre-merge.yml`, `security.yml`, `dependabot.yml`. Root `Makefile` + `make.ps1` (Windows) with `fmt/lint/test/security/ci` and `dev-up/down/reset/check`.

---

## Build / run / deploy

**Data plane:**
```bash
make dev-up        # docker compose up (waits for healthchecks; auto-creates infra/.env)
make dev-check     # validate every service reachable
make dev-down      # stop (keep volumes) ; make dev-reset wipes volumes
```

**core-api (from `services/core-api/`):** requires env `DATABASE_URL`, `REDIS_URL`, `JWT_PRIVATE_KEY_PATH`, `JWT_PUBLIC_KEY_PATH` (see `.env.example`). Dev DSN: `postgres://core_api:dev_core_api_change_me@localhost:5432/sarva_core?sslmode=disable`.
```bash
go run ./cmd/server                 # starts HTTP server on :8080
go run ./cmd/server migrate up      # run migrations (needs DATABASE_URL_MIGRATOR)
go run ./cmd/server migrate down 1  # roll back
docker build -t sarva/core-api .    # Dockerfile present
```
The binary also auto-runs migrations on boot if `DATABASE_URL_MIGRATOR` is set.

**Mobile (from `mobile/`):**
```bash
make -C mobile pub-get       # flutter pub get
make -C mobile gen-codegen   # build_runner: freezed + json_serializable
make -C mobile gen-l10n      # AppLocalizations from ARB
make -C mobile run           # launch on Android device/emulator
make -C mobile build-apk     # release arm64 APK
```

**Deploy:** none wired. Terraform is committed but not applied (Decision 01: dev mode, no AWS account). No k8s manifests; the only container artifact is `core-api/Dockerfile`.

---

## Tests

- **Go:** 15 `_test.go` files, **97 `func Test*`**, ~1,955 LOC. Coverage spans auth (jwt, otp, middleware, handlers incl. logout/refresh edge cases), config validation, db + migrate, httpx, log, audit, users. `internal/testdb/` uses **testcontainers** to spin a real Postgres so RLS is exercised the same way as prod (`cov.out` is checked in). This service is genuinely well-tested.
- **Dart:** 10 test files (unit + widget + integration) under `mobile/test` and `mobile/integration_test`; `mocktail` for mocking, `coverage/` directory present.

---

## Status, completeness & notable gaps

**Real and working:**
- `core-api` auth/identity service — production-grade patterns (RS256 JWT, refresh rotation+revocation, argon2id OTP, RLS-per-request-tx, append-only audit, DPDP consent), heavily tested.
- Flutter auth flow (phone → OTP → home/profile) wired end-to-end to that service.
- Dev data-plane docker-compose; CI (lint/test/security); Terraform skeleton.

**Scaffolded / placeholder:**
- `home_screen.dart` (explicit placeholder), `services/README.md`.

**Claimed but absent (no code):**
- **All four Quartet pillars** (Samay/Parivar/Adhikar/Awaaz) and all 30 feature-pillars.
- **`geo`, `realtime`, `ai` services** — directories don't exist; only CODEOWNERS/README paths.
- **MQTT, WebSockets, Kafka** producers/consumers (broker runs; nothing connects).
- **pgvector / embeddings** — extension installed, no vector column, no embedding pipeline, **no LLM/embedding provider** anywhere in code (the only "Bhashini"/AI mentions are in spec/SQL comments).
- **OSM / MapLibre / Valhalla** geo.
- **mTLS** internal transport security.
- **Redis** — required by config but never used (no client); rate limits are Postgres-backed by design.
- **Python/FastAPI AI service** (README "AI/ML").

This is best understood as **one well-built vertical slice (identity) under an extremely large product vision**, with the gap between the two captured honestly in `docs/status/`.

---

## README vs. code

| README / spec claim | Reality in code | Verdict |
|---|---|---|
| "Go backend **microservices**" / "four MVP services" | One service (`core-api`); `geo`/`realtime`/`ai` don't exist | **Overstated** (README does qualify it as foundation-phase) |
| "~6 direct deps per service; trust-minimized" | `core-api` `go.mod` has **exactly 6** direct runtime deps | **Accurate** |
| "mTLS internally" | No TLS/x509 transport code; only test-key gen uses x509 | **Aspirational** |
| "MQTT (mobile uplink) + WebSockets + Kafka" | Mosquitto broker in compose; **no client code**, no WS, no Kafka | **Infra-only / aspirational** |
| "pgvector" + AI/embeddings | `vector` extension installed; **no vector column, no embeddings, no LLM/embedding provider** | **Aspirational** |
| "OSM + MapLibre + Valhalla — no Google Maps" | No geo code at all | **Aspirational** |
| "PostgreSQL + Row-Level Security on every user-data table" | True — RLS enabled + policies on all 5 tables, enforced via per-request session GUCs | **Accurate (and a real strength)** |
| "Python 3.12 + FastAPI AI service" | No Python service in repo | **Aspirational** |
| "Bhashini-first Indian-language NLP" | Mentioned only in comments; UI localized for **en/hi/mr** | **Partially (3 locales hardcoded, no NLP)** |
| "Redis 7 (cache + rate limit)" | `REDIS_URL` required by config but **no Redis client**; rate limits are Postgres-backed (ADR-0007) | **Discrepancy — config demands Redis the code never uses** |
| Status: "Foundation phase… internal demo only" | Matches reality | **Accurate and refreshingly honest** |

The README is unusually candid that most of the stack is not yet built ("several targets are no-ops because no Go or Flutter code exists yet"). The main genuine discrepancy is **Redis being a required config value that no code consumes**.

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\TravelIndia. Source of truth: https://github.com/AmeyaBorkar/SARVA*
