# SARVA

> Sab kuch. Sabke liye. — a pan-India life utility giving residents and visitors agency over their time, family, rights, and voice (the Sarva Quartet: Samay · Parivar · Adhikar · Awaaz). Go services with ~6 direct deps each, PostgreSQL with row-level security on every user-data table, Flutter mobile, MQTT + WebSockets + Kafka, mTLS internally. Currently in foundation phase.

**Repository:** [`AmeyaBorkar/SARVA`](https://github.com/AmeyaBorkar/SARVA)  
**Category:** Platform / Civic Infrastructure  
**Visibility:** Private  
**Primary language:** Go  
**Default branch:** `main`  
**License:** Other  
**Created:** 2026-05-09  
**Last pushed:** 2026-05-09  
**Metadata updated:** 2026-05-09  
**Size (GitHub reported):** 539 KB  

---

## What it is (one-paragraph version)

Sab kuch. Sabke liye. — a pan-India life utility giving residents and visitors agency over their time, family, rights, and voice (the Sarva Quartet: Samay · Parivar · Adhikar · Awaaz). Go services with ~6 direct deps each, PostgreSQL with row-level security on every user-data table, Flutter mobile, MQTT + WebSockets + Kafka, mTLS internally. Currently in foundation phase.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Go | 151,701 | 39.9% |
| Dart | 89,797 | 23.6% |
| HCL | 51,605 | 13.6% |
| PowerShell | 32,238 | 8.5% |
| Makefile | 26,626 | 7.0% |
| Shell | 13,848 | 3.6% |
| PLpgSQL | 12,402 | 3.3% |
| Dockerfile | 1,011 | 0.3% |
| Swift | 676 | 0.2% |
| Kotlin | 124 | 0.0% |
| Objective-C | 38 | 0.0% |

## File tree

- Total entries indexed: **393** (280 files, 113 directories)

```
.editorconfig  (1 KB)
.gitattributes  (794 B)
.gitignore  (2 KB)
.golangci.yml  (5 KB)
CODEOWNERS  (2 KB)
CONTRIBUTING.md  (6 KB)
LICENSE  (1 KB)
Makefile  (19 KB)
README.md  (10 KB)
SARVA.md  (82 KB)
go.work  (679 B)
go.work.sum  (15 KB)
make.ps1  (18 KB)
sqlc.yaml  (2 KB)
.githooks/    [7 files]
  .githooks/README.md
  .githooks/commit-msg
  .githooks/commit-msg.ps1
  .githooks/install.ps1
  .githooks/install.sh
  .githooks/pre-commit
  .githooks/pre-commit.ps1
.github/    [7 files]
  .github/ISSUE_TEMPLATE/bug.md
  .github/ISSUE_TEMPLATE/feature.md
  .github/PULL_REQUEST_TEMPLATE.md
  .github/dependabot.yml
  .github/workflows/dependabot.yml
  .github/workflows/pre-merge.yml
  .github/workflows/security.yml
docs/    [54 files]
  docs/adr/0001-monorepo-structure.md
  docs/adr/0002-cross-platform-tooling.md
  docs/adr/0003-rds-vs-self-managed-postgres.md
  docs/adr/0004-ecs-fargate-vs-eks-mvp.md
  docs/adr/0005-sqlc-no-orm.md
  docs/adr/0006-jwt-rs256-vs-hs256.md
  docs/adr/0007-rls-pattern.md
  docs/adr/0009-flutter-state-management.md
  docs/adr/0010-mobile-locked-deps.md
  docs/adr/README.md
  docs/adr/_template.md
  docs/decisions/01-mvp-plan-approval.md
  docs/decisions/02-git-setup-and-no-co-author-rule.md
  docs/md/00-index.md
  docs/md/01-brand-identity.md
  ... and 39 more under docs/
infra/    [37 files]
  infra/README.md
  infra/dev/.env.example
  infra/dev/check.ps1
  infra/dev/check.sh
  infra/dev/meilisearch/.gitkeep
  infra/dev/mosquitto/mosquitto.conf
  infra/dev/mosquitto/passwords.example
  infra/dev/pgadmin/servers.json
  infra/dev/postgres/init/01-extensions.sql
  infra/dev/postgres/init/02-databases.sql
  infra/dev/postgres/init/03-roles.sql
  infra/dev/postgres/postgresql.conf.sample
  infra/dev/redis/redis.conf
  infra/docker-compose.yml
  infra/terraform/.gitignore
  ... and 22 more under infra/
mobile/    [101 files]
  mobile/.gitignore
  mobile/.metadata
  mobile/Makefile
  mobile/README.md
  mobile/analysis_options.yaml
  mobile/android/.gitignore
  mobile/android/app/build.gradle.kts
  mobile/android/app/proguard-rules.pro
  mobile/android/app/src/debug/AndroidManifest.xml
  mobile/android/app/src/main/AndroidManifest.xml
  mobile/android/app/src/main/kotlin/app/sarva/sarva_mobile/MainActivity.kt
  mobile/android/app/src/main/res/drawable-v21/launch_background.xml
  mobile/android/app/src/main/res/drawable/launch_background.xml
  mobile/android/app/src/main/res/mipmap-hdpi/ic_launcher.png
  mobile/android/app/src/main/res/mipmap-mdpi/ic_launcher.png
  ... and 86 more under mobile/
services/    [59 files]
  services/README.md
  services/core-api/.dockerignore
  services/core-api/.env.example
  services/core-api/.golangci.yml
  services/core-api/Dockerfile
  services/core-api/Makefile
  services/core-api/README.md
  services/core-api/api/openapi.yaml
  services/core-api/cmd/server/main.go
  services/core-api/db/embed.go
  services/core-api/db/migrations/0001_init.down.sql
  services/core-api/db/migrations/0001_init.up.sql
  services/core-api/db/migrations/0002_extensions_check.down.sql
  services/core-api/db/migrations/0002_extensions_check.up.sql
  services/core-api/db/migrations/0003_revoked_tokens.down.sql
  ... and 44 more under services/
tools/    [1 files]
  tools/README.md
```

## README (verbatim)

# Sarva — comprehensive India life utility

> **Sab kuch. Sabke liye.** (Everything. For everyone.)

A single trusted India-rooted utility that gives every person — resident
and visitor — agency over their **time, family, rights, and voice** (the
Sarva Quartet: Samay · Parivar · Adhikar · Awaaz).

---

## Status

**Foundation phase — engineering kickoff.** The v1 dev-mode build is in
progress per
[`docs/status/cto-reports/02-revised-plan.md`](docs/status/cto-reports/02-revised-plan.md).

- v1 is **internal demo + iteration only** — no public launch.
- v1 runs in **dev mode**: real architecture, sandboxed/mocked partner
  integrations, no real money flows, no real PII at scale, working name
  "Sarva" (no legal entity yet, no domain registered).
- "Build v1 as if real users were on it day one — because they will be,
  soon" — Decision 01.

The full master spec lives at [`SARVA.md`](SARVA.md) and is split for
navigation under [`docs/md/`](docs/md/). Decisions that bind engineering
are recorded in [`docs/decisions/`](docs/decisions/) and
[`docs/status/cto-reports/`](docs/status/cto-reports/).

---

## Architecture overview

The full tech stack is locked in
[`docs/md/14-tech-stack.md`](docs/md/14-tech-stack.md). The headline:

- **Backend:** Go 1.22+ — trust-minimized (~6 direct deps per service),
  `chi` router, `pgx` driver, `sqlc` for compile-time-safe SQL (no
  runtime ORM, ever), `golang-migrate`, `slog`, `golang-jwt/v5`.
- **AI / ML:** Python 3.12 + FastAPI, isolated, anonymised payloads only.
- **Mobile:** Flutter (Dart), Android-first.
- **Database:** PostgreSQL 16 + PostGIS + TimescaleDB + pgvector, with
  Row-Level Security (RLS) enforced on every user-data table.
- **Real-time:** MQTT (mobile uplink) + WebSockets (live UI) + Kafka
  (durable event log).
- **Maps:** OSM + MapLibre + Valhalla — no Google Maps API.
- **Languages:** Bhashini-first for Indian language NLP/ASR/TTS.
- **Cloud:** AWS Mumbai (`ap-south-1`); DR in Hyderabad (`ap-south-2`).
- **Defense in depth:** mTLS internally; gosec, govulncheck, gitleaks,
  Trivy, Dependabot in CI on every commit; append-only audit log on every
  user-data write.

The four MVP services and their owners are listed in
[`CODEOWNERS`](CODEOWNERS).

---

## Repo layout

```
TravelIndia/                       # working name; renames to "sarva" once entity registered
├── SARVA.md                       # master spec (locked)
├── README.md                      # you are here
├── LICENSE                        # proprietary, all rights reserved
├── CONTRIBUTING.md                # how Claude conversations work in this repo
├── CODEOWNERS                     # path → reviewer mapping
├── Makefile                       # ci/lint/test/security/fmt targets
├── make.ps1                       # PowerShell wrapper for Windows-native users
├── go.work                        # Go workspace — populated by T3 onwards
├── .golangci.yml                  # canonical lint config, shared across Go services
├── sqlc.yaml                      # canonical sqlc config root
├── .editorconfig
├── .gitignore
├── .gitattributes
│
├── services/                      # Go services (Core API, Geo, Realtime) + Python AI service
├── mobile/                        # Flutter Android app (T4 onwards)
├── infra/                         # Terraform — AWS Mumbai dev env (T2 onwards)
├── tools/                         # internal scripts and dev helpers
│
├── .github/                       # GitHub Actions, Dependabot, PR + issue templates
├── .githooks/                     # gitleaks + gofmt pre-commit; Conventional Commits commit-msg
│
└── docs/
    ├── md/                        # split spec (16 files; do not modify — derived from SARVA.md)
    ├── adr/                       # Architectural Decision Records
    ├── decisions/                 # Head's strategic decisions
    ├── roles/                     # role charters (CTO, Head, founder)
    ├── status/
    │   ├── cto-reports/           # CTO → Head reports
    │   └── sub-agent-reports/     # sub-agent conversation reports → CTO
    ├── sub-agent-prompts/         # CTO-authored prompts for engineering conversations
    ├── pitch/                     # Head's pitch material
    ├── data/                      # schema docs (later)
    └── security/                  # threat model + security notes (T6 onwards)
```

---

## Quickstart

### One-time setup

```bash
# 1. Install hooks (sets git config core.hooksPath .githooks for this clone)
bash .githooks/install.sh                 # Linux / macOS / WSL / Git Bash
# or
powershell -ExecutionPolicy Bypass -File .githooks/install.ps1   # Windows
```

### Daily commands

```bash
make help          # list every target
make fmt           # gofmt + goimports + dart format
make lint          # golangci-lint + flutter analyze (when those services exist)
make test          # go test ./... per service + flutter test
make security      # gosec + govulncheck + gitleaks
make ci            # the gate — fmt-check + lint + test + security
make hooks-install # (re-)install git hooks
make clean         # remove build artifacts
```

On Windows without GNU Make installed, use the PowerShell wrapper:

```powershell
.\make.ps1 help
.\make.ps1 ci
```

The wrapper accepts the same targets as `make`.

In Sprint 1, several targets are no-ops because no Go or Flutter code
exists yet — the chassis is intentionally empty. Targets degrade
gracefully and exit 0 when their inputs are absent.

---

## Local development

The full data plane runs in Docker via `infra/docker-compose.yml`:
PostgreSQL 16 (with PostGIS + TimescaleDB + pgvector), Redis 7,
Meilisearch, Eclipse Mosquitto (MQTT), and pgAdmin. The Go services
(T3+) and the Flutter app (T4+) connect to it on `localhost`.

Prerequisite: Docker Desktop for Windows / macOS, or `docker` + `docker
compose` on Linux. Verify with `docker info`.

```bash
# Bring the stack up (waits for healthchecks; auto-creates infra/.env)
make dev-up               # or .\make.ps1 dev-up

# Verify every service is reachable
make dev-check            # or .\make.ps1 dev-check

# Tail logs / inspect status
make dev-logs             # or .\make.ps1 dev-logs
make dev-ps               # or .\make.ps1 dev-ps

# Tear down (preserves volumes)
make dev-down             # or .\make.ps1 dev-down

# Tear down + wipe volumes (pgdata, meili-data, ...)
make dev-reset            # or .\make.ps1 dev-reset
```

Connection strings (dev defaults; rotate in prod via Secrets Manager):

| Service     | URL / DSN                                                                 |
| ----------- | ------------------------------------------------------------------------- |
| Postgres    | `postgres://core_api:dev_core_api_change_me@localhost:5432/sarva_core?sslmode=disable` |
| Redis       | `redis://localhost:6379/0`                                                |
| Meilisearch | `http://localhost:7700` (master key `dev_master_key_change_me`)           |
| MQTT (TCP)  | `tcp://localhost:1883`                                                    |
| MQTT (WS)   | `ws://localhost:9001`                                                     |
| pgAdmin     | `http://localhost:5050` (admin@sarva.dev / dev_pgadmin_change_me)         |

The AWS Terraform skeleton is at `infra/terraform/`. It is committed
but **not applied** in dev mode (per
[`docs/decisions/01-mvp-plan-approval.md`](docs/decisions/01-mvp-plan-approval.md)).
Validate with:

```bash
make terraform-validate   # or .\make.ps1 terraform-validate
```

See [`infra/terraform/README.md`](infra/terraform/README.md) for the
apply procedure once an AWS account exists.

## Mobile dev (T4 onwards)

The Flutter Android app lives at [`mobile/`](mobile/) and has its own
Makefile for day-to-day work. The repo-root `make lint` and `make test`
targets auto-detect `mobile/pubspec.yaml` and call `flutter analyze` /
`flutter test` for free, so the mobile-specific Makefile is for actions
the root doesn't need to know about.

```bash
make -C mobile pub-get        # fetch deps
make -C mobile gen-codegen    # build_runner: freezed + json_serializable
make -C mobile gen-l10n       # regenerate AppLocalizations from ARB
make -C mobile analyze        # very_good_analysis lints
make -C mobile test           # unit + widget tests with coverage
make -C mobile run            # launch on default Android device
make -C mobile build-apk      # release APK (arm64)
```

Prerequisites: Flutter 3.22+, Android SDK with build-tools 34, an
Android device or running emulator. Locales English / Hindi / Marathi
are baked into the auth flow today; the full Phase-1 set of seven
languages lands in Sprint 4 per
[`docs/md/15-multilingual.md`](docs/md/15-multilingual.md).

The state-management primitive (Riverpod) is recorded in
[`docs/adr/0009-flutter-state-management.md`](docs/adr/0009-flutter-state-management.md).
The locked dependency list is recorded in
[`docs/adr/0010-mobile-locked-deps.md`](docs/adr/0010-mobile-locked-deps.md).
**Adding a new direct mobile dependency requires a successor ADR.**

---

## How this repo gets built

Engineering is executed by **Claude Code conversations** orchestrated by
a CTO conversation. Read [`CONTRIBUTING.md`](CONTRIBUTING.md) and then
[`docs/sub-agent-prompts/README.md`](docs/sub-agent-prompts/README.md)
for the full protocol. The short version:

```
Founder  →  Head  →  CTO  →  sub-agent conversations  →  CTO review  →  merge
```

Decisions, prompts, and reports are all files in this repo. Nothing
important happens in chat that isn't also captured on disk.

---

## License

Proprietary — all rights reserved. See [`LICENSE`](LICENSE). Specific
algorithms with civic impact (fair-fare pricing, safety scoring,
eligibility matching) will be released separately under Apache-2.0
before public launch, in keeping with the project's Contract.

---

## The Contract

Sarva makes ten binding promises to its users about what it will **never**
do — never sell user data, never paywall life-critical features, never
manipulate civic discourse, never run interruptive ads, etc. They are
listed in
[`docs/md/03-mission-values-contract.md`](docs/md/03-mission-values-contract.md).

These rules survive growth, investor, government, and monetization
pressure. Any contribution that risks violating them is **escalated**, not
worked around.

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/SARVA*
