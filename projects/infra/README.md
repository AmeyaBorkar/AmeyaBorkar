# infra — the ThrottleKit family

ThrottleKit is **one product across three repositories** (plus a fourth component that lives inside the core repo). The unifying design goal: rate-limiting whose decision is **bit-identical** no matter where it runs — in-memory, Redis (Lua), Postgres, the Node core over gRPC, or the Python client — proven by shared golden vectors and TLA+ specs, and (since July) stress-tested by mutation, model-based and fuzz suites.

```
                 ┌─────────────────────────────┐
                 │  throttlekit (TS core)       │  v1.7.0 on npm, zero deps
                 │  GCRA · two-tier leasing ·   │  decisionTransform() is the
                 │  Count-Min DDoS sketch ·     │  single source of truth
                 │  5 TLA+ specs + JS twins ·   │
                 │  Stryker 77.5% · fuzz ·      │
                 │  live-store CI               │
                 │   └─ server/ = throttlekit-  │  ← 4th component (gRPC door),
                 │      server 0.4.4 (gRPC)     │     not its own repo
                 └───────────┬─────────────────┘
              wire/ contract │ (golden vectors + vendored Lua + .proto)
              ┌──────────────┴───────────────┐
   ┌──────────▼──────────┐        ┌──────────▼───────────┐
   │ throttlekit-py      │        │ throttlekit-site     │
   │ PyPI 0.5.1          │        │ Astro 6 marketing +  │
   │ gRPC + direct-Redis │        │ docs site, light/dark│
   │ (sha256-pinned Lua, │        │ (content collections)│
   │  synced ≤ core 1.5) │        │                      │
   └─────────────────────┘        └──────────────────────┘
```

Each linked doc below was written by **reading the actual source**, with computed line counts and a "README vs. code" section. Refreshed 2026-10-12: every repo has been quiet since early July 2026. The core's last release was 1.7.0 on 2026-07-01, py's was 0.5.1 on 06-30, and the site's last push was on 07-02.

| Repo | Role | Stack / size | Notable code-level finding |
|------|------|--------------|----------------------------|
| [throttlekit](throttlekit.md) | **TS core** — the algorithms and the canonical decision transform | TypeScript · 120 src files / 25,255 LOC (+ 37.5k test LOC) · npm `throttlekit` 1.7.0, `throttlekit-server` 0.4.4 | The "one transform, bit-identical" claim is real: `core/transform.ts` is shared by limiter + two-tier; Memory/Postgres run the pure-JS path, Redis runs vendored Lua over the same integer ARGV, **conformance-tested** JS-vs-Lua, and since July the store suites run against live Redis/Postgres/DynamoDB in CI. The fleet-size-independent overshoot bound is a genuine **TLA+** refinement (5 specs + JS twins). 1.6–1.7 added ~20 audit fixes, a byte-identical perf sweep (multi-check −40%) and a fuzz-driven hardening pass (non-finite decisions now rejected at construction), with Stryker mutation scoring **77.5%** on the decision math. Caveat: the Dependabot/CodeQL commit is unpushed, and GitHub has auto-disabled the nightly mutation job for inactivity. |
| [throttlekit-py](throttlekit-py.md) | **Python client** — same decisions from Python | Python · 23 src / 2,663 LOC · PyPI `throttlekit-py` 0.5.1 | Two transports (gRPC `ServiceBackend`, direct-Redis with **sha256-pinned vendored Lua**) plus tier-2 fleet leasing, async twins, and FastAPI/Django/Flask adapters. Bit-identical is *proven* by replaying the core's golden vectors through real Redis. **New drift:** the vendored `slidingWindowLog.check.lua` and golden vectors predate core 1.6.0's wire change, and the drift gate only checks the files against this repo's own manifests, so CI stays green. `Monitor.Watch` streaming is still unimplemented. |
| [throttlekit-site](throttlekit-site.md) | **Marketing + docs site** | Astro 6 · ~6,530 code LOC (+~5 MB assets) · 3 deps | Static site, no Tailwind, no UI framework; now has a pre-paint light/dark theme. Versions, changelog and benchmark numbers are current, and its benchmark figures are now *more* up to date than the core README's. The 18 design docs were edited site-side and no longer byte-match the core's, so `sync-design.mjs` would overwrite those edits. There is still no live playground and no in-repo Vercel config. The canonical origin is `throttlekit.in`. |
