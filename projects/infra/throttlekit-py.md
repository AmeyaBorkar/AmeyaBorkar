# throttlekit-py

> The Python client for **ThrottleKit**, the TS/Node distributed rate-limiting core. It re-implements **no** rate-limiting math: every `Decision` comes from the one Node core, reached through one of two pluggable doors — **`ServiceBackend`** (gRPC to a `throttlekit-server`, full `check`/`check_many`/`peek`/`forecast`/`debit`/`admit` surface) or **`RedisBackend`** (the core's own *vendored* Lua run directly against the same Redis a Node fleet uses, `check` only, bit-identical). Bit-identical behavior vs. the core is *proven*, not asserted: the direct path replays the core's full time-parametrized golden vectors through real Redis and matches the Node oracle field-for-field. Ships sync + async twins of every door, Tier-2 fleet leasing (`FleetBackend`/`LeaseSpender`), a read-only `MonitorBackend`, and FastAPI/Starlette/Django/Flask integrations.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/throttlekit-py |
| **Visibility** | Public |
| **Category** | infra |
| **Primary language(s)** | Python (3.10+); Lua (the vendored Redis scripts); Protobuf (the wire contract) |
| **Local path** | `C:\Users\ameya\Documents\throttlekit-py` |
| **Default branch** | `main` |
| **Lines of code** (computed, tracked, generated `*_pb2*` excluded) | **2,617** Python in `src/throttlekit/` · **2,469** Python in `tests/` · ~5,623 Python total (src + tests + scripts + examples) · **173** lines of Lua (10 scripts) · **345** lines `throttlekit.proto` · **1,404** lines `golden-vectors.json` |
| **Source files** (computed) | **23** Python modules in `src/throttlekit/` (incl. 5 in `contrib/`) · **18** test modules · **3** scripts · **6** example scripts · **10** vendored `.lua` · 1 `.proto` |
| **PyPI package** | Dist name **`throttlekit-py`**, import name **`throttlekit`** (PyPI's `throttlekit` is an unrelated project). **Published** — latest **0.5.0**; all of `0.1.0, 0.2.0, 0.2.1, 0.3.0, 0.4.0, 0.5.0` are live (HTTP 200, verified against pypi.org). `pyproject.version` = `0.5.0`; classifier `Development Status :: 3 - Alpha`. |
| **Key dependencies** | Runtime: `grpcio>=1.60`, `protobuf>=6.31.1,<7`. Optional extras: `redis>=5` (`[redis]`), `fastapi`/`starlette`/`django`/`flask` (one extra each, `[all]` for all four). Build: `hatchling` + `grpcio-tools~=1.80.0` (pinned). Dev: `pytest>=8`, `ruff==0.15.15` (exact-pinned), `mypy>=1.11`, `httpx>=0.27`. |
| **License** | MIT (`Copyright (c) 2026 Ameya Borkar`) |
| **Last commit** | `4ea7029` — *"style: ruff format tests/test_fleet_backend.py"* (2026-06-10) |

---

## What it actually is

A pure-Python **client library** for ThrottleKit. It contains *no rate-limiting algorithm of its own* — that is the entire architectural premise, stated in `__init__.py` and every backend docstring: "exactly one thing computes a `Decision` — the Node core, directly or as Lua-in-Redis." The client's job is marshalling, transport, and decoding.

Two doors reach that one core:

1. **`ServiceBackend`** (`service_backend.py`) — a thin gRPC client speaking `throttlekit.v1.RateLimiter`. The service (running the same Node core) computes every decision; the client decodes the reply into frozen `Decision` / `Forecast` dataclasses. Full surface: `check`, `check_many`, `peek`, `forecast`, `debit` (token-budget cost axis), `admit` (concurrency / unified axis with held-slot lease lifecycle).
2. **`RedisBackend`** (`redis_backend.py`) — runs the core's **vendored Lua** (`src/throttlekit/_scripts/*.lua`) directly against the same Redis a Node fleet uses. Exposes **`check` only**, by design (see below). Decisions are bit-identical to an embedded Node library because it executes the *identical script bytes*.

On top of those, 0.5.0 added: **`FleetBackend`** + `LeaseSpender` + `LeasedLimiter` (Tier-2 client-held leasing over `Fleet.Reserve`), a read-only **`MonitorBackend`**, `asyncio` twins of every door, and `throttlekit.contrib.*` web-framework adapters.

Despite the prompt's framing as a single-purpose client, the working tree is a fairly complete client SDK: **23 source modules** spanning six client classes (× sync/async), five strategy configs, an HTTP-headers helper, a framework-agnostic decorator, and four framework integrations.

## Where it sits in the ThrottleKit family

Part of a three-repo family:

- **`throttlekit`** — the TS/Node core (npm `throttlekit`), the **one oracle**. Owns the algorithms (GALE distributed leasing, TALE token-budget escrow), the Lua scripts, the gRPC `throttlekit-server`, and the language-neutral wire contract (proto + golden vectors) under its `wire/` tree.
- **`throttlekit-py`** *(this repo)* — a *consumer* of that contract. It **vendors** the core's artifacts with sha256 checksums (`scripts/sync_contract.py`) so the two repos cannot silently drift, and reaches the core either over gRPC (the service door) or by running the core's own Lua directly (the Redis door).
- **`throttlekit-site`** — the Astro marketing/docs site (throttlekit.in).

The relationship is explicitly encoded in code: `pyproject.toml` links `"Core (Node)" = "https://www.npmjs.com/package/throttlekit"`; `sync_contract.py` defaults its `--source` to a sibling `../GreenfeildProject` checkout (the core's local dev name) or `$THROTTLEKIT_REPO`; `manifest.json` records `"generatedFrom": "throttlekit@1.0.1"` and the golden vectors `"generatedFrom": "throttlekit@1.4.0"`. The README points to the core repo's `research/polyglot/battle-test` as the cross-repo proof that drives both backends against a real 3-server fleet.

## Architecture & how it's structured (gRPC path vs direct-Redis path)

The two paths share `Decision`/`Forecast` domain objects and the error taxonomy, but nothing else:

- **gRPC path:** `ServiceBackend` → generated stubs (`_generated/throttlekit_pb2*`) → `throttlekit-server` → Node core → store (memory / Redis / Postgres / DynamoDB). The client never touches raw Lua; the proto is the only contract it depends on. Math lives entirely server-side.
- **Direct-Redis path:** `RedisBackend` → `_contract.script()` loads vendored Lua + ARGV order from `manifest.json` → `EVALSHA`/`EVAL` against Redis → Redis runs the Lua. Math lives in the Lua (which is the core's own script, vendored byte-for-byte).

Annotated tree (generated/cache dirs omitted):

```
throttlekit-py/
├── contract/                         # DEV/TEST artifacts (not shipped in the wheel)
│   ├── throttlekit.proto             # 345 lines — the v1 wire contract → gRPC stubs
│   ├── golden-vectors.json           # 1404 lines — 18 suites: rateLimit, tokenBudget, lease
│   └── manifest.sha256               # sha256 of proto + vectors (drift-gate)
├── src/throttlekit/
│   ├── __init__.py                   # public API; LAZY backend imports (grpc/redis optional)
│   ├── _version.py                   # __version__ = "0.5.0"
│   ├── decision.py                   # frozen Decision / Forecast dataclasses (5 + 3 fields)
│   ├── errors.py                     # ThrottleKitError + 3 operational subclasses (grpc-free)
│   ├── strategies.py                 # Gcra/TokenBucket/FixedWindow/SlidingWindow/SlidingWindowLog + from_spec
│   ├── _contract.py                  # loads vendored Lua + ARGV order; computes sha1 for EVALSHA
│   ├── _grpc.py                      # shared stub-import guard + gRPC→exception mapping (Fleet/Monitor)
│   ├── service_backend.py            # gRPC ServiceBackend + Admission + _HeartbeatPump (sync)
│   ├── service_backend_async.py      # grpc.aio twin
│   ├── redis_backend.py              # direct Lua RedisBackend (check only, sync)
│   ├── redis_backend_async.py        # redis.asyncio twin (reuses _decode/_is_noscript)
│   ├── fleet_backend.py              # Tier-2 FleetBackend + Lease + LeasedLimiter (sync)
│   ├── fleet_backend_async.py        # async twin
│   ├── lease_spender.py              # PURE port of the core's leased L1 spend (no transport)
│   ├── monitor_backend.py            # read-only MonitorBackend (get_snapshot) sync
│   ├── monitor_backend_async.py      # async twin
│   ├── headers.py                    # decision_headers() — IETF / legacy / both rate-limit headers
│   ├── ratelimit.py                  # rate_limit decorator, RateLimited, bind_policy, Checker
│   ├── py.typed                      # PEP 561 typed marker
│   ├── _scripts/                     # RUNTIME data (shipped in the wheel)
│   │   ├── manifest.json             # 5 strategies × {check, read} = 10 entries; sha256 + ARGV order
│   │   └── *.check.lua / *.read.lua  # the core's vendored Lua (10 files, 173 lines total)
│   └── contrib/                      # framework adapters (each needs its own extra)
│       ├── fastapi.py / starlette.py / django.py / flask.py
│       └── __init__.py               # re-exports the shared helpers
├── scripts/
│   ├── sync_contract.py              # vendors proto+vectors+Lua from the core repo, sha256-pinned
│   ├── gen_proto.py                  # grpc_tools.protoc → src/throttlekit/_generated/ (git-ignored)
│   └── check_wheel_stubs.py          # CI: assert stubs are inside the built wheel
├── examples/                         # 6 runnable scripts + policies.yaml
├── tests/                            # 18 modules, 2469 lines
├── hatch_build.py                    # custom build hook: generate gRPC stubs into the wheel
├── pyproject.toml                    # hatchling; deps; ruff/mypy/pytest config
└── .github/workflows/{ci,release}.yml
```

Note `src/throttlekit/_generated/` (the `throttlekit_pb2.py` / `throttlekit_pb2_grpc.py` gRPC stubs) is **git-ignored and not committed** — generated from the vendored proto at install/build time. It is force-included into the wheel via `[tool.hatch.build.targets.wheel].artifacts` + the build hook.

## Code walkthrough (the important parts)

**`__init__.py`** — defines `__all__` and routes backend names through a `__getattr__` **lazy loader** (`_LAZY` dict). `import throttlekit` therefore needs *neither* grpc nor a redis client; the heavy backends are imported only on first attribute access. Eagerly importable surface = the strategies, domain types, errors, and the dependency-light ergonomics (`decision_headers`, `rate_limit`, `bind_policy`, `Checker`, `RateLimited`).

**`decision.py`** — frozen `Decision(allowed, limit, remaining, reset_at, retry_after_ms)` (the five frozen core fields) and `Forecast(spendable_now, next_replenish_at, full_at)`. snake_case to match Python protobuf; JSON vectors' camelCase keys are mapped explicitly where consumed.

**`strategies.py`** — five frozen dataclasses (`Gcra`, `TokenBucket`, `FixedWindow`, `SlidingWindow`, `SlidingWindowLog`), each declaring a `kind` ClassVar and a `params()` returning **manifest ARGV names** (camelCase, e.g. `periodMs`) → configured values. Pure config, **no math**. `from_spec(kind, options)` is the bridge the conformance harness uses to build a strategy from a golden-vector `strategy.{kind, options}` spec.

**`_contract.py`** — loads `_scripts/manifest.json` and the `*.lua` beside it. `script(strategy, role)` returns a frozen `Script` carrying source, ordered `argv` (from the manifest — the wire is the source of truth for marshalling), and **`sha1(source)` hex** (the key Redis uses for `EVALSHA`). Cached. The wire `contractVersion` is exposed via `contract_version()`.

**`redis_backend.py`** — the direct door. `check(key, cost=1, now=None)`: looks up the `check` script, builds a `values` dict `{now, cost, **strategy.params()}`, projects it into `argv` in the manifest's order, prefixes the key (`f"{prefix}:{key}"`), and runs it. `_eval` tries `EVALSHA` first and falls back to `EVAL` on `NOSCRIPT` (detected client-agnostically by `_is_noscript`, mirroring the Node store's `isNoScript`). `_decode` reads the five-int reply tuple. `now=None` sends the sentinel `0` → the Lua uses the **Redis server clock** (`redis.call('TIME')`), the skew-free fleet default.

**`service_backend.py`** — the gRPC door. Each RPC wraps the stub call and maps `grpc.RpcError` to operational exceptions via `_mapped` (`NOT_FOUND`→`PolicyNotFoundError`, `UNIMPLEMENTED`→`OperationNotSupportedError`, `UNAVAILABLE`→`ServiceUnavailableError`). A *denial* is a normal `Decision`, never an error. `admit()` returns an `Admission` context manager holding a server lease; `_HeartbeatPump` is a lazily-started daemon thread that batches `Heartbeat` for opt-in long holds and marks reclaimed leases.

**`lease_spender.py`** — a **pure, transport-free** port of the core's `twoTier(leased, windowCoupled)` L1 path: `apply_lease` adds a grant's capacity; `spend(now, cost)` serves from local credits (synthesizing an allow, byte-identical to the core's `synthAllow`) or returns `None` ("refresh needed") — it **never synthesizes a denial**. The window-coupled discard zeroes leftover credits at the window boundary. `now` injected per call for determinism.

**`ratelimit.py`** — the `Checker` protocol (`key -> Decision | Awaitable[Decision]`), `bind_policy(backend, policy)` (turns a service backend into a checker; preserves sync/async nature), and the `rate_limit` decorator that works on both sync and `async def` functions (detected via a custom `_is_async_callable` that also catches async-`__call__` callable classes). On denial it raises `RateLimited` (deliberately **not** a `ThrottleKitError` — a denial is the backend succeeding).

## The two transports

### gRPC — the proto and the calls

`contract/throttlekit.proto` (`package throttlekit.v1`, proto3, 345 lines) defines **three services** on one port:

- **`RateLimiter`** — `Check`, `CheckMany`, `Peek`, `Forecast`, `Debit`, `Admit`, `Release`, `Heartbeat`. `Decision` pins five fields (tags 1–5; tags 6–15 `reserved` for additive attributes). The `Admit`/`Release`/`Heartbeat` trio is a TTL + heartbeat + reclaim-on-crash lease lifecycle mirroring the core's node↔coordinator contract.
- **`Monitor`** — `GetSnapshot` and a server-streaming `Watch` (live denial feed). Read-only, loopback-only by default unless a `x-monitor-secret` is presented.
- **`Fleet`** — `Reserve` (one unary RPC in v1) for Tier-2 client-held leasing; `x-fleet-secret`-gated. Carries an `Axis` enum (`RATE`/`CONCURRENCY`/`TOKEN_BUDGET`).

**Client coverage of the proto** (a real gap to note): the clients implement `Check`/`CheckMany`/`Peek`/`Forecast`/`Debit`/`Admit`/`Release`/`Heartbeat` (`ServiceBackend`), `GetSnapshot` (`MonitorBackend`), and `Reserve` (`FleetBackend`). The streaming **`Watch` RPC is defined in the proto but not implemented** in either `MonitorBackend` or `MonitorBackend` async — only `get_snapshot` exists. `MonitorSnapshot` decodes the typed envelope and exposes the full `raw_json` via `.parsed()`.

The stubs are generated by `scripts/gen_proto.py` (`grpc_tools.protoc`), which also rewrites protoc's absolute `import throttlekit_pb2` to a package-relative `from . import`. Generated stubs are git-ignored and regenerated at build time by `hatch_build.py`.

### Direct-Redis — the Lua, and where it comes from

The Lua is **vendored from the core, byte-for-byte**, not hand-ported. `scripts/sync_contract.py` reads the core's `wire/scripts/manifest.json`, copies exactly the manifest-declared `*.lua` set into `src/throttlekit/_scripts/`, and these ship inside the wheel (so they resolve identically in a source checkout and an installed wheel).

`manifest.json` declares **5 strategies × {check, read} = 10 scripts**, each with `argv` order, `keys`, and a **sha256**. The `check` scripts are the ones `RedisBackend` runs (e.g. `gcra.check.lua` — 30-odd lines of GCRA: `T = period/limit`, `tau = T*burst`, derives `now` from `redis.call('TIME')` on the `0` sentinel, writes the new TAT with `SET ... PX`). The `read` scripts are present but minimal one-liners (e.g. `gcra.read.lua` = `return redis.call('GET', KEYS[1])`) and **not used by `RedisBackend`** — read-derived ops (`peek`/`forecast`) deliberately route through the service door so the core, not a re-derived client port, computes them.

Two distinct hashes are in play: the manifest stores **sha256** (the integrity/drift record — verified, e.g. `gcra.check.lua` = `53d10f…`), while `_contract.py` computes **sha1** over the same bytes (what Redis caches the script under for `EVALSHA`). `test_redis_backend.py` proves the client's sha1 matches Redis's own script cache — the cross-language cache-sharing property (a Node fleet and this client `EVALSHA` the same cached script).

## Conformance — how bit-identical behavior vs. the core is tested (golden vectors)

`contract/golden-vectors.json` carries **18 suites** across three primitives, all generated by the Node oracle (`generatedFrom: throttlekit@1.4.0`, `decisionFields: [allowed, limit, remaining, resetAt, retryAfterMs]`):

- **9 `rateLimit` suites** — one or more per strategy (gcra burst/fractional-T/cost-gt-1/large-epoch, tokenBucket, fixedWindow, slidingWindow, slidingWindowLog). Each is a `strategy` spec + an `ops` timeline (`{now, cost, expect:{...five fields}}`).
- **3 `tokenBudget` suites** — `{now, tokens, expect}` (per-token, crossing-debit, window-roll).
- **6 `lease` suites** — scripted `events` (`grant`/`spend`/…) with `limit`, `ttlMs`, `windowCoupled`.

The proofs:

- **`tests/test_redis_backend.py`** (the capstone): replays **every `rateLimit` golden vector** — the full, time-parametrized timeline — through `RedisBackend` → vendored Lua → **real Redis**, asserting all five reply fields against the Node oracle. Because the direct path puts an explicit `now` in ARGV, it can do the rigorous replay the cross-process service door can't. It adds a `BASE = 1000` clock offset (documented at length): the smallest window-aligned offset that clears the `now=0` server-time sentinel *and* keeps GCRA's fractional-rate cold-path arithmetic IEEE-754-exact (a large offset like `1_000_000` trips a float-boundary flip on the `gcra/fractional-T` suite). Four fields are asserted equal; `resetAt` (the one absolute-epoch field) `== oracle + BASE`.
- **`tests/test_lease_spender.py`** replays the **6 `lease` suites** through the pure `LeaseSpender`, proving byte-identity with the core's leased L1 path.
- **`tests/test_contract.py`** is the pure-Python **drift-gate** (no gRPC/Redis): vendored proto + vectors must match `manifest.sha256`; each shipped `.lua` must match its sha256 in `_scripts/manifest.json`; the pinned `contractVersion == "1"`; the reply tuple must agree across the scripts manifest and the vectors. Re-running `sync_contract.py` after a core change surfaces drift as a reviewable diff.

`tokenBudget` suites have no extracted wire script, so they're covered by the Node in-process path / service door and skipped in the Redis replay.

## Public API — the surface a Python user calls

Exported from `throttlekit` (selected signatures):

**Service door (gRPC) — sync `ServiceBackend` / async `AsyncServiceBackend`:**
```python
ServiceBackend(target="localhost:50051", *, credentials=None, heartbeat_interval=1.0)
.check(policy, key, cost=1) -> Decision
.check_many(policy, keys, cost=1) -> list[Decision]
.peek(policy, key) -> Decision
.forecast(policy, key, cost=1) -> Forecast
.debit(policy, key, tokens=1) -> Decision           # token-budget cost axis
.admit(policy, key, cost=1, *, hold=0, value=1, heartbeat=False) -> Admission
.close()                                             # context manager
```
`Admission` is a context manager: `.allowed`, `.held`, `.reclaimed`, `.binding_axis`, `.release(dropped=False)`. (Async twin awaits, uses `async with`, `aclose()`.)

**Direct door (Redis) — sync `RedisBackend` / async `AsyncRedisBackend`:**
```python
RedisBackend(client, strategy, *, prefix="")          # client: any evalsha/eval object
.check(key, cost=1, *, now=None) -> Decision          # check ONLY; now=None => Redis server clock
```

**Tier-2 fleet leasing:**
```python
FleetBackend(target="localhost:50051", *, credentials=None, secret=None)
.reserve(policy, *, domain="", wants=1, has=0, used=0, axis="rate") -> Lease
.leased(policy, *, domain="", batch=1, window_coupled=True) -> LeasedLimiter
LeasedLimiter.check(cost=1, *, now=None) -> Decision   # serves locally, refreshes on shortfall
LeaseSpender(*, limit, ttl_ms=0, window_coupled=True)  # pure; .apply_lease / .spend(now, cost) / .reset
```

**Monitor (read-only):** `MonitorBackend(...).get_snapshot() -> MonitorSnapshot` (`.policies`, `.raw_json`, `.parsed()`); async twin.

**Strategies:** `Gcra(limit, period_ms, burst)`, `TokenBucket(capacity, refill_per_sec)`, `FixedWindow(limit, window_ms)`, `SlidingWindow(limit, window_ms, buckets)`, `SlidingWindowLog(limit, window_ms)`, `from_spec(kind, options)`.

**Ergonomics:** `decision_headers(decision, style="ietf"|"legacy"|"both", *, now_ms=None) -> dict[str, str]`; `rate_limit(checker, *, key, cost=1)` decorator (sync or async); `RateLimited(decision)`; `bind_policy(backend, policy)`; `Checker`, `OnUnavailable`.

**Domain/errors:** `Decision`, `Forecast`, `ThrottleKitError`, `PolicyNotFoundError`, `OperationNotSupportedError`, `ServiceUnavailableError`.

**Sync vs. async:** every transport-bearing door has a sync class and an `Async*` twin (gRPC over `grpc.aio`, Redis over `redis.asyncio`). The async twins reuse the sync pure helpers (`_decode`, `_is_noscript`, `_decision`, `_forecast`, `_mapped`). `LeaseSpender` is sync-only (it's pure, no I/O). `rate_limit` handles both, running a sync checker in a worker thread (`asyncio.to_thread`) from an async call site so it never blocks the loop.

## Tech stack & dependencies

- **Build:** Hatchling, with a custom build hook (`hatch_build.py`) that runs `gen_proto.py` to generate the gRPC stubs into the wheel; `grpcio-tools~=1.80.0` pinned in `[build-system].requires` (the pyproject comment explains: an unpinned floor let the isolated build float to protoc 6.33.5, whose gencode out-ran the declared `protobuf>=6.31.1` runtime floor — release smoke caught it).
- **Runtime:** `grpcio>=1.60`, `protobuf>=6.31.1,<7` (grpcio does not pull protobuf in; both needed for the service door to import).
- **Optional extras:** `[redis]` (`redis>=5`), `[fastapi]`/`[starlette]`/`[django]`/`[flask]`, `[all]`, `[dev]`.
- **Tooling:** ruff (line-length 100, `E,F,I,UP,B`, ignore `E501`; `ruff==0.15.15` exact-pinned for format-check reproducibility), mypy `strict` over `src/throttlekit` (generated stubs and untyped frameworks excluded/`Any`-treated), pytest with `integration` and `redis` markers.
- **Vendored, not depended-on:** the core's Lua, proto, and golden vectors (sha256-pinned via `sync_contract.py`).

## Build / install / test (actual commands; packaging)

```bash
pip install throttlekit-py             # gRPC ServiceBackend
pip install "throttlekit-py[redis]"    # + redis client for the direct door
pip install "throttlekit-py[all]"      # + all four framework integrations

# Develop
pip install -e .[dev]
python scripts/sync_contract.py        # vendor proto+vectors+Lua from the core (../GreenfeildProject)
python scripts/gen_proto.py            # generate gRPC stubs into src/throttlekit/_generated/
pytest                                 # unit + contract; Redis/service tests skip if backend absent
ruff check . && ruff format --check . && mypy
```

**Packaging:** sdist carries the proto + Lua + scripts; a wheel build (direct or wheel-from-sdist) runs `hatch_build.py` → regenerates the stubs and force-includes `src/throttlekit/_generated/**/*.py` via `artifacts` (past `.gitignore`). `check_wheel_stubs.py` asserts the stubs are actually inside the built wheel — guarding the historical bug where a stub-less wheel's `import ServiceBackend` failed only on a fresh install.

**Release:** `release.yml` publishes to PyPI on a `v*` tag via **trusted publishing (OIDC, no token)**, smoke-tests the wheel in a clean venv (incl. installing the exact `protobuf==6.31.1` floor to prove it satisfies the gencode pin), then cuts a GitHub Release from the CHANGELOG section. `skip-existing: true` makes re-runs idempotent.

## Tests

**18 test modules, 2,469 lines.** Coverage by area:

- **Conformance / contract:** `test_contract.py` (drift-gate), `test_redis_backend.py` + `test_redis_backend_async.py` (golden-vector replay through real Redis, `@pytest.mark.redis`), `test_lease_spender.py` (lease-vector replay), `test_strategies.py`.
- **gRPC backends:** `test_service_backend.py` + `_async` (start a **real `throttlekit-server`** via node, `@pytest.mark.integration`; skip if node / stubs / built server absent), `test_service_backend_postgres.py` + `test_service_backend_dynamodb.py` (store-agnostic conformance against `--store postgres`/`dynamodb` servers — proving a Python client reaches non-Redis stores through the door), `test_fleet_backend.py` (248 lines, the largest), `test_monitor_backend.py`, `test_async_heartbeat_pump.py`.
- **Ergonomics & contrib:** `test_headers.py`, `test_ratelimit_decorator.py`, and one per framework (`test_contrib_fastapi/starlette/django/flask.py`, driven by in-process TestClients).

CI (`ci.yml`) runs four jobs: **lint** (ruff), **test** (3.10–3.13 matrix, `mypy --python-version` per floor + pytest, with a `redis:7-alpine` service container so the vector replay actually *runs*), **contract** (the standalone pure-Python drift-gate), and **build** (build the real wheel, assert stubs inside, clean-install and import both doors). The service-door integration tests skip in CI (no Node server built there).

## Status, completeness & notable gaps

- **Published and maintained:** on PyPI through `0.5.0`, six releases, automated trusted-publishing pipeline. Classified `Alpha`.
- **Honest about its experimental edge:** the raw Lua wire is **not frozen** — `manifest.json` and the golden vectors both ship `"frozen": false` — so the `RedisBackend` is explicitly experimental and may change with the core's scripts. The proto service door is the "comfortable freeze" (additive-evolvable).
- **Gaps:**
  - **`Watch` (the streaming Monitor RPC) is unimplemented.** The proto defines `Monitor.Watch(stream)`, but the `MonitorBackend`/`AsyncMonitorBackend` clients only call `GetSnapshot`. No streaming denial feed on the Python side.
  - **`RedisBackend` is `check`-only by design** — no `peek`/`forecast`/`check_many`/`debit`/`admit` on the direct door (those route through the service door). This is a deliberate one-oracle constraint, not an oversight, but it means the direct door is a strict subset.
  - The vendored `*.read.lua` scripts are shipped (and integrity-checked) but **unused** by the client.
  - `admit`'s `hold`/`value` joint-LP terms are marked experimental in both proto and docstrings.

Overall: a thorough, well-tested, properly-packaged client SDK whose central claim (bit-identical to the core) is genuinely backed by executable conformance, not just asserted.

## README vs. code

The README is unusually accurate; discrepancies are minor:

- **Accurate:** dist-vs-import name split, the two-door model, `RedisBackend` being `check`-only with `now` defaulting to the Redis server clock, the lazy-import design, the sync/async twins, the `from_spec`/`bind_policy`/`decision_headers` ergonomics, the sha256-pinned vendoring flow, the `BASE`-offset conformance proof, and the store-agnostic service door (memory/Redis/Postgres/DynamoDB) — all match the code and tests.
- **Watch not mentioned but also not implemented:** the README describes `MonitorBackend.get_snapshot()` only (consistent with the code) and does **not** claim a `Watch`/streaming client — so no over-claim there, but a reader comparing to the proto should know `Watch` exists on the wire yet has no Python client.
- **`debit(policy, key, tokens)`** is documented in the README and implemented on both `ServiceBackend` and `AsyncServiceBackend` — matches.
- **GALE/TALE/TLA⁺ guarantees** the README leans on (machine-checked overshoot bound, etc.) are properties of the **core**, not this repo — the README is clear that the client merely *reaches* the proven core, which is correct: no such proofs live in this codebase (the `lease`-vector replay in `test_lease_spender.py` is the local witness of the leased-path byte-identity).
- **Minor version-doc lag:** `pyproject` and `_version.py` are `0.5.0` and PyPI's latest is `0.5.0`, but the checked-in `dist/` still holds stale `0.4.0` artifacts (a local-build leftover, harmless — the real release pipeline rebuilds in CI).

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\throttlekit-py. Source of truth: https://github.com/AmeyaBorkar/throttlekit-py*
