# ThrottleKit (core)

> A TypeScript rate-limiting library whose distinguishing claim is *provability*: rate-limit algorithms are written as pure functions of `(state, now, cost)`, every storage backend implements exactly one atomic `apply` primitive (so the *same* algorithm runs in memory, on Redis via one `EVALSHA`, and on Postgres via an advisory-lock transaction — proven bit-identical by a dual-path conformance suite), and the distributed two-tier leasing engine has its global overshoot bound machine-checked in TLA+/TLC. It ships a synchronous allocation-light check API, multi-dimensional limits, fixed-memory Count-Min-Sketch DDoS shedding, and a wide set of framework adapters — with zero runtime dependencies.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/throttlekit |
| **Visibility** | Public |
| **Category** | infra |
| **Primary language(s)** | TypeScript (with embedded Redis Lua and TLA+ specs) |
| **Local path** | `C:\Users\ameya\Documents\GreenfeildProject` |
| **Default branch** | `main` |
| **Lines of code (computed)** | ~24,502 lines across `src/**.ts`; ~33,106 lines of tests in `test/**.ts`. Counting `src` + `server/src` together: ~33,553 TS lines. |
| **Source files (computed)** | 120 `.ts` files in `src/`. Repo-wide (src/test/wire/spec/server): 374 `.ts`, 10 `.lua`, 5 `.tla`, 5 TLC `.cfg`, 1 `.proto`. 174 files under `test/`. |
| **npm package** | `throttlekit` — **published**. `package.json` version `1.5.2`; npm registry `dist-tags.latest` = `1.5.2` (matches). Release history spans `0.1.0 → 1.5.2`. |
| **Key dependencies** | **Zero runtime dependencies.** Optional peer deps: `ioredis`, `pg`, `express`, `fastify`, `hono`, `koa`, `@opentelemetry/api` (all marked optional). Dev/bench: `vitest`, `fast-check`, `tsup`, `biome`, `@bufbuild/buf`, `publint`, `@arethetypeswrong/cli`, plus `redis`/`ioredis`/`pg` for store tests and `express-rate-limit`/`rate-limiter-flexible` for benchmark comparison. |
| **License** | MIT (Copyright (c) 2026 Ameya Borkar) |
| **Last commit** | `e2798aa` — `chore(server): release throttlekit-server 0.4.2` (2026-06-13). First commit 2026-05-25; 424 commits on `main`. |

---

## What it actually is

ThrottleKit is a framework-agnostic rate-limiting library for Node and the web, built around three cleanly separated concerns (the comment in `src/core/types.ts` states this explicitly):

1. **Strategies** — pure algorithms. A `Strategy<S>` (`src/core/types.ts`) is a deterministic transition `check(state, now, cost) -> StrategyOutcome<S>` with no I/O and no clock reads. Built-ins: `gcra` (default), `tokenBucket`, `fixedWindow`, `slidingWindow`, `slidingWindowLog`, `quota`, `leakyBucket` (`src/algorithms/`).
2. **Stores** — one atomic primitive. A `Store` (`src/core/types.ts`) exposes a single mutating method, `apply<S,R>(key, transform): Promise<R>`, plus an optional synchronous `applySync`. "Adding a backend is implementing one method; adding an algorithm never touches a store."
3. **Adapters** — thin glue to a framework (Express, Fastify, Hono, Koa, Next, Remix, SvelteKit, Elysia, tRPC, NestJS, gRPC, Lambda, Web `fetch`) under `src/adapters/`.

The wiring is done by `rateLimit(...)` in `src/core/limiter.ts`, which builds **one transform** (`src/core/transform.ts`) carrying both the pure JS transition and (when the strategy ships one) the atomic Lua form. An in-memory store runs the JS transition locally; a Redis store collapses it to one round trip — *from the same configuration*. That single shared transform is the technical heart of the "one transform proven bit-identical" claim.

Beyond the core limiter, the package is genuinely large and spans well past basic rate limiting: a two-tier leasing engine (`src/twotier/`, branded **GALE**), a concurrency-limiting tier (`src/concurrency/`), an LLM cost/token-budget admission stack (`src/admission/`, branded **TALE**), cross-region federation (`src/federation/`), a Count-Min-Sketch DDoS limiter (`src/sketch/`), multi-dimensional limits (`src/multi/`), policy planning/replay testkit (`src/policy/`, `src/testkit/`), and observability/OTel taps (`src/observability/`).

Per-directory size gives a sense of the weight: `src/adapters` ~4,072 lines, `src/admission` ~3,633, `src/federation` ~2,500, `src/twotier` ~2,369, `src/concurrency` ~2,137, `src/testkit` ~1,674, `src/algorithms` ~1,557, `src/core` ~800.

---

## Where it sits in the ThrottleKit family (core / py / site)

This repo (`throttlekit`) is the **TypeScript core** and the source of truth for the family:

- **`throttlekit` (this repo)** — the TS core library on npm, plus an embedded gRPC service door (`server/`, published separately as `throttlekit-server`, currently `0.4.2`) and a vendored wire protocol (`wire/`) so other-language clients can talk to the same Redis with bit-identical decisions.
- **`throttlekit-py`** — the Python client (the family's "polyglot today" surface). Its contract lives partly *in this repo*: `wire/WIRE-PROTOCOL.md`, `wire/scripts/*.lua`, and `wire/vectors/golden-vectors.json` are the language-neutral conformance vectors a Python `RedisBackend` must satisfy; `docs/design/15-python-client.md` describes it. The gRPC `throttlekit.proto` (`wire/throttlekit.proto`, `server/proto/`) is the recommended polyglot path.
- **`throttlekit-site`** — the Astro marketing site (`throttlekit.in`). Note: a full Astro site is also checked into *this* working tree under `research/website/site/` (its own nested `.git`), used during design; the published marketing site is the separate `throttlekit-site` repo.

An infra/ index should link all three. Discrepancy to flag: the README's comparison table says "Polyglot from one verified core (**Python today**)" — consistent with `throttlekit-py` being the live sibling client.

---

## Architecture & how it's structured (annotated tree, package layout, exports)

```text
src/
  index.ts                 # main barrel — the `throttlekit` entry point
  version.ts               # single-sourced version ("1.5.2"); asserted == package.json
  core/                    # the domain model + wiring (the small, proven core)
    types.ts               #   Clock, Decision, Strategy, Store, Transform, ApplyOutcome, LuaProgram, Limiter
    limiter.ts             #   rateLimit() — builds the limiter; sync fast path
    transform.ts           #   decisionTransform / readOnlyTransform — the ONE shared transform
    lua.ts                 #   LUA_NOW preamble + decodeDecision (shared 5-int reply decoder)
    combine.ts, key.ts, clock.ts, errors.ts, validate.ts, duration.ts, math.ts
  algorithms/              # pure strategies: gcra, token-bucket, fixed/sliding-window, sliding-window-log,
                           #   quota (calendar-aware), leaky-bucket, calendar
  stores/                  # memory.ts (CLOCK eviction + timing wheel), timing-wheel.ts
  redis/                   # store.ts (EVALSHA + OCC fallback), clients.ts, index.ts
  postgres/                # store.ts (advisory-lock transaction), index.ts
  dynamodb/, deno/, cloudflare/   # additional backends (DynamoDB, Deno KV, CF KV/D1/Durable Object)
  twotier/                 # GALE: leasing, weighted-fair escrow, region-fair pool, lease sizer (EOQ)
  concurrency/             # adaptive + distributed concurrency guards; Redis/PG coordinators
  admission/               # TALE: token budgets, fused Lua, fluid LP, unified rate×concurrency×cost
  federation/              # cross-region coordinators, regional escrow, window-coupled lease
  multi/                   # multi-dimensional (all/any) limits, with a fused multi-Lua
  sketch/                  # Count-Min-Sketch DDoS limiter + mergeable cluster sketch
  policy/                  # policy plans: render/plan/gate/artifact/corpus (replay a change → allow↔deny diff)
  testkit/                 # test harness + replay engine (record/score/diverge)
  observability/           # decision taps, OpenTelemetry
  analytics/, security/, http/, config/, cli/, adapters/
spec/                      # 5 TLA+ specs + TLC .cfg + README (the formal models)
wire/                      # vendored Lua scripts, golden vectors, .proto, manifest (polyglot contract)
server/                    # embedded gRPC service (throttlekit-server, released separately)
docs/                      # DESIGN-NOTES, FORMAL-MODEL, FAILURE-MODES, + docs/design/01..18
test/                      # 174 files, ~33k lines, mirroring src by subdir
bench/, research/          # benchmarks; design research (incl. an arXiv paper draft + an Astro site)
```

**Package layout / exports.** `package.json` declares **25 functional subpath entry points** plus `./package.json` (26 `exports` keys total). Each has dual `import`/`require` conditions with separate `.d.ts`/`.d.cts` types — full ESM + CJS, tree-shakeable, `"sideEffects": false`. Subpaths: `.`, `./redis`, `./postgres`, `./dynamodb`, `./deno`, `./cloudflare`, `./express`, `./fetch`, `./hono`, `./next`, `./fastify`, `./koa`, `./grpc`, `./lambda`, `./sveltekit`, `./remix`, `./elysia`, `./trpc`, `./nest`, `./otel`, `./testkit`, `./config`, `./twotier`, `./federation`, `./policy`. There is also a `bin`: `throttlekit` → `./dist/cli.js`. (README says "24 entry points"; the code currently exposes 25 functional subpaths — a minor undercount in the README.)

The main barrel `src/index.ts` re-exports the public surface: `rateLimit`, all strategies, `MemoryStore`, the two-tier family (`twoTier`, `weightedFairEscrow`, `federatedWeightedFairEscrow`, `regionFairPool`, `leaseSizer`, `eoqOptimum`, …), concurrency guards, the admission/TALE family, `multiRateLimit`/`all`/`any`, `sketchRateLimit`/`mergeableSketch`, `buildRateLimitHeaders`, `createEnforcer`, `clientIp`, `withAnalytics`, `tapDecisions`, plus all the types.

---

## Code walkthrough (the important parts) — module by module

### `core/types.ts` — the contracts
Defines `Decision` (`{ allowed, limit, remaining, resetAt, retryAfterMs }`, **all integers and readonly**, with a documented "frozen + evolvable" SemVer stance), `Strategy<S>` (the pure algorithm interface, with optional `lua`, `peek`, `forecast`, `readState`), `Store` (one `apply`, optional `applySync`/`resetSync`/`close`), `Transform<S,R>` (a pure RMW function that optionally carries a `LuaInvocation` for atomic acceleration), `ApplyOutcome` (`{ state, result, ttlMs, persist }`), and `LuaProgram` (`script` + `buildKeys` + `buildArgv`). The comment is explicit that every numeric field is an integer "so the JavaScript and Redis-Lua execution paths can produce bit-identical values (Redis truncates Lua numbers to integers on reply)."

### `core/transform.ts` — the one shared transform
`decisionTransform(strategy, now, cost)` wraps `strategy.check` and, if the strategy ships Lua, attaches a `LuaInvocation` (`program`, `now`, `cost`, `decode: decodeDecision`). `readOnlyTransform(...)` builds a non-mutating transform (`persist: false`) for `peek`/`forecast`. Both the limiter and the two-tier engine call `decisionTransform`, so **a single code path produces both the local and the distributed decision** — this is the structural basis of bit-identicality.

### `core/limiter.ts` — `rateLimit()`
Wires strategy + store + clock + key prefix. Notable: a **reused `syncTransform`** closure with mutable `syncNow`/`syncCost` slots, justified by Node's single-threaded execution (the slots are read before any other call can run through a synchronous `applySync`), so the synchronous hot path allocates no per-call closure. `check()` takes a sync fast path when `store.applySync` exists (resolving an already-computed value as a Promise) and only falls to async `store.apply` + a fresh `decisionTransform` for genuinely async stores. Implements `check`, `checkSync`, `checkMany`, `checkManySync`, `peek(Sync)`, `forecast(Sync)`, `reset`, `close` (closes the store only if the limiter created it).

### `stores/memory.ts` — `MemoryStore`
In-process store; atomicity is free because Node is single-threaded, so `applySync` needs no locks. Keeps native values (no JSON). When `maxKeys` is set, bounds cardinality with a **CLOCK (second-chance) eviction** policy over a ring, plus a freed-slot reclaim list, so an adversarial flood of unique keys can't OOM. Lazy inline expiry off each entry's own `exp`, backed by a `TimingWheel` (`src/stores/timing-wheel.ts`) and an `unref`'d background sweep. `apply` simply resolves `applySync`.

### `redis/store.ts` — `RedisStore`
Structural `RedisClientLike` (ioredis/node-redis satisfy it). Built-in strategies run their atomic Lua via **`EVALSHA` with `EVAL` fallback on `NOSCRIPT`** (a `Map` caches the sha1). `useServerTime` (default true) passes `ARGV[1]=0` so the script reads the Redis `TIME` clock — skew-proof for a shared fleet. Strategies *without* Lua fall back to **optimistic concurrency** (`WATCH`/`GET`/`MULTI`/`EXEC`) with bounded retries (default 5), JSON-encoding state.

### `postgres/store.ts` — `PostgresStore`
Distributed store with no Redis. Runs the limiter's **existing pure JS transform** (the very code the in-memory store runs — "there is no Postgres-specific algorithm to keep in sync") inside a transaction serialized per key by a **transaction-scoped advisory lock** (`pg_advisory_xact_lock(hashtextextended(key,0))`). The doc-comment explains *why* an advisory lock, not `SELECT … FOR UPDATE`: `FOR UPDATE` can't lock a not-yet-existing row, so two first-touch transactions could race. State is the same JSON text the Redis OCC path writes, so doubles round-trip exactly. Table identifier is regex-validated (`^[A-Za-z_][A-Za-z0-9_]*(\.…)?$`) since identifiers can't be parameterized. Async-only (no `applySync`), with auto-create + an `unref`'d sweep.

### `sketch/index.ts` — Count-Min-Sketch DDoS
A full `CountMinSketch` over a single `Uint32Array` (footprint a function of `epsilon`/`delta` only, never key count), `width = ceil(e/epsilon)`, `depth = ceil(ln(1/delta))` (Cormode & Muthukrishnan 2005), with FNV-1a double-hashing, optional Estan–Varghese **conservative update** (default), saturating arithmetic at `0xffffffff` to preserve the never-underestimate invariant, plus `snapshot`/`toBytes`/`mergeSnapshot` for a cluster-wide `mergeableSketch`. `sketchRateLimit` is a fixed-window, **check-before-add** approximate limiter that *never over-admits* (a hard, non-probabilistic guarantee) and only over-denies by `≤ epsilon·N` w.p. `≥ 1-delta`.

### `multi/index.ts` — multi-dimensional checks
`all(dimensions)` / `any(dimensions)` compose named per-axis `Dimension`s (key fn, strategy, optional weight). On a sync store it reads every dimension, decides via a `combine` rule, then commits **all-or-none** in one uninterrupted turn (no partial consume). On Redis it fuses every dimension into a **single Lua round trip** (`MULTI_LUA`) supporting gcra/tokenBucket/fixedWindow, with per-type math mirroring the standalone scripts so JS and Lua composite decisions are bit-identical.

### `twotier/index.ts` — GALE leasing (see Algorithms)
`twoTier({ strategy, l2, mode, lease, l1 })` with modes `strict` | `cached-deny` | `leased`. The `leased` path keeps per-key local `credits`, coalesces concurrent misses onto one in-flight lease (`pending`), supports `lowWater` proactive refill, `windowCoupled` credit expiry, `returnIdleAfterMs` reclaim, and **adaptive lease sizing** (EOQ learner, `src/twotier/sizing.ts`).

---

## The algorithms — GCRA, two-tier leasing, DDoS sketches, multi-dimensional

### GCRA (default) — `src/algorithms/gcra.ts`
The Generic Cell Rate Algorithm tracks a single number per key, the **theoretical arrival time (TAT)**. `gcra({ limit, periodMs, burst? })` derives `T = periodMs/limit` (emission interval), `tau = T·burst` (burst tolerance), `inc = T·cost`. `check` computes `tatEff = max(tat, now)` (jump-safe), `newTat = tatEff + inc`, `allowAt = newTat - tau`; admits iff `now >= allowAt`. O(1) time and memory. It also implements `peek` (non-consuming current capacity) and `forecast` (`spendableNow`/`nextReplenishAt`/`fullAt`). The atomic Redis form (`GCRA_LUA`) derives `T`, `tau`, `inc` from **the same integer ARGV** with the same rounding, and stores the TAT via `string.format('%.17g', new_tat)` so it round-trips at full IEEE-754 precision — the in-line justification of bit-identicality. A separate read-only Lua (`GCRA_READ_LUA = GET`) backs `peek`/`forecast` without writing.

### Two-tier token leasing + the overshoot bound — `src/twotier/` + `spec/`
In `leased` mode each node leases a `batch` of tokens from L2 (Redis/Postgres) in one round trip and serves them locally at in-process speed (steady-state ≈ 1 round trip per `batch` requests). The **formal result** lives in `spec/`:

- `spec/DistributedLeasing.tla` (+ `.cfg`) — TLA+ model of the default `leased` path (`lowWater=0`, `cost=1`), model-checked with TLC, proving the *tight* per-window bound:
  `admitted_per_window ≤ Limit + |Nodes|·(Batch−1)` — **tightness witnessed by a counterexample trace** (an off-by-one invariant `OvershootTight` is violated reaching `admitted = Limit + N·(Batch−1)`). The README captures TLC's exact run output (config 1: 31 distinct states, depth 10; config 3: 441 distinct states).
- `spec/GaleWindowCoupledLeasing.tla` — the **GALE refinement**: credits expire on the L2 window boundary (the carryover is the *sole* source of cross-window overshoot), collapsing the per-window bound to **exactly `Limit`, independent of fleet size N**. This is the "formally-verified overshoot bound independent of fleet size" of the project brief. In code it is `lease.windowCoupled: true` in `twoTier`, which zeroes `credits` once `now >= lastDecision.resetAt`.
- `spec/GaleFederatedLeasing.tla` + `spec/GaleHeartbeatLeasing.tla` / `GaleHeartbeatHandoff.tla` — the cross-region federation lift (`Nodes → Regions`) proving `admitted ≤ Limit` **independent of region count K**, and heartbeat-based lease handoff.

Crucially, each `.tla` has a **Java-free JS twin** that enumerates the *identical* transition system with BFS and asserts the same invariant and the same distinct-state counts (`test/twotier/leasing-model.test.ts` reproduces TLC's 31 and 441; `test/gale/leasing-variants.test.ts` covers the window-coupled variant). These run in CI in ~150 ms. So the formal model is *checked twice*: by TLC offline and by a CI-runnable model checker.

**Adaptive lease sizing** (`src/twotier/sizing.ts`, GALE "Pillar 2"): an online learner descends onto the EOQ optimum `√(2·orderCost·demand/strandPenalty)` per key, trading coordination against stranding — and, by the Pillar 1 proof, can never loosen the cap.

### DDoS sketches — the fixed-memory structure
A **Count-Min Sketch** (`src/sketch/index.ts`), backed by one `Uint32Array` of `depth × width` counters. The structure is `O((1/epsilon)·ln(1/delta))` and is **independent of the number of distinct keys**, so it tracks an unbounded IP universe in bounded space — the point being that per-key state would itself be the memory-exhaustion vector in a volumetric attack. Conservative-update by default; the rate limiter's only error is in the safe direction (over-deny, never over-admit).

### Multi-dimensional checks
See `multi/index.ts` above: per-axis combine rules (all-allow → tightest headroom; all-deny → largest retryAfter; any-allow → most remaining; any-deny → soonest recovery), all-or-none commit, fused into one Lua round trip on Redis.

---

## Storage backends — memory vs Redis vs Postgres, and the "one transform proven bit-identical" shared core

| Backend | File | Atomicity mechanism | State encoding | Sync? |
|---|---|---|---|---|
| **Memory** | `stores/memory.ts` | Single-threaded RMW (no lock); CLOCK eviction + timing wheel | native values (no serialization) | **yes** (`applySync`) |
| **Redis** | `redis/store.ts` | Atomic Lua `EVALSHA` (EVAL on NOSCRIPT); OCC `WATCH/MULTI/EXEC` fallback for Lua-less strategies | strategy-specific Lua encoding (`%.17g` floats), else JSON | no |
| **Postgres** | `postgres/store.ts` | Per-key transaction-scoped advisory lock; runs the *JS* transform in-transaction | same JSON text as Redis OCC path | no |
| **DynamoDB / Deno KV / Cloudflare** | `dynamodb/`, `deno/`, `cloudflare/` | backend-native conditional/atomic ops | — | no |

The README counts **six stores** (memory, Redis, Postgres, DynamoDB, Deno KV, Cloudflare) — matching the six store-ish exports (`.`/MemoryStore, `./redis`, `./postgres`, `./dynamodb`, `./deno`, `./cloudflare`).

**The "one transform proven bit-identical" shared core.** There is no per-backend re-implementation of any algorithm. Either:
- the store runs the strategy's **vendored Lua** (Redis), which derives every value from the same integer ARGV and clamps identically to the JS `check`; or
- the store runs the strategy's **pure JS `check`** directly inside its atomic primitive (Memory's single-threaded RMW, Postgres's advisory-lock transaction).

Because `Decision` fields are integers and Redis truncates Lua reply numbers to integers, the two execution paths produce equal `Decision` streams. This is *proven*, not asserted: `test/conformance/conformance.test.ts` runs each of ~19 strategy configs through the JS path (`MemoryStore`) and the Redis-Lua path (`RedisStore`) over `40 × 25` randomized `(delta, cost)` timelines and asserts `toEqual` field-for-field (plus a final `peek` equality through the Redis-side HASH/ZSET/string state). `test/conformance/lua-property.test.ts` adds property-based checks. The README's "200-way concurrent read-modify-write" appears in `test/postgres/store.test.ts` (a pool sized for a 200-way concurrency test, asserting N concurrent applies land exactly N). The Redis and Postgres conformance suites are **gated** on `THROTTLEKIT_TEST_REDIS` / `THROTTLEKIT_TEST_POSTGRES` env vars (they `describe.skip` without a live store).

---

## Formal verification / conformance — what's actually in the repo

This is real, not marketing. Concretely present:

- **5 TLA+ specs** (`spec/*.tla`) with matching TLC configs (`spec/*.cfg`): `DistributedLeasing`, `GaleFederatedLeasing`, `GaleHeartbeatHandoff`, `GaleHeartbeatLeasing`, `GaleWindowCoupledLeasing`. `spec/README.md` documents the models, the exact code correspondence (`src/twotier/index.ts`), captured TLC output (state/distinct counts, tightness counterexample), and how to run TLC. Prose writeup in `docs/FORMAL-MODEL.md`.
- **Java-free JS twins** of the specs in `test/twotier/` and `test/gale/` that re-enumerate the same transition systems and assert the same invariants **and the same distinct-state counts** in CI.
- **Golden vectors** — `wire/vectors/golden-vectors.json` (language-neutral `(now, cost)` op streams with `expect`ed `Decision` per field) for gcra/tokenBucket/fixedWindow/slidingWindow/slidingWindowLog, the contract a polyglot client must satisfy. `generatedFrom: throttlekit@1.4.0`, `frozen: false`, `contractVersion: 1`.
- **Vendored Lua + manifest** — `wire/scripts/*.lua` (check + read scripts per strategy) pinned by `wire/scripts/manifest.json` (sha256 per script). `test/wire/conformance-scripts.test.ts` re-derives the scripts from the shipped strategy objects and fails if any byte or sha256 drifted, so the wire can't change silently.
- **Property tests** — `test/property/invariants.test.ts` and `test/conformance/lua-property.test.ts` use `fast-check`.
- **gRPC contract** — `wire/throttlekit.proto`, `buf.yaml`, and `buf breaking` against a committed `wire/throttlekit.baseline.binpb` baseline (npm scripts `wire:check`/`wire:breaking`).

Honest scoping in the code: `mergeableSketch`/`sketchRateLimit` are marked `@experimental` (excluded from 1.x SemVer); the wire protocol is explicitly "documented + behavior-locked, NOT frozen" (`manifest.json frozen:false`); the cluster sketch is documented as an *eventually-consistent estimator*, not a strongly-consistent global limiter.

---

## Public API — the surface consumers call

The everyday surface (from `src/index.ts`):

```ts
import { rateLimit, gcra } from "throttlekit";

const limiter = rateLimit({ strategy: gcra({ limit: 100, periodMs: 60_000, burst: 20 }) });
const d = limiter.checkSync(userId);   // sync, allocation-light
//  d: { allowed, limit, remaining, resetAt, retryAfterMs }   (all integers)
if (!d.allowed) throw new Error(`retry in ${d.retryAfterMs}ms`);
```

`Limiter` (`src/core/types.ts`) exposes: `check`, `checkSync`, `checkMany`, `checkManySync`, optional `peek`/`peekSync`, `forecast`/`forecastSync`, `reset`, optional `close`. Sync variants throw a `ThrottleKitError` unless the store supports `applySync` (e.g. `MemoryStore`).

Other public entry points:
- **Strategies**: `gcra`, `tokenBucket`, `fixedWindow`, `slidingWindow`, `slidingWindowLog`, `quota`, `leakyBucket`.
- **Stores**: `MemoryStore` (core), `RedisStore` (`throttlekit/redis`), `PostgresStore` (`throttlekit/postgres`), plus DynamoDB / Deno / Cloudflare subpaths.
- **Distributed (GALE)**: `twoTier`, `weightedFairEscrow`, `federatedWeightedFairEscrow`, `regionFairPool`, `RedisRegionFairPool`, `LeaseSpender`, `leaseSizer`/`predictiveLeaseSizer`/`eoqOptimum`.
- **Concurrency**: `adaptiveConcurrency`, `distributedAdaptiveConcurrency`, `Redis/Postgres/TestConcurrencyCoordinator`.
- **Admission (TALE)**: `tokenBudget`, `distributedTokenBudget`, `unifiedAdmission`, `fairShare`/`weightedFairShare`/`weightedMaxMin`, `adaptiveThrottle`, `solveFluidLp`, `FusedDispatcher`, `FUSED_GCRA_TOKEN_BUCKET_LUA`, …
- **Multi-dimensional**: `all`, `any`, `multiRateLimit`.
- **DDoS**: `sketchRateLimit`, `mergeableSketch`, `sketchSnapshotFromBytes`.
- **HTTP/adapters**: `buildRateLimitHeaders`, `createEnforcer`, `clientIp`, `hashKey`/`hmacKeyer`, plus framework subpaths.
- **Observability**: `tapDecisions`, `withAnalytics`, `throttlekit/otel`.
- **CLI**: `throttlekit` bin.

---

## Tech stack & dependencies

- **Language**: TypeScript (ESM source, `"type": "module"`, `engines.node >= 18`), targeting ESM + CJS dual output.
- **Build**: `tsup` (`tsup.config.ts`) → `dist/` with `.js`/`.cjs` + `.d.ts`/`.d.cts`.
- **Test**: `vitest` (`vitest.config.ts`), with `@vitest/coverage-v8` and `fast-check` for property tests.
- **Lint/format**: Biome (`biome.json`).
- **Wire/proto**: `@bufbuild/buf` (`buf.yaml`).
- **Package hygiene**: `publint` + `@arethetypeswrong/cli` (`attw`).
- **Runtime deps**: **none** (`dependencies` is absent). All store/framework integrations are *optional peer deps* (`ioredis`, `pg`, `express`, `fastify`, `hono`, `koa`, `@opentelemetry/api`), so consumers install only what they use. `node:crypto` is used in the Redis store for sha1.
- **Lockfile**: `package-lock.json` is present (npm) — not read.

---

## Build / test / publish (actual commands)

From `package.json` scripts:

```sh
npm run build        # tsup → dist (ESM + CJS + types)
npm run dev          # tsup --watch
npm run typecheck    # tsc --noEmit
npm test             # vitest run
npm run test:cov     # vitest run --coverage
npm run check        # lint + typecheck + test + check:package
npm run check:package# build + publint + attw  (package-correctness gate)

# benchmarks
npm run bench        # tsx bench/run.ts
npm run bench:gate   # perf regression gate (--update to rebaseline)
npm run bench:fleet  # server/bench/fleet.ts

# wire / polyglot contract
npm run wire         # regenerate vectors + scripts
npm run wire:check   # buf lint + buf breaking against the committed baseline

# publish (npm)
# prepublishOnly runs `npm run build`; package `files`: dist, README, CHANGELOG, STABILITY, BENCH, LICENSE
```

TLA+ model-checking is run out-of-band with TLC (`tla2tools.jar`, instructions in `spec/README.md`); the CI-safe equivalent is the JS twins under `npm test`.

---

## Tests

- **174 files / ~33,106 lines** under `test/`, mirroring `src` by subdirectory. Largest clusters: `test/adapters` (22 files), `test/admission` (19), `test/twotier`, `test/federation`, `test/concurrency` (13 each), `test/algorithms` (10), `test/core` (9), `test/gale` (8).
- **Dual-path conformance**: `test/conformance/conformance.test.ts` (JS vs Redis Lua over thousands of generated ops) + `lua-property.test.ts`. Gated on `THROTTLEKIT_TEST_REDIS`.
- **Postgres conformance**: `test/postgres/store.test.ts` incl. the 200-way concurrency test. Gated on `THROTTLEKIT_TEST_POSTGRES`.
- **Formal-model twins**: `test/twotier/leasing-model.test.ts`, `test/gale/leasing-variants.test.ts` (BFS state enumeration matching TLC counts).
- **Property tests**: `fast-check` in `test/property/` and `test/conformance/`.
- **Wire conformance**: `test/wire/conformance-scripts.test.ts` (script bytes + manifest sha256 can't drift silently).
- **Version sync**: `test/version-sync.test.ts` asserts `src/version.ts` == `package.json#version`.
- A `coverage/` dir and `cov.log` exist in the tree; `docker-compose.stores.yml` spins up Redis/Postgres for the gated suites.

---

## Status, completeness & notable gaps

- **Mature and published.** v1.5.2 on npm, 424 commits, a 118 KB `CHANGELOG.md`, `STABILITY.md` SemVer policy, `SECURITY.md`, `BENCH.md`, `SCOREBOARD.md`, `THROTTLEKIT.md` (48 KB design doc), 18 component design docs under `docs/design/`. This is a polished, heavily-documented library, not a prototype.
- **Genuinely broad.** The surface goes far past "rate limiting": concurrency limiting, LLM token-budget escrow (TALE), federation, policy plans, replay testkit, Lens monitoring. Much of this is real, tested code; some pieces are explicitly experimental.
- **Experimental / non-frozen areas (self-flagged in code)**: `sketchRateLimit`/`mergeableSketch` (`@experimental`, outside 1.x SemVer); the raw Redis wire (`frozen: false`, recommended polyglot path is the gRPC door); the cluster mergeable sketch is an estimator, not a hard global limiter.
- **Build artifacts in tree.** `dist/`, `coverage/`, `node_modules/`, and a nested Astro site with its own `.git` under `research/website/site/` are checked into the working tree (not ignored here) — noise for a reader, not a correctness issue.
- **Gated infra tests.** The bit-identical Redis/Postgres conformance only runs when the env vars point at a live store; CI must provision them (docker-compose is provided). The TLA+ proofs require an external TLC jar, but the CI-runnable JS twins cover the same invariants.
- **In-progress markers.** Some `src/twotier/index.ts` doc-comments reference TK ticket numbers and "lands in TK-905"/"next commit" notes (e.g. a federated JS twin), indicating active development on the federation/GALE frontier.

---

## README vs. code

The README is unusually accurate for a project this ambitious; most headline claims check out against the source. Verified-true: zero runtime dependencies (no `dependencies` block); the TLA+/TLC overshoot bound `Limit + N·(Batch−1)`, tight, with `windowCoupled` → `Limit` independent of fleet size (`spec/`, twins in `test/`); one algorithm proven bit-identical across backends (`core/transform.ts` + `test/conformance/`); the 200-way concurrent RMW (`test/postgres/store.test.ts`); MIT license; Node ≥ 18; ESM + CJS + types.

Minor discrepancies / things to read critically:
- **Entry-point count.** README says "24 entry points"; `package.json` currently declares **25** functional subpaths (26 `exports` keys incl. `./package.json`). Mild undercount.
- **"169 ns / 5.9M ops/s, effectively allocation-free."** This is a benchmark performance claim, not verifiable by reading source; it depends on hardware and the `bench/` harness. The *mechanism* that makes a fast sync path plausible (reused `syncTransform`, native-value store, no JSON, single-threaded RMW) is real in `core/limiter.ts` and `stores/memory.ts`, but the specific nanosecond figure is unverified here.
- **"Six stores, proven bit-identical."** The bit-identical *proof* (the conformance suite) covers **Memory ↔ Redis** directly and Postgres via shared-transform + the concurrency test; DynamoDB/Deno/Cloudflare are counted as stores but their bit-identicality is by construction (they run the same JS transform), not exercised by the same Redis dual-path suite. So "six backends" is fair, but the rigorous dual-path equality proof is strongest for Memory/Redis/Postgres.
- **Branding.** "GALE" (leasing) and "TALE" (token budget) are README brand names; in code they map to `src/twotier/` and `src/admission/` respectively — no `GALE`/`TALE` literal modules, just the features they describe.
- **Wire freeze.** README leans on "verified core / polyglot from one verified core"; the code is careful to note the raw Redis wire is **not frozen yet** (`wire/WIRE-PROTOCOL.md`, `manifest.json frozen:false`) and the supported polyglot path is the gRPC door. Worth keeping that nuance.

Net: the README oversells nothing structurally — the proofs, the shared transform, and the conformance suite are all genuinely in the repo — and the few gaps are a stale entry-point number, an unverifiable benchmark figure, and the usual "counted vs. rigorously-proven" distinction across the six backends.

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\GreenfeildProject. Source of truth: https://github.com/AmeyaBorkar/throttlekit*
