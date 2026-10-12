# ThrottleKit (core)

> A TypeScript rate-limiting library whose distinguishing claim is *provability*: rate-limit algorithms are written as pure functions of `(state, now, cost)`, every storage backend implements exactly one atomic `apply` primitive (so the *same* algorithm runs in memory, on Redis via one `EVALSHA`, and on Postgres via an advisory-lock transaction — proven bit-identical by a dual-path conformance suite), and the distributed two-tier leasing engine has its global overshoot bound machine-checked in TLA+/TLC. It ships a synchronous allocation-light check API, multi-dimensional limits, fixed-memory Count-Min-Sketch DDoS shedding, and a wide set of framework adapters — with zero runtime dependencies. Since June it has also gained a test-quality layer on top of the proofs: Stryker mutation testing on the decision math (77.5%), fast-check model-based and fuzz suites, and a CI job that runs the store suites against live Redis, Postgres and DynamoDB.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/throttlekit |
| **Visibility** | Public |
| **Category** | infra |
| **Primary language(s)** | TypeScript (with embedded Redis Lua and TLA+ specs) |
| **Local path** | `C:\Users\ameya\Documents\GreenfeildProject` |
| **Default branch** | `main` |
| **Lines of code (computed)** | ~25,255 lines across `src/**.ts` (+753 since 2026-06-16); ~37,550 lines of tests in `test/**.ts` (+4,444). Counting `src` + `server/src` together: ~34,407 TS lines (`server/src` = 9,152). Git-tracked files only; `node_modules`/`dist`/lockfiles excluded. |
| **Source files (computed)** | 120 `.ts` files in `src/` (unchanged). Repo-wide (src/test/wire/spec/server): 391 `.ts`, 10 `.lua`, 5 `.tla`, 5 TLC `.cfg`, 2 `.proto` (`wire/throttlekit.proto` + `server/proto/health.proto`). 188 files under `test/` (was 174). |
| **npm package** | `throttlekit` — **published**. `package.json` version `1.7.0`; npm `dist-tags.latest` = `1.7.0` (matches; published 2026-07-01). 38 versions published, `0.1.0 → 1.7.0`. Sibling `throttlekit-server` (built from `server/`) is `0.4.4` in `server/package.json` and on npm (published 2026-06-30). |
| **Key dependencies** | **Zero runtime dependencies.** Optional peer deps: `ioredis`, `pg`, `express`, `fastify`, `hono`, `koa`, `@opentelemetry/api` (all marked optional). Dev/bench: `vitest`, `fast-check`, `@stryker-mutator/core` + `@stryker-mutator/vitest-runner` (new), `tsup`, `biome`, `@bufbuild/buf`, `publint`, `@arethetypeswrong/cli`, plus `redis`/`ioredis`/`pg` for store tests and `express-rate-limit`/`rate-limiter-flexible` for benchmark comparison. |
| **License** | MIT (Copyright (c) 2026 Ameya Borkar) |
| **Last commit** | Local `main`: `1177e1d` — `ci(security): Wave-1 supply-chain hardening — Dependabot + CodeQL + prod-audit gate` (2026-07-01), **not pushed**. GitHub `main`: `867678d` — `chore(release): throttlekit 1.7.0` (2026-07-01, tag `v1.7.0`; last push 2026-07-01). First commit 2026-05-25; 486 commits on local `main` (485 on GitHub), 62 since the June doc. |

---

## What it actually is

ThrottleKit is a framework-agnostic rate-limiting library for Node and the web, built around three cleanly separated concerns (the comment in `src/core/types.ts` states this explicitly):

1. **Strategies** — pure algorithms. A `Strategy<S>` (`src/core/types.ts`) is a deterministic transition `check(state, now, cost) -> StrategyOutcome<S>` with no I/O and no clock reads. Built-ins: `gcra` (default), `tokenBucket`, `fixedWindow`, `slidingWindow`, `slidingWindowLog`, `quota`, `leakyBucket` (`src/algorithms/`).
2. **Stores** — one atomic primitive. A `Store` (`src/core/types.ts`) exposes a single mutating method, `apply<S,R>(key, transform): Promise<R>`, plus an optional synchronous `applySync`. "Adding a backend is implementing one method; adding an algorithm never touches a store."
3. **Adapters** — thin glue to a framework (Express, Fastify, Hono, Koa, Next, Remix, SvelteKit, Elysia, tRPC, NestJS, gRPC, Lambda, Web `fetch`) under `src/adapters/`.

The wiring is done by `rateLimit(...)` in `src/core/limiter.ts`, which builds **one transform** (`src/core/transform.ts`) carrying both the pure JS transition and (when the strategy ships one) the atomic Lua form. An in-memory store runs the JS transition locally; a Redis store collapses it to one round trip — *from the same configuration*. That single shared transform is the technical heart of the "one transform proven bit-identical" claim.

Beyond the core limiter, the package is genuinely large and spans well past basic rate limiting: a two-tier leasing engine (`src/twotier/`, branded **GALE**), a concurrency-limiting tier (`src/concurrency/`), an LLM cost/token-budget admission stack (`src/admission/`, branded **TALE**), cross-region federation (`src/federation/`), a Count-Min-Sketch DDoS limiter (`src/sketch/`), multi-dimensional limits (`src/multi/`), policy planning/replay testkit (`src/policy/`, `src/testkit/`), and observability/OTel taps (`src/observability/`).

Per-directory size gives a sense of the weight: `src/adapters` ~4,106 lines, `src/admission` ~3,662, `src/federation` ~2,524, `src/twotier` ~2,492, `src/concurrency` ~2,159, `src/testkit` ~1,756, `src/algorithms` ~1,650, `src/core` ~847.

**What changed since June (1.5.2 → 1.7.0).** No new engines or subpaths were added. The 62 commits are three release waves plus CI work:
- **1.6.0 (2026-06-23) — correctness.** A multi-agent audit sweep (security / math / logic / performance / concurrency) produced ~20 fixes, each pinned by a regression test that fails on the prior commit. Examples: `MemoryStore` CLOCK eviction gave an expired key a "second chance" and evicted a live neighbour; the Redis OCC fallback shared one connection, so concurrent transactions could lose updates; `multiRateLimit` wrote Redis keys without the store prefix (making `reset()` a silent no-op); the federation and two-tier paths could carry a lease across a window roll; and `slidingWindowLog` charged fractional cost differently on the JS and Lua paths. It also added opt-in options: `RedisStore.ttlFloorMs`, optional `duplicate()` on `RedisClientLike`, `useServerTime` on `RedisConcurrencyCoordinator`, and a sketch `seed`.
- **1.6.1 (2026-06-30) — performance.** A measured sweep that kept every decision byte-identical. `multiRateLimit.checkSync` got ~40% faster (2-dim 4,968→2,968 ns), the `weightedFairEscrow` grant got 15% faster (182→155 ns), and the server `Check` handler got 25% faster (898→675 ns). The `ttlFloorMs` floor is now pipelined into the strategy eval, and the Postgres coordinator dropped a `SELECT … FOR UPDATE`.
- **1.7.0 (2026-07-01) — hardening.** New fuzz harnesses found five issues (F1–F5), and a follow-up sweep found a sixth (F6). The fixes enforce one rule: a strategy that constructs successfully only ever emits finite `Decision` fields. Every strategy now rejects parameters that would overflow its time arithmetic. The worst case was a subnormal `leakyBucket` rate that made `schedule()` call `sleep(Infinity)`. The release also caps YAML nesting at 64, caps `cost` at `MAX_SAFE_INTEGER`, caps `slidingWindow.buckets` at 100k, and rejects poisoned sketch totals.

---

## Where it sits in the ThrottleKit family (core / py / site)

This repo (`throttlekit`) is the **TypeScript core** and the source of truth for the family:

- **`throttlekit` (this repo)** — the TS core library on npm, plus an embedded gRPC service door (`server/`, published separately as `throttlekit-server`, currently `0.4.4`) and a vendored wire protocol (`wire/`) so other-language clients can talk to the same Redis with bit-identical decisions.
- **`throttlekit-py`** — the Python client (the family's "polyglot today" surface). Its contract lives partly *in this repo*: `wire/WIRE-PROTOCOL.md`, `wire/scripts/*.lua`, and `wire/vectors/golden-vectors.json` are the language-neutral conformance vectors a Python `RedisBackend` must satisfy; `docs/design/15-python-client.md` describes it. The gRPC `throttlekit.proto` (`wire/throttlekit.proto`, `server/proto/`) is the recommended polyglot path. **Drift note:** 1.6.0 changed `wire/scripts/slidingWindowLog.check.lua` (fractional cost is now charged as whole units) and moved the golden `rateLimit` ops off `now=0`. `throttlekit-py` 0.5.1 has not re-synced, so its vendored copies of that script and the vectors are now stale (see the py doc).
- **`throttlekit-site`** — the Astro marketing site (`throttlekit.in`). Correction to the June doc: the Astro tree under `research/website/site/` is not a separate design copy. It *is* the local working clone of the `throttlekit-site` repo (its own `.git`, remote `AmeyaBorkar/throttlekit-site`), and the core's `.gitignore` excludes it (`research/website/`). The site's `scripts/sync-design.mjs` reads `../../../../docs/design` because of this nesting.

An infra/ index should link all three. Discrepancy to flag: the README's comparison table says "Polyglot from one verified core (**Python today**)" — consistent with `throttlekit-py` being the live sibling client.

---

## Architecture & how it's structured (annotated tree, package layout, exports)

```text
src/
  index.ts                 # main barrel — the `throttlekit` entry point
  version.ts               # single-sourced version ("1.7.0"); asserted == package.json
  core/                    # the domain model + wiring (the small, proven core)
    types.ts               #   Clock, Decision, Strategy, Store, Transform, ApplyOutcome, LuaProgram, Limiter
    limiter.ts             #   rateLimit() — builds the limiter; sync fast path
    transform.ts           #   decisionTransform / readOnlyTransform — the ONE shared transform
    lua.ts                 #   LUA_NOW preamble + decodeDecision (shared 5-int reply decoder)
    validate.ts            #   requireCost / requireFinite / requireFiniteWindow / requireAtMost (1.7.0 guards)
    combine.ts, key.ts, clock.ts, errors.ts, duration.ts, math.ts
  algorithms/              # pure strategies: gcra, token-bucket, fixed/sliding-window, sliding-window-log,
                           #   quota (calendar-aware), leaky-bucket, calendar
  stores/                  # memory.ts (CLOCK eviction + timing wheel), timing-wheel.ts
  redis/                   # store.ts (EVALSHA + OCC fallback + ttlFloorMs), clients.ts, index.ts
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
test/                      # 188 files, ~37.5k lines, mirroring src by subdir (+ models/, fuzz/ since June)
bench/                     # harnesses + committed run manifests (bench/manifests/)
stryker.config.mjs         # mutation-testing scope + thresholds (new)
research/                  # tracked: design research, eval data, hardening/MUTATION-BACKLOG.md, perf/FOLLOWUPS.md
                           # gitignored/local-only: research/paper/ (arXiv draft) + research/website/ (the site clone)
```

**Package layout / exports.** `package.json` declares **25 functional subpath entry points** plus `./package.json` (26 `exports` keys total), unchanged since June. Each has dual `import`/`require` conditions with separate `.d.ts`/`.d.cts` types — full ESM + CJS, tree-shakeable, `"sideEffects": false`. Subpaths: `.`, `./redis`, `./postgres`, `./dynamodb`, `./deno`, `./cloudflare`, `./express`, `./fetch`, `./hono`, `./next`, `./fastify`, `./koa`, `./grpc`, `./lambda`, `./sveltekit`, `./remix`, `./elysia`, `./trpc`, `./nest`, `./otel`, `./testkit`, `./config`, `./twotier`, `./federation`, `./policy`. There is also a `bin`: `throttlekit` → `./dist/cli.js`. (README and a CI-job comment both still say "24"; the code exposes 25 functional subpaths.)

The main barrel `src/index.ts` re-exports the public surface: `rateLimit`, all strategies, `MemoryStore`, the two-tier family (`twoTier`, `weightedFairEscrow`, `federatedWeightedFairEscrow`, `regionFairPool`, `leaseSizer`, `eoqOptimum`, …), concurrency guards, the admission/TALE family, `multiRateLimit`/`all`/`any`, `sketchRateLimit`/`mergeableSketch`, `buildRateLimitHeaders`, `createEnforcer`, `clientIp`, `withAnalytics`, `tapDecisions`, plus all the types.

---

## Code walkthrough (the important parts) — module by module

### `core/types.ts` — the contracts
Defines `Decision` (`{ allowed, limit, remaining, resetAt, retryAfterMs }`, **all integers and readonly**, with a documented "frozen + evolvable" SemVer stance), `Strategy<S>` (the pure algorithm interface, with optional `lua`, `peek`, `forecast`, `readState`), `Store` (one `apply`, optional `applySync`/`resetSync`/`close`), `Transform<S,R>` (a pure RMW function that optionally carries a `LuaInvocation` for atomic acceleration), `ApplyOutcome` (`{ state, result, ttlMs, persist }`), and `LuaProgram` (`script` + `buildKeys` + `buildArgv`). The comment is explicit that every numeric field is an integer "so the JavaScript and Redis-Lua execution paths can produce bit-identical values (Redis truncates Lua numbers to integers on reply)."

### `core/transform.ts` — the one shared transform
`decisionTransform(strategy, now, cost)` wraps `strategy.check` and, if the strategy ships Lua, attaches a `LuaInvocation` (`program`, `now`, `cost`, `decode: decodeDecision`). `readOnlyTransform(...)` builds a non-mutating transform (`persist: false`) for `peek`/`forecast`. Both the limiter and the two-tier engine call `decisionTransform`, so **a single code path produces both the local and the distributed decision** — this is the structural basis of bit-identicality.

### `core/limiter.ts` — `rateLimit()`
Wires strategy + store + clock + key prefix. Notable: a **reused `syncTransform`** closure with mutable `syncNow`/`syncCost` slots, justified by Node's single-threaded execution (the slots are read before any other call can run through a synchronous `applySync`), so the synchronous hot path allocates no per-call closure. `check()` takes a sync fast path when `store.applySync` exists (resolving an already-computed value as a Promise) and only falls to async `store.apply` + a fresh `decisionTransform` for genuinely async stores. Implements `check`, `checkSync`, `checkMany`, `checkManySync`, `peek(Sync)`, `forecast(Sync)`, `reset`, `close` (closes the store only if the limiter created it).

### `core/validate.ts` — the 1.7.0 boundary guards (new)
Shared validators that every strategy constructor now uses: `requireFinite`, `requireFiniteWindow` (ensures `floor(now/span)*span` stays finite for a realistic clock), `requireAtMost`, and `requireCost`, which now rejects costs above `Number.MAX_SAFE_INTEGER`. The rate-based strategies (`gcra`, `tokenBucket`, `leakyBucket`) also require that their derived interval is a *safe-integer* count of ms, not just finite. The JOURNEY entry notes that a finite but astronomically large emission interval still overflowed GCRA's accumulating TAT after about two requests. `STABILITY.md` gained a matching rule: validation may tighten to reject inputs that "could only misbehave" without counting as a breaking change.

### `stores/memory.ts` — `MemoryStore`
In-process store; atomicity is free because Node is single-threaded, so `applySync` needs no locks. Keeps native values (no JSON). When `maxKeys` is set, bounds cardinality with a **CLOCK (second-chance) eviction** policy over a ring, plus a freed-slot reclaim list, so an adversarial flood of unique keys can't OOM. Since 1.6.0, eviction reclaims an expired-but-unswept entry *before* the ref-bit check. Previously it gave that dead key a second chance and evicted the adjacent live key. Lazy inline expiry off each entry's own `exp`, backed by a `TimingWheel` (`src/stores/timing-wheel.ts`) and an `unref`'d background sweep. `apply` simply resolves `applySync`.

### `redis/store.ts` — `RedisStore`
Structural `RedisClientLike` (ioredis/node-redis satisfy it). Built-in strategies run their atomic Lua via **`EVALSHA` with `EVAL` fallback on `NOSCRIPT`** (a `Map` caches the sha1). `useServerTime` (default true) passes `ARGV[1]=0` so the script reads the Redis `TIME` clock — skew-proof for a shared fleet. Strategies *without* Lua fall back to **optimistic concurrency** (`WATCH`/`GET`/`MULTI`/`EXEC`) with bounded retries (default 5), JSON-encoding state. New since June:
- `ttlFloorMs` floors the physical key TTL independently of the logical window, so Redis real-time GC can't reclaim a live key under a node-clock deployment. For clients with `pipeline()` (ioredis) the floor is coalesced into the strategy eval in one round trip, and it is skipped on read-only `peek`/`forecast`.
- When the client exposes `duplicate()`, the OCC path runs each transaction on its own connection. This fixes a lost-update race when concurrent applies shared one connection.

### `postgres/store.ts` — `PostgresStore`
Distributed store with no Redis. Runs the limiter's **existing pure JS transform** (the very code the in-memory store runs — "there is no Postgres-specific algorithm to keep in sync") inside a transaction serialized per key by a **transaction-scoped advisory lock** (`pg_advisory_xact_lock(hashtextextended(key,0))`). The doc-comment explains *why* an advisory lock, not `SELECT … FOR UPDATE`: `FOR UPDATE` can't lock a not-yet-existing row, so two first-touch transactions could race. State is the same JSON text the Redis OCC path writes, so doubles round-trip exactly. Table identifier is regex-validated (`^[A-Za-z_][A-Za-z0-9_]*(\.…)?$`) since identifiers can't be parameterized. Async-only (no `applySync`), with auto-create + an `unref`'d sweep. (`research/perf/FOLLOWUPS.md` records that a proposed 5-statement → CTE collapse was *declined* because the advisory-lock ordering is load-bearing for the proof.)

### `sketch/index.ts` — Count-Min-Sketch DDoS
A full `CountMinSketch` over a single `Uint32Array` (footprint a function of `epsilon`/`delta` only, never key count), `width = ceil(e/epsilon)`, `depth = ceil(ln(1/delta))` (Cormode & Muthukrishnan 2005), with optional Estan–Varghese **conservative update** (default), saturating arithmetic at `0xffffffff` to preserve the never-underestimate invariant, plus `snapshot`/`toBytes`/`mergeSnapshot` for a cluster-wide `mergeableSketch`.

**Hashing changed in 1.6.0.** Each row now uses its own FNV-1a hash, `fnv1a(key, rowSeed[i]) mod width`, with the per-row seeds derived from a per-instance *random* 32-bit `seed`. This replaces Kirsch–Mitzenmacher double-hashing. The doc-comment explains the security reason: with double-hashing, a key that collides with a victim on two rows collides on every row, so forging a full-column collision cost about `width²` rather than `width^depth`. Merges also got stricter: a counters array of the wrong length now fails closed instead of silently zeroing the tail (1.6.0), and a non-finite or negative `total` is rejected on decode and ignored on merge (1.7.0, finding F1).

`sketchRateLimit` is a fixed-window, **check-before-add** approximate limiter that *never over-admits* (a hard, non-probabilistic guarantee) and only over-denies by `≤ epsilon·N` w.p. `≥ 1-delta`.

### `multi/index.ts` — multi-dimensional checks
`all(dimensions)` / `any(dimensions)` compose named per-axis `Dimension`s (key fn, strategy, optional weight). On a sync store it reads every dimension, decides via a `combine` rule, then commits **all-or-none** in one uninterrupted turn (no partial consume). On Redis it fuses every dimension into a **single Lua round trip** (`MULTI_LUA`) supporting gcra/tokenBucket/fixedWindow, with per-type math mirroring the standalone scripts so JS and Lua composite decisions are bit-identical.

Since June:
- The sync read phase is non-destructive. Before, a `slidingWindow` dimension mutated its ring in place, so it could partially consume on a denied composite.
- Multi keys now go through the store prefix, so `reset()` actually clears them.
- The read phase only `structuredClone`s object or array state, not immutable primitives. This was the source of the 1.6.1 ~40% speed-up.

### `twotier/index.ts` — GALE leasing (see Algorithms)
`twoTier({ strategy, l2, mode, lease, l1 })` with modes `strict` | `cached-deny` | `leased`. The `leased` path keeps per-key local `credits`, coalesces concurrent misses onto one in-flight lease (`pending`), supports `lowWater` proactive refill, `windowCoupled` credit expiry, `returnIdleAfterMs` reclaim, and **adaptive lease sizing** (EOQ learner, `src/twotier/sizing.ts`). Since 1.6.0, a late proactive refill that lands after the window rolled is re-checked against the window that granted it and forfeited. `weightedFairEscrow` gained an O(1) running `Σweight`/`Σused` aggregate in 1.6.1. It is gated on integer weights and costs and falls back to the exact rescan for fractional input, because a float running sum can drift by one ULP and flip a floor. A conformance test with a gate-guard proves the gate is load-bearing.

---

## The algorithms — GCRA, two-tier leasing, DDoS sketches, multi-dimensional

### GCRA (default) — `src/algorithms/gcra.ts`
The Generic Cell Rate Algorithm tracks a single number per key, the **theoretical arrival time (TAT)**. `gcra({ limit, periodMs, burst? })` derives `T = periodMs/limit` (emission interval), `tau = T·burst` (burst tolerance), `inc = T·cost`. `check` computes `tatEff = max(tat, now)` (jump-safe), `newTat = tatEff + inc`, `allowAt = newTat - tau`; admits iff `now >= allowAt`. O(1) time and memory. It also implements `peek` (non-consuming current capacity) and `forecast` (`spendableNow`/`nextReplenishAt`/`fullAt`). The atomic Redis form (`GCRA_LUA`) derives `T`, `tau`, `inc` from **the same integer ARGV** with the same rounding, and stores the TAT via `string.format('%.17g', new_tat)` so it round-trips at full IEEE-754 precision — the in-line justification of bit-identicality. A separate read-only Lua (`GCRA_READ_LUA = GET`) backs `peek`/`forecast` without writing. Since 1.7.0, construction rejects a `limit`/`periodMs` whose `T` is not a positive safe-integer ms, and any `tau = T·burst` that is not finite (findings F4 and F6).

### Two-tier token leasing + the overshoot bound — `src/twotier/` + `spec/`
In `leased` mode each node leases a `batch` of tokens from L2 (Redis/Postgres) in one round trip and serves them locally at in-process speed (steady-state ≈ 1 round trip per `batch` requests). The **formal result** lives in `spec/` (unchanged since June):

- `spec/DistributedLeasing.tla` (+ `.cfg`) — TLA+ model of the default `leased` path (`lowWater=0`, `cost=1`), model-checked with TLC, proving the *tight* per-window bound:
  `admitted_per_window ≤ Limit + |Nodes|·(Batch−1)` — **tightness witnessed by a counterexample trace** (an off-by-one invariant `OvershootTight` is violated reaching `admitted = Limit + N·(Batch−1)`). The README captures TLC's exact run output (config 1: 31 distinct states, depth 10; config 3: 441 distinct states).
- `spec/GaleWindowCoupledLeasing.tla` — the **GALE refinement**: credits expire on the L2 window boundary (the carryover is the *sole* source of cross-window overshoot), collapsing the per-window bound to **exactly `Limit`, independent of fleet size N**. This is the "formally-verified overshoot bound independent of fleet size" of the project brief. In code it is `lease.windowCoupled: true` in `twoTier`, which zeroes `credits` once `now >= lastDecision.resetAt`.
- `spec/GaleFederatedLeasing.tla` + `spec/GaleHeartbeatLeasing.tla` / `GaleHeartbeatHandoff.tla` — the cross-region federation lift (`Nodes → Regions`) proving `admitted ≤ Limit` **independent of region count K**, and heartbeat-based lease handoff (`SCOREBOARD.md` records the handoff spec as TLC-verified at 250,624 states).

Crucially, each `.tla` has a **Java-free JS twin** that enumerates the *identical* transition system with BFS and asserts the same invariant and the same distinct-state counts (`test/twotier/leasing-model.test.ts` reproduces TLC's 31 and 441; `test/gale/leasing-variants.test.ts` covers the window-coupled variant). These run in CI in ~150 ms. So the formal model is *checked twice*: by TLC offline and by a CI-runnable model checker. New since June, `test/models/twotier-lease.commands.test.ts` adds a third, implementation-level check. It is a fast-check `asyncModelRun` that drives several real `twoTier` nodes against one shared `fixedWindow` L2 and asserts that total allowed cost per epoch-aligned window is ≤ `L`.

**Adaptive lease sizing** (`src/twotier/sizing.ts`, GALE "Pillar 2"): an online learner descends onto the EOQ optimum `√(2·orderCost·demand/strandPenalty)` per key, trading coordination against stranding — and, by the Pillar 1 proof, can never loosen the cap.

### DDoS sketches — the fixed-memory structure
A **Count-Min Sketch** (`src/sketch/index.ts`), backed by one `Uint32Array` of `depth × width` counters. The structure is `O((1/epsilon)·ln(1/delta))` and is **independent of the number of distinct keys**, so it tracks an unbounded IP universe in bounded space — the point being that per-key state would itself be the memory-exhaustion vector in a volumetric attack. Conservative-update by default; the rate limiter's only error is in the safe direction (over-deny, never over-admit). Since 1.6.0, rows use independent hashes under a per-instance random seed (see above), so attackers cannot precompute colliding keys offline to grief a victim into false denial.

### Multi-dimensional checks
See `multi/index.ts` above: per-axis combine rules (all-allow → tightest headroom; all-deny → largest retryAfter; any-allow → most remaining; any-deny → soonest recovery), all-or-none commit, fused into one Lua round trip on Redis. `test/models/multi.commands.test.ts` (new) checks at every step that a denied composite changes no dimension's stored state, using byte-equal deep snapshots. `slidingWindow` is deliberately in the strategy pool because it reproduces the original 1.6.0 bug.

---

## Storage backends — memory vs Redis vs Postgres, and the "one transform proven bit-identical" shared core

| Backend | File | Atomicity mechanism | State encoding | Sync? |
|---|---|---|---|---|
| **Memory** | `stores/memory.ts` | Single-threaded RMW (no lock); CLOCK eviction + timing wheel | native values (no serialization) | **yes** (`applySync`) |
| **Redis** | `redis/store.ts` | Atomic Lua `EVALSHA` (EVAL on NOSCRIPT); OCC `WATCH/MULTI/EXEC` fallback for Lua-less strategies, isolated per connection when `duplicate()` exists; optional `ttlFloorMs` | strategy-specific Lua encoding (`%.17g` floats), else JSON | no |
| **Postgres** | `postgres/store.ts` | Per-key transaction-scoped advisory lock; runs the *JS* transform in-transaction | same JSON text as Redis OCC path | no |
| **DynamoDB / Deno KV / Cloudflare** | `dynamodb/`, `deno/`, `cloudflare/` | backend-native conditional/atomic ops | — | no |

The README counts **six stores** (memory, Redis, Postgres, DynamoDB, Deno KV, Cloudflare) — matching the six store-ish exports (`.`/MemoryStore, `./redis`, `./postgres`, `./dynamodb`, `./deno`, `./cloudflare`).

**The "one transform proven bit-identical" shared core.** There is no per-backend re-implementation of any algorithm. Either:
- the store runs the strategy's **vendored Lua** (Redis), which derives every value from the same integer ARGV and clamps identically to the JS `check`; or
- the store runs the strategy's **pure JS `check`** directly inside its atomic primitive (Memory's single-threaded RMW, Postgres's advisory-lock transaction).

Because `Decision` fields are integers and Redis truncates Lua reply numbers to integers, the two execution paths produce equal `Decision` streams. This is *proven*, not asserted: `test/conformance/conformance.test.ts` runs each of ~19 strategy configs through the JS path (`MemoryStore`) and the Redis-Lua path (`RedisStore`) over `40 × 25` randomized `(delta, cost)` timelines and asserts `toEqual` field-for-field (plus a final `peek` equality through the Redis-side HASH/ZSET/string state). `test/conformance/lua-property.test.ts` adds property-based checks, and `test/testkit/replay/cross-store-equivalence.test.ts` replays across stores. The README's "200-way concurrent read-modify-write" appears in `test/postgres/store.test.ts` (a pool sized for a 200-way concurrency test, asserting N concurrent applies land exactly N). Each store suite is still **gated** on its `THROTTLEKIT_TEST_*` env var and calls `describe.skip` without a live store. What changed since June is that CI now sets those vars (see Tests).

One honest footnote from the audit: the dual-path suite had **not** caught a real JS↔Lua divergence. For a fractional `cost`, `slidingWindowLog` charged `ceil` units in JS but effectively `floor` in Lua (zero log entries for `cost < 1`), which over-admitted. 1.6.0 fixed both paths to `ceil` and changed the vendored `slidingWindowLog.check.lua`. The randomized conformance timelines only exercise the costs they generate, so "proven bit-identical" holds for the input space the suite covers.

---

## Formal verification / conformance — what's actually in the repo

This is real, not marketing. Concretely present:

- **5 TLA+ specs** (`spec/*.tla`) with matching TLC configs (`spec/*.cfg`): `DistributedLeasing`, `GaleFederatedLeasing`, `GaleHeartbeatHandoff`, `GaleHeartbeatLeasing`, `GaleWindowCoupledLeasing`. `spec/README.md` documents the models, the exact code correspondence (`src/twotier/index.ts`), captured TLC output (state/distinct counts, tightness counterexample), and how to run TLC. Prose writeup in `docs/FORMAL-MODEL.md`. No specs were added or changed since June.
- **Java-free JS twins** of the specs in `test/twotier/` and `test/gale/` that re-enumerate the same transition systems and assert the same invariants **and the same distinct-state counts** in CI.
- **Golden vectors** — `wire/vectors/golden-vectors.json` (18 suites of language-neutral `(now, cost)` op streams with `expect`ed `Decision` per field) for gcra/tokenBucket/fixedWindow/slidingWindow/slidingWindowLog plus tokenBudget/lease, the contract a polyglot client must satisfy. Now `generatedFrom: throttlekit@1.5.2`, `frozen: false`, `contractVersion: 1`. In 1.6.0 every `rateLimit` op moved off `now=0`, which the Lua scripts read as the "use the server clock" sentinel, so a raw-Lua port can replay the vectors literally.
- **Vendored Lua + manifest** — `wire/scripts/*.lua` (check + read scripts per strategy) pinned by `wire/scripts/manifest.json` (sha256 per script). `test/wire/conformance-scripts.test.ts` re-derives the scripts from the shipped strategy objects and fails if any byte or sha256 drifted, so the wire can't change silently. (`slidingWindowLog.check.lua` changed in 1.6.0 through this regenerate-and-pin flow.)
- **Property tests** — `test/property/invariants.test.ts` and `test/conformance/lua-property.test.ts` use `fast-check`.
- **gRPC contract** — `wire/throttlekit.proto`, `buf.yaml`, and `buf breaking` against a committed `wire/throttlekit.baseline.binpb` baseline (npm scripts `wire:check`/`wire:breaking`). The only proto change since June is a comment clarifying that `AdmitRequest.cost` has no effect on the rate axis.

**New since June — a test-quality layer (`test/models/`, `test/fuzz/`, Stryker):**
- **Mutation testing** (`stryker.config.mjs`, `npm run mutation`, nightly `.github/workflows/mutation.yml`). It is scoped to the decision math: the 8 algorithm files plus `twotier/weighted-fair-escrow.ts`. The first sweep scored 65.4%. A campaign against surviving mutants then raised the aggregate to **77.5%** (1,077 killed / 261 survived / 54 no-coverage / 8 timeout, about 24 min per sweep), with `thresholds.break = 75` as a regression floor. Per-file scores, worst first (`research/hardening/MUTATION-BACKLOG.md`): token-bucket 61.7%, quota 65.5%, gcra 71.8%, fixed-window 74.8%, leaky-bucket 76.2%, weighted-fair-escrow 79.0%, sliding-window 83.5%, sliding-window-log 84.6%, calendar 93.8%. The backlog notes the nightly score is store-less and so understates coverage (leaky-bucket is 76% store-less but ~92% with the Redis conformance). The admission layer (fluid-LP, unified, distributed-budget) is explicitly out of scope ("Phase 2").
- **Model-based tests** — 4 `fc.commands` / `asyncModelRun` suites (10 tests, about 0.9 s, deterministic seeds):
  - atomic strategies, model-vs-implementation and byte-identical store equivalence;
  - `multiRateLimit`: a denied composite changes no state, and reset clears every dimension;
  - `weightedFairEscrow`: Σused ≤ L, conservation, and no cross-window credit smuggling;
  - the two-tier windowCoupled bound.

  According to the commit message, these found no new bugs. The multi detector was checked to go red when the original bug is reintroduced.
- **Fuzz harnesses** — 4 fast-check suites on attacker-controlled boundaries: config text (`parseYaml`/`loadConfig`, including a prototype-pollution check), `X-Forwarded-For`/`clientIp`, `(key, cost)`, and peer sketch bytes. They found F1–F5: poisoned sketch total, YAML recursion stack overflow, huge cost, subnormal limit, and unbounded `buckets`. A follow-up sweep found F6 (subnormal `tokenBucket` refill). All six are fixed and run as regular tests. `test/algorithms/decision-finiteness.test.ts` turns the F4 class into an invariant over every strategy.

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
- **Stores**: `MemoryStore` (core), `RedisStore` (`throttlekit/redis`; new `ttlFloorMs` option), `PostgresStore` (`throttlekit/postgres`), plus DynamoDB / Deno / Cloudflare subpaths.
- **Distributed (GALE)**: `twoTier`, `weightedFairEscrow`, `federatedWeightedFairEscrow`, `regionFairPool`, `RedisRegionFairPool`, `LeaseSpender`, `leaseSizer`/`predictiveLeaseSizer`/`eoqOptimum`.
- **Concurrency**: `adaptiveConcurrency`, `distributedAdaptiveConcurrency`, `Redis/Postgres/TestConcurrencyCoordinator` (Redis coordinator gains `useServerTime`, default true).
- **Admission (TALE)**: `tokenBudget`, `distributedTokenBudget`, `unifiedAdmission`, `fairShare`/`weightedFairShare`/`weightedMaxMin`, `adaptiveThrottle`, `solveFluidLp`, `FusedDispatcher`, `FUSED_GCRA_TOKEN_BUCKET_LUA`, …
- **Multi-dimensional**: `all`, `any`, `multiRateLimit`.
- **DDoS**: `sketchRateLimit`, `mergeableSketch`, `sketchSnapshotFromBytes` (new optional `seed`).
- **HTTP/adapters**: `buildRateLimitHeaders`, `createEnforcer`, `clientIp`, `hashKey`/`hmacKeyer`, plus framework subpaths.
- **Observability**: `tapDecisions`, `withAnalytics`, `throttlekit/otel`.
- **CLI**: `throttlekit` bin.

The 1.x surface is unchanged in shape. 1.6.0 added only opt-in options. 1.7.0 adds new `RangeError`s, but only for absurd values that previously produced corrupt output, which is why it shipped as a minor rather than a patch.

---

## Tech stack & dependencies

- **Language**: TypeScript (ESM source, `"type": "module"`, `engines.node >= 18`), targeting ESM + CJS dual output.
- **Build**: `tsup` (`tsup.config.ts`) → `dist/` with `.js`/`.cjs` + `.d.ts`/`.d.cts`.
- **Test**: `vitest` 4 (`vitest.config.ts`), with `@vitest/coverage-v8` and `fast-check` for property, model-based and fuzz tests; **Stryker** 9.6 (`@stryker-mutator/core` + `vitest-runner`) for mutation testing.
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
npm run mutation     # stryker run (scoped to the decision math)        ← new
npm run check        # lint + typecheck + test + check:package
npm run check:package# build + publint + attw  (package-correctness gate)

# benchmarks
npm run bench        # tsx bench/run.ts
npm run bench:compare# head-to-head vs rate-limiter-flexible / express-rate-limit
npm run bench:gate   # perf regression gate (bench:baseline to rebaseline)
npm run bench:lease  # Tier-2 lease spend
npm run bench:fleet  # server/bench/fleet.ts
npm run bench:check  # server/bench/check.ts — Check handler CPU in isolation   ← new

# wire / polyglot contract
npm run wire         # regenerate vectors + scripts
npm run wire:check   # buf lint + buf breaking against the committed baseline

# publish (npm)
# prepublishOnly runs `npm run build`; package `files`: dist, README, CHANGELOG, STABILITY, BENCH, LICENSE
```

TLA+ model-checking is run out-of-band with TLC (`tla2tools.jar`, instructions in `spec/README.md`); the CI-safe equivalent is the JS twins under `npm test`.

**CI (`.github/workflows/`, as on GitHub at `v1.7.0`):** `ci.yml` has 7 jobs:
- lint + typecheck
- package surface (`publint` + `attw`)
- wire (`buf lint` + `buf breaking`)
- test on Node 20/22/24 with a Redis service
- **live-store gate** (new): Redis + Postgres + `amazon/dynamodb-local` service containers, running the serialized core store suite plus the server's live-backend gRPC e2e
- a blocking machine-independent bench regression gate (now also covers the multi-combine and WFE rows)
- the server build/test

The other workflows are `mutation.yml` (nightly Stryker, new), `release.yml`, `release-server.yml` and `gh-packages.yml`. CI was green on all 9 job runs for the `v1.7.0` push. Three more files exist **only in the unpushed local commit `1177e1d`**: `dependabot.yml` (weekly grouped npm/server/Actions updates), `codeql.yml` (`security-extended` JS/TS analysis) and `supply-chain.yml` (`npm audit --omit=dev --audit-level=high` for core and server). They are not active on GitHub.

---

## Tests

- **188 files / ~37,550 lines** under `test/`, mirroring `src` by subdirectory. By file count: `test/adapters` 22, `test/admission` 19, `test/gale` 18 (incl. helper modules), `test/twotier` 14, `test/concurrency` 14, `test/federation` 13, `test/algorithms` 11, `test/core` 9, `test/cost` 8, `test/redis` 7, plus the new `test/models` (4) and `test/fuzz` (4). About 1,470 `it`/`test` call sites statically.
- **Executed counts (from commit messages; not re-run here):** 1,551 storeless tests green at 1.7.0. The live-store gate validated 1,651 core and 323 server tests against real containers.
- **Dual-path conformance**: `test/conformance/conformance.test.ts` (JS vs Redis Lua over thousands of generated ops) + `lua-property.test.ts`. Gated on `THROTTLEKIT_TEST_REDIS`; runs in the CI test matrix and the live-store job.
- **Postgres conformance**: `test/postgres/store.test.ts` incl. the 200-way concurrency test. Gated on `THROTTLEKIT_TEST_POSTGRES`. It now **runs in CI** through the live-store job, which it did not in June.
- **DynamoDB**: the core `test/dynamodb/store.test.ts` uses an in-memory fake client. A real DynamoDB (dynamodb-local) is exercised only through the server's `runtime.dynamodb.e2e.test.ts` in the live-store job. Deno KV and Cloudflare are tested against fakes.
- **Formal-model twins**: `test/twotier/leasing-model.test.ts`, `test/gale/leasing-variants.test.ts` (BFS state enumeration matching TLC counts).
- **Property / model-based / fuzz**: `fast-check` in `test/property/`, `test/conformance/`, `test/models/` (4 stateful suites) and `test/fuzz/` (4 boundary harnesses). There is also a decision-finiteness invariant test in `test/algorithms/`.
- **Mutation**: 77.5% aggregate on the 9 math files, with a regression floor of 75 (see above). On GitHub the nightly job ran **61 times between 2026-07-02 and 2026-08-31: 60 successes, 1 cancelled, 0 failures**. Because the job fails when the score drops below the floor, those successes mean the score stayed ≥ 75%. After that GitHub auto-disabled the schedule for inactivity (workflow state `disabled_inactivity`).
- **Wire conformance**: `test/wire/conformance-scripts.test.ts` (script bytes + manifest sha256 can't drift silently).
- **Version sync**: `test/version-sync.test.ts` asserts `src/version.ts` == `package.json#version`.
- `docker-compose.stores.yml` spins up Redis/Postgres locally for the gated suites; `coverage/` and `cov.log` exist on disk but are gitignored.

---

## Status, completeness & notable gaps

- **Mature and published.** v1.7.0 on npm, 486 commits, and a 128 KB `CHANGELOG.md`. The repo also carries `STABILITY.md` (SemVer policy, now with the "validation may tighten" rule), `SECURITY.md`, `BENCH.md`, `SCOREBOARD.md`, `THROTTLEKIT.md` (48 KB design doc), a 697-line `JOURNEY.md` build log, and 18 component design docs under `docs/design/`. This is a polished, heavily-documented library, not a prototype.
- **Since June the work has been depth, not breadth.** No new engines, subpaths or TLA+ specs. Instead there was a correctness audit (1.6.0), a measured performance sweep with decisions kept byte-identical (1.6.1), and a hardening pass (1.7.0) built on mutation, model-based and fuzz testing. The live-store CI gate closes a gap the June doc flagged: the Postgres and cross-store suites now execute in CI rather than only locally.
- **Genuinely broad.** The surface goes far past "rate limiting": concurrency limiting, LLM token-budget escrow (TALE), federation, policy plans, replay testkit, Lens monitoring. Much of this is real, tested code; some pieces are explicitly experimental.
- **Experimental / non-frozen areas (self-flagged in code)**: `sketchRateLimit`/`mergeableSketch` (`@experimental`, outside 1.x SemVer); the raw Redis wire (`frozen: false`, recommended polyglot path is the gRPC door); the cluster mergeable sketch is an estimator, not a hard global limiter.
- **Dormant since 2026-07-01.** There have been no commits or pushes for over three months. As a result, GitHub has disabled the nightly mutation workflow, and the supply-chain commit (Dependabot, CodeQL, prod-audit) is still unpushed. `1177e1d`'s own message also defers OpenSSF Scorecard, SBOM-on-release and SHA-pinning of actions.
- **Mutation backlog is open.** `token-bucket` (62%) and `quota` (66%) are the next targets, and the admission layer is not mutated at all.
- **Local-only artifacts (correction to the June doc).** `dist/`, `coverage/`, `node_modules/`, `cov.log`, `research/paper/` and `research/website/` (the site clone) are on disk but **gitignored**, so they do not appear in the GitHub repo.
- **TLC is out-of-band.** The TLA+ proofs need an external TLC jar; the CI-runnable JS twins cover the same invariants.
- **In-progress markers.** Some `src/twotier/index.ts` doc-comments still carry "next commit"-style notes (e.g. "the multi-process L2 backing arrives in the next commit") from the GALE/federation build-out.

---

## README vs. code

The README is unusually accurate for a project this ambitious, and most headline claims check out against the source. It has not been edited since June, so it neither claims nor mentions any of the 1.6–1.7 work. Verified-true:
- zero runtime dependencies (no `dependencies` block);
- the TLA+/TLC overshoot bound `Limit + N·(Batch−1)`, tight, with `windowCoupled` giving `Limit` independent of fleet size (`spec/`, twins in `test/`);
- one algorithm proven bit-identical across backends (`core/transform.ts` + `test/conformance/`);
- the 200-way concurrent RMW (`test/postgres/store.test.ts`);
- MIT license, Node ≥ 18, and ESM + CJS + types.

Minor discrepancies / things to read critically:
- **Entry-point count.** README says "24 entry points" (as does a comment in `ci.yml`); `package.json` declares **25** functional subpaths (26 `exports` keys incl. `./package.json`). Still an undercount.
- **Benchmark figures have drifted from `BENCH.md`.** The README's numbers date from the 1.0 `BENCH.md` (2026-05-31): 169 ns `checkSync`, 66.4k ops/s leased over Redis, async GCRA "301 vs 331 ns" vs rate-limiter-flexible, and a "~35×" Postgres lever. The committed `BENCH.md` (machine table: 1.4.0, 2026-06-10) shows something different:
  - 170 ns (~1 B/op)
  - 71.2k ops/s leased vs 735 strict (~97×)
  - 295 vs 337 ns async
  - 118 vs 342 ops/s on a raw Postgres counter, with leased Postgres at 9.4k ops/s

  The 1.6.1 sweep re-measured the hot path as flat: gcra `checkSync` at 176 → 173 ns, median of 3 (`JOURNEY.md`). So "~170 ns" is well supported, but the README's exact digits are stale. The marketing site resynced to `BENCH.md` on 2026-07-02 and is now *more* current than the README.
- **Perf-sweep table provenance.** `BENCH.md` says every table traces to a committed manifest. However, the only manifest added in the sweep (`bench/manifests/check-2026-06-29…json`) records the server `Check` handler at **971 ns**, which is a pre-optimization capture. The before/after figures in the table (898 → 675 ns, plus the multi-combine and WFE rows) have no committed manifest behind them.
- **"Six stores, proven bit-identical."** The rigorous dual-path proof covers **Memory ↔ Redis** directly. Postgres is covered by the shared transform plus the concurrency and cross-store tests, which now run in CI. DynamoDB is exercised live only through the server e2e, and Deno KV and Cloudflare against fakes. Their bit-identicality comes from construction (they run the same JS transform). The 1.6.0 `slidingWindowLog` fractional-cost JS↔Lua bug is a reminder that "proven" means "proven over the inputs the suite generates."
- **Strategy count.** README says "8 strategies"; the barrel exports 7 named strategies. The count reaches 8 only if `sketchRateLimit` or quota's cadences are counted separately.
- **Branding.** "GALE" (leasing) and "TALE" (token budget) are README brand names; in code they map to `src/twotier/` and `src/admission/` respectively — no `GALE`/`TALE` literal modules, just the features they describe.
- **Wire freeze.** README leans on "verified core / polyglot from one verified core"; the code is careful to note the raw Redis wire is **not frozen yet** (`wire/WIRE-PROTOCOL.md`, `manifest.json frozen:false`) and the supported polyglot path is the gRPC door. The 1.6.0 script change shows why that caveat matters: it left the Python client's vendored copy behind.
- **Under-claims.** The README does not mention mutation testing, model-based tests, fuzzing or the live-store gate, which are some of the strongest verification evidence added since June.

Net: the README still oversells nothing structurally. The proofs, the shared transform and the conformance suite are all genuinely in the repo, now backed by mutation, model and fuzz layers. The gaps are stale numbers (entry points and benchmark digits) and the usual "counted vs. rigorously-proven" distinction across the six backends.

---

*Doc generated 2026-06-16, refreshed 2026-10-12 by reading the source at C:\Users\ameya\Documents\GreenfeildProject. Source of truth: https://github.com/AmeyaBorkar/throttlekit*
