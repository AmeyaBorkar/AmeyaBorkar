# infra — the ThrottleKit family

ThrottleKit is **one product across three repositories** (plus a fourth component that lives inside the core repo). The unifying design goal: rate-limiting whose decision is **bit-identical** no matter where it runs — in-memory, Redis (Lua), Postgres, the Node core over gRPC, or the Python client — proven by shared golden vectors and TLA+ specs.

```
                 ┌─────────────────────────────┐
                 │  throttlekit (TS core)       │  v1.5.2 on npm, zero deps
                 │  GCRA · two-tier leasing ·   │  decisionTransform() is the
                 │  Count-Min DDoS sketch ·     │  single source of truth
                 │  5 TLA+ specs + JS twins     │
                 │   └─ server/ = throttlekit-  │  ← 4th component (gRPC door),
                 │      server (gRPC door)      │     not its own repo
                 └───────────┬─────────────────┘
              wire/ contract │ (golden vectors + vendored Lua + .proto)
              ┌──────────────┴───────────────┐
   ┌──────────▼──────────┐        ┌──────────▼───────────┐
   │ throttlekit-py      │        │ throttlekit-site     │
   │ PyPI 0.5.0          │        │ Astro 6 marketing +  │
   │ gRPC + direct-Redis │        │ docs site            │
   │ (sha256-pinned Lua) │        │ (content collections)│
   └─────────────────────┘        └──────────────────────┘
```

Each linked doc below was written by **reading the actual source**, with computed line counts and a "README vs. code" section.

| Repo | Role | Stack / size | Notable code-level finding |
|------|------|--------------|----------------------------|
| [throttlekit](throttlekit.md) | **TS core** — the algorithms and the canonical decision transform | TypeScript · 120 src files / 24,502 LOC · npm `throttlekit` 1.5.2 | The "one transform, bit-identical" claim is real: `core/transform.ts` is shared by limiter + two-tier; Memory/Postgres run the pure-JS path, Redis runs vendored Lua over the same integer ARGV, **conformance-tested** JS-vs-Lua. DDoS sketch is a Count-Min Sketch w/ conservative update; the fleet-size-independent overshoot bound is a genuine **TLA+** refinement (5 specs + JS twins in CI). |
| [throttlekit-py](throttlekit-py.md) | **Python client** — same decisions from Python | Python · 23 src / 2,617 LOC · PyPI `throttlekit-py` 0.5.0 | Two transports (gRPC `ServiceBackend`, direct-Redis with **sha256-pinned vendored Lua**) plus tier-2 fleet leasing, async twins, and FastAPI/Django/Flask adapters. Bit-identical is *proven* by replaying the core's golden vectors through real Redis. `Monitor.Watch` streaming is defined in the proto but unimplemented in Python. |
| [throttlekit-site](throttlekit-site.md) | **Marketing + docs site** | Astro 6 · ~6,050 code LOC (+~5 MB assets) · 3 deps | Static site, no Tailwind, no UI framework. Design docs are synced from the core monorepo via `sync-design.mjs` (a staleness risk). No live rate-limit playground — the only interactivity is a cosmetic scroll-scrub canvas hero. **No in-repo Vercel config** despite the "hosted on Vercel" claim; canonical origin is `throttlekit.in`. |
