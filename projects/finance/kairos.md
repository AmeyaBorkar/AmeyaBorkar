# kairos

> A payment-recovery system for Indian merchants, built around **bounds it cannot exceed**. It watches payment outcomes per "rail" (method × bank × app/network), detects a degrading rail with a bank of Page's CUSUM statistics, steers customers off it by rewriting Razorpay Checkout's `config.display` (with a 10% randomized holdout as a control group), then works the failed payments ("casualties"): it classifies why each one failed, prices every possible action with an expected-value gate, and retries on the rail's *recovery edge* rather than a fixed +1h/+24h/+72h ladder. Every action, including each steer, retry, message and LLM call, has to clear **Terminus**, a governance kernel that admits work against an HMAC-signed mandate using atomic commitment accounting on Ameya's own [ThrottleKit](../infra/throttlekit.md) store. Every decision, refusals included, goes to a hash-chained audit ledger. The numbers come from a seeded, offline benchmark harness that runs against **simulated** traffic. Outbound charges and messages are **dry-run only**. Built solo for the Razorpay AI Buildathon, Track 03 (AI Revenue Recovery).

| | |
| --- | --- |
| **Repository** | https://github.com/AmeyaBorkar/kairos |
| **Visibility** | Public (0 stars at time of writing) |
| **Category** | finance |
| **Primary language(s)** | TypeScript (strict, ESM, NodeNext), plus vanilla HTML/CSS for three static UIs and a few `.mjs` scripts |
| **Local path** | `C:\Users\ameya\Documents\razorpay\kairos` |
| **Default branch** | `main` |
| **Lines of code (computed)** | **30,935** source TS + **15,772** test TS = **46,707 LOC of TypeScript** (+48-line `vitest.config.ts`). By layer (src / test): `packages/` 11,412 / 8,550 · `adapters/` 5,047 / 3,335 · `apps/` 14,476 / 3,887. The benchmark harness `apps/bench` is the largest single package at 5,405 lines, then the site at 3,459. Also 1,369 lines of JS/MJS scripts, 2,648 CSS, 1,881 HTML, and ~4,092 lines of Markdown docs. Counted over `git ls-files`, so `node_modules`, local `dist/`, `pnpm-lock.yaml`, the committed data/results JSON (e.g. `copy-library.json`, 3,755 lines) and the 41.6 MB film are all excluded. |
| **Source files (computed)** | 179 non-test `.ts` modules + 71 `.test.ts` files = 250 TS files across **21 private workspace packages** (9 `packages/`, 4 `adapters/`, 8 `apps/`); **~1,184 test cases** (`it`/`test` call sites, 9 of them `.each`); 389 tracked files in total |
| **Package / host platform** | pnpm workspace monorepo (`pnpm@11.15.1`, Node ≥ 22, pnpm ≥ 11). Nothing is published to npm; every package is `private`. Deployables: a Docker image (`node:22-alpine`, defaults to the sentry), a `docker-compose.yml` stack (Postgres 18 + sentry + worker + traffic + console), and the static site on **GitHub Pages at https://kairosrecovery.site** (custom-domain `CNAME`). **Live: HTTP 200 on 2026-10-12**, last deployed 2026-09-05; `llms.txt` and the 41,638,235-byte film are served. |
| **Key dependencies** | Runtime: `throttlekit` ^1.7.0 (Ameya's own library: atomic `Store.apply`, rolling quotas, reservation sizers, `PostgresStore`), `fastify` ^5.2, `pg` ^8.23, `zod` ^4.1. `@kairos/domain` has no dependencies; `@kairos/ledger` uses only `node:crypto`. Dev: `typescript` ^7.0.2, `vitest` ^4.1.11 (+ `coverage-v8`), `@biomejs/biome` ^2.5, `@electric-sql/pglite` ^0.5.8 (in-process Postgres for tests), commitlint. External: Razorpay test-mode API, and Google Gemini (`gemini-3.1-flash-lite`) used only at build time and for optional `explain` prose. |
| **License** | MIT © Ameya Borkar |
| **Last commit** | `f16c31e`: "docs: the fixed ladder no longer out-recovers Kairos, and four files said it did" (2026-09-05). First commit `e95c5cb` "chore: initialise repository" (2026-08-25). **153 commits** on `main` over six working days (Aug 25–27, Sep 3–5), all conventional commits (71 feat, 24 docs, 20 fix, 10 refactor, 10 chore, 8 ci, 5 build, 4 test, plus 1 web-UI "Create CNAME"). Five Dependabot branches are open and unmerged. |

---

## What it actually is (be concrete)

### Context: what it was built for

Ameya built Kairos alone, in roughly 12 days (2026-08-24 to 09-05), for the **Razorpay AI Buildathon**. That is a build-first hiring programme for in-person internships, where candidates ship a project instead of sending a CV. The entry was **Track 03, "AI Revenue Recovery"**, whose bar reads: *"Show measured money recovered across a batch, with compliant escalation, stopping rules, and an audit trail."* The submission form asked for the public repo, a pitch-video link (`https://kairosrecovery.site/#film`) and the architecture (`docs/ARCHITECTURE.md`). Three files in the parent folder (`C:\Users\ameya\Documents\razorpay\`) record how it was planned and shipped, and none of them is in the repo: a submission checklist, a postmortem, and a pre-build memo, *"Two Products, One Choice"*. The memo compared two candidate ideas: **Kavach** (catch a payment-rail outage, steer around it, recover the casualties) and **Rekha** (a spending limit for AI agents that holds under concurrency, using commitment accounting). Kairos ended up as Kavach's product built on Rekha's machinery: Terminus *is* the commitment-accounting kernel.

### What the code does

There are three operational arms and one measurement arm, and all of them share a pure core:

1. **Detect** (`packages/detect`). Per-slice failure-rate monitoring over a slice tree (`upi` → `upi|hdfc` → `upi|hdfc|phonepe`). Each slice runs four parallel CUSUMs (shift sizes δ = 0.03 / 0.08 / 0.18 / 0.40) against an empirically-shrunk baseline, inside a `quiet → alarmed → clearing` state machine. Incidents roll up to the coarsest slice that explains them. A separate *recovery* statistic closes incidents (see below).
2. **Prevent / steer** (`packages/policy`, `apps/sentry`). For each open incident it prices the candidate steers: either **suppress** the failing instrument (`config.display.hide`, which only works for slices Checkout can name, such as card BIN/network, a netbanking bank or a wallet) or **demote** a whole method in the display sequence (the only lever for UPI, because Checkout cannot see a UPI payment's issuer). A steer is only made if its net expected change in failure probability across *all* traffic is negative, and it is often declined. Each steer is a Terminus grant. A 10% sticky per-(customer, incident) holdout keeps a control arm. `POST /plan` always answers HTTP 200 and falls back to the merchant's own ordering.
3. **Recover** (`packages/recover`, `apps/recover-worker`). Failed payments become casualties. A 47-rule table, keyed on Razorpay's `error_source`/`step`/`reason` triple, classifies each one into `transient | timed | customer-action | customer-retry | dead | unknown`. A hierarchical Beta-Binomial model predicts the chance that an action recovers the payment, and an expected-value gate subtracts both postage and the *priced cost of an opt-out*. Scheduling waits through a "spontaneous window" (default 45 min), then a rail-recovery edge, "balance-likely" days of the month (1st, 2nd, 7th, last), or a bounded ladder. A 10% recovery holdout makes every reported figure *incremental*.
4. **Govern** (`packages/terminus`). One `admit()` method is the only path to spending money or contacting a person. It checks 11 named binding axes in a fixed order and never throws. Allowed and refused decisions are both written to the `packages/ledger` hash chain.
5. **Measure** (`apps/bench`, `packages/proof`). Seeded experiments for detection, spend under concurrency, steering lift and recovery. Results are consolidated into a scorecard of **19 exact invariants and 27 banded metrics**, and a CI job gates the scorecard against a committed baseline.

**On the "AI" in it:** most of it is statistics, not a language model. As `llms.txt` itself puts it, detection is CUSUM and recovery probability is Beta-Binomial. An LLM (Gemini) appears in two places only. It wrote the **copy library** once, at build time: 468 message variants over 180 situations (5 failure classes × rail × 4 languages × 3 channels), 190 calls, ₹8.15 at list price, committed to `data/copy-library.json` and replayed offline by every test. It also optionally writes prose for `explain`. No inference runs on the money path.

## Architecture & how it's structured

A pnpm workspace with a pure core (`packages/`), ports and adapters (`adapters/`) and runnable apps (`apps/`). Every package builds with plain `tsc -p tsconfig.build.json`; there is no bundler. Annotated tree (`node_modules`/`dist` omitted):

```
kairos/
├─ package.json             root "kairos" (private): build · test · bench:* · demo · razorpay:probe · check
├─ pnpm-workspace.yaml      packages/* · adapters/* · apps/*
├─ vitest.config.ts         tests run against src/ via aliases (no build needed); coverage gate 85/85/80/85
├─ tsconfig.base.json       strict + noUncheckedIndexedAccess + exactOptionalPropertyTypes, NodeNext, ES2023
├─ biome.json · commitlint.config.js · Dockerfile · docker-compose.yml (db, sentry, worker, traffic, console)
├─ scripts/                 demo.mjs (Compose wrapper; mints a signing key into .env.demo) · check-deps.mjs (workspace-graph lint)
├─ data/copy-library.json   468 Gemini-written variants, 180 segments, provenance {model, promptHash, calls 190, spentPaise 815}
├─ docs/
│  ├─ ARCHITECTURE.md       1,194 lines: thesis, principles, modules, detection, steering, recovery, Terminus, failure modes, open questions
│  ├─ MEASUREMENT.md        1,152 lines: every published number, phase by phase, with caveats
│  ├─ decisions/0001–0010   ten ADRs (commitment accounting, two levers, 6th recoverability class, lease column, two clocks…)
│  ├─ results/              blessed output: scorecard-full.{txt,json}, baseline-quick.json (the gate), recovery.{txt,json}, variance-quick.*
│  └─ razorpay-probe.json   transcript of 3 live test-mode API calls (2026-09-03)
├─ packages/                pure core: no I/O, no clock reads
│  ├─ domain/      Paise/branded ids, Slice tree, Attempt, Casualty, Mandate, 8 ActionKinds, 11 BindingAxes, 5 StopReasons, stableDraw/holdout, SMS segment math
│  ├─ detect/      cusum.ts (bank + recovery statistic) · baseline.ts (EB shrinkage) · detector.ts (FSM) · engine.ts (slice tree, rollup)
│  ├─ policy/      evaluate.ts (price a steer) · addressability.ts · controller.ts (steer = Terminus grant) · holdout.ts · plan.ts · window.ts
│  ├─ recover/     classify.ts (47 rules) · probability.ts · decide.ts (EV gate) · schedule.ts · worker.ts · calibration.ts · residual.ts (LLM port)
│  ├─ terminus/    kernel.ts (admit/settle/abandon) · budget.ts (pure commitment accounting) · store.ts (atomic apply) · caps.ts ·
│  │               stops.ts · signature.ts (HMAC-SHA256 mandates) · stop-switch.ts (out-of-band kill) · reservation.ts (sizers)
│  ├─ ledger/      chain.ts (SHA-256 hash chain) · canonical.ts (canonical JSON) · memory.ts (MemoryLedger, FailingLedger)
│  ├─ proof/       metric · invariant · variance · compare · scorecard · report: the regression gate
│  ├─ reason/      prompt · gauntlet (copy validator) · honesty (figure check) · library · source · segment · variant · price
│  └─ explain/     retrieve.ts + explain.ts: audit chain → redacted answer
├─ adapters/
│  ├─ simulator/       seeded Indian traffic (INDIA_PROFILES, ~25 slices), injectable degradations, customer choice, recovery world
│  ├─ razorpay/        client.ts (fetch, retries, whole-call deadline) · checkout.ts (config.display) · inbound.ts + webhook.ts
│  │                   (raw-body HMAC, replay window, required pseudonymiser) · codes.ts · copy.ts (English templates) · scripts/probe.mjs
│  ├─ reasoner-gemini/ transport (reads google.rpc.RetryInfo, not Retry-After) · composer/classifier/explainer · cassette replay
│  └─ postgres/        casualty queue with a lease column (ADR 0008), schema printer
└─ apps/
   ├─ bench/           measurement harness (5,405 lines): detect curve, spend bound, prevent, recover (5 arms), scorecard/gate/bless/variance
   ├─ sentry/          Fastify: POST /plan · POST /outcomes · POST /webhooks/razorpay · GET / (operator view) · /health · /ledger · /metrics
   ├─ recover-worker/  drains casualties under a mandate; dry-run delivery only; memory or Postgres backing; /health · /ready · /metrics
   ├─ console/         Fastify API over the real components + 6 scenarios; `explain` CLI
   ├─ mandate/         `kairos-mandate` bin: form · seal · explain · verify · status · stop · resume
   ├─ scribe/          build-time copy-library generator, itself spending under a signed mandate
   ├─ traffic/         "a merchant that is not there": demo traffic generator that asks /plan and files casualties
   └─ site/            kairosrecovery.site: static, hash-routed (6 views), canvas pixel-art scenes, recorded console run, film
```

**Entry points:** sentry `apps/sentry/src/main.ts` (the Docker `CMD`) · worker `apps/recover-worker/src/main.ts` · console `apps/console/src/main.ts` · CLI `apps/mandate/src/main.ts` · benches `apps/bench/src/*-main.ts` · site `apps/site/src/main.ts`.

## Code walkthrough (the important parts)

**`terminus/kernel.ts`: `Terminus.admit()`, the single gate.** The checks run in this order: mandate HMAC signature → the mandate's own `killSwitch` → the out-of-band kill switch (**if it can't be read, it is treated as engaged**) → validity window → `allowedActions` → stop rules (`recovered > opted-out > disputed > dead-class > 3 hard declines`) → Terminus's own EV floor (`p × EV > estimatedCost`) → quiet hours (contacts only) → a *non-consuming* peek at the contact cap → atomic budget/in-flight reservation → atomic contact-cap consume (if another worker took the last allowance in between, the budget reservation is released) → audit append. If the audit record can't be written, the action is refused on the `audit` axis and the reservation released ("an unrecorded decision must not become an action"). Settlement is the reverse: it *throws* `SettlementUnrecordedError` with the receipt attached, because by then the money has moved. Reservation ids derive from `(kind, customer, casualty, incident, attemptNo)`, so a worker that crashes and replays gets the same grant back instead of double-committing.

**`terminus/budget.ts` + `store.ts`: commitment accounting.** The state is a pure value `{settled, committed, live{}}`, and `reserve`, `settle` and `release` are pure functions lifted into one ThrottleKit `Store.apply` (an atomic read-modify-write on a single key, available in memory or Postgres), so check and commit are indivisible. Expired reservations are swept at the head of every operation, which means no background process is needed. A settlement against a reservation that has already lapsed is still booked (and counted as an `orphan`). `overspendBoundPaise()` states the bound as code.

**`detect/cusum.ts`: the detector and its "way back".** Standard Page's CUSUM over the Bernoulli LLR, with each statistic capped at 24. The comments are unusually honest: they explicitly decline to quote Ville's `e^-h` bound, because a resetting CUSUM does not inherit it. The interesting part is the **recovery statistic** (`updateRecovery`): the same LLR with p₀ and p₁ swapped, run only while an incident is open, against a *frozen* claim (the **smallest** δ that crossed the threshold). It was added after Ameya found that a max-over-bank alarm clears on the *slowest-falling* statistic, so incidents stayed open about six hours after rails healed. With the fix, median resolution is 2.5 min and detection latency is bit-identical at h ≥ 10.

**`detect/baseline.ts` + `engine.ts`.** p₀ per slice is a decayed Beta-Binomial posterior mean shrunk toward the parent slice (κ = 120 pseudo-observations, decay 0.9995, clamped to [0.2%, 50%]). A new slice may not alarm on its own until it has seen 200 observations. `DetectionEngine` folds each attempt into every ancestor slice, coarsest first, and emits `opened` / `resolved` / `superseded` events (rollup).

**`policy/evaluate.ts` + `controller.ts`: steering.** `evaluateSteer` computes `movedShare`, `rescuedShare`, `collateralShare` and the destination failure rate, and returns a signed `netFailureDelta`. Because UPI fails ~2% and cards ~11–12%, a method-wide UPI demotion often *loses*, and the code declines it with a reason (break-even table in MEASUREMENT: `upi/hdfc` must reach 26.5% failure; `upi/canara/paytm` never). A steer is admitted through Terminus with `estimatedCost: 0`. Its blast radius is the mandate's in-flight cap (3 concurrent steers), and its duration is the reservation TTL (30 min), so a dead sentry's steers lapse by themselves. Lever and target selection have their own hysteresis. Rates are read from the holdout arm, because a steer contaminates the blended estimate that justified it.

**`recover/decide.ts` + `schedule.ts` + `probability.ts`.** When a payment can be charged again without the customer (only **12.0%** of benchmark casualties), retry is the whole decision and nobody is messaged. Otherwise each contact channel is priced as `p × amount × 0.3 margin − send cost − P(opt-out) × ₹200`, where opt-out risk rises 60% per prior contact. Channel exploration uses a hash of the casualty id (8%), not an RNG, so replays are deterministic. The schedule records *why* it is waiting (`rail-recovery`, `spontaneous-window`, `balance-likely`, `ladder`). A transient casualty whose rail stays down past a patience limit is re-read as needing the customer.

**`ledger/chain.ts`.** Each record carries `seq`, `at`, `actor`, `action`, `target` (an id, never personal data), `allowed`, `reason`, `binding` (the axis) and `meta`. Records are hashed over canonical JSON with `prevHash` from a 64-zero genesis. `MemoryLedger` keeps the chain in process.

**`proof/` + `apps/bench/src/scorecard.ts`: measurement as a gate.** Each claim is either an **invariant** (exact: e.g. `spend.worstCaseOverspendPaise = 0`, `recover.ledgerVerified = true`) or a **metric** with a tolerance of 3 standard deviations across 8 seeds from `bench:variance`, rounded up. `bench:bless` refuses to widen a band. A config hash and code revision are stamped into every result, so the gate never compares two different experiments. The report prints which bands are "wider than half the value they guard".

**`reason/gauntlet.ts` + `honesty.ts`: constraining the model.** The copy gauntlet requires `{amount}` and `{link}` placeholders and rejects invented URLs, invented rupee figures, the wrong script, text over the encoding-aware segment budget, and a prohibited-phrase list (for `timed`/`unknown` classes, any invented cause). The model never sees a customer name, because it writes a template that a pure function later fills in. `verifyExplanation` refuses an `explain` answer if any digit-figure in it does not appear character-for-character in the retrieved audit records. A test exists specifically for the bug where digits inside ids like `cas_9f21` used to launder fabricated numbers.

**`adapters/razorpay`.** The client: a `fetch` transport, retries only on 429/5xx with a clamped `Retry-After`, and a whole-call deadline sized to fit inside a reservation TTL. `renderCheckout` translates a steering plan into Checkout's `config.display` (`sequence`, `hide`, `show_default_blocks: false`), and an inexpressible suppression comes back as a `RenderDiagnostic` instead of being dropped silently. Inbound webhooks: HMAC-SHA256 over the **raw** body, timestamp window and event-id dedupe; failures still return 200 with the reason, because Razorpay retries non-2xx responses for hours. The translator takes a **required** pseudonymiser, so email and phone never get past the boundary. `probe.mjs` makes real test-mode calls and refuses a live key.

**`adapters/postgres`: fleet safety.** The casualty queue is a `jsonb` payload plus derived index columns and a `leased_until` lease column (ADR 0008: a column, not a row lock). It is tested against in-process Postgres (PGlite), including "acts on each casualty exactly once" under a simulated fleet. Budget and contact caps live in ThrottleKit's `PostgresStore` in the same database, so `KAIROS_DATABASE_URL` is the entire difference between one instance and a fleet.

## Quantitative methods — the actual math implemented

| Method | Formula / parameters | Where |
| --- | --- | --- |
| **Change detection** | `Sₙ = min(24, max(0, Sₙ₋₁ + LLR))`, `LLR = log(p₁/p₀)` on failure, `log((1−p₁)/(1−p₀))` on success; `p₁ = min(0.95, p₀+δ)`, δ ∈ {0.03, 0.08, 0.18, 0.40}; alarm at **h = 12**, clear threshold 3.6, 120 s dwell, 200-observation warm-up | `detect/cusum.ts`, `config.ts` |
| **Incident resolution** | Same LLR with `p₀ ↔ p₁` against the frozen smallest crossing δ; clears at 12, ceiling 24 | `cusum.ts: updateRecovery, alternativeRate` |
| **Per-slice baseline** | `p₀ = (fails + κ·p_parent)/(total + κ)`, κ = 120, geometric decay 0.9995 (~2,000-obs window), clamp [0.002, 0.5] | `detect/baseline.ts` |
| **Budget bound** | `available = budget − settled − committed`; reserve only if it fits ⇒ `settled + committed ≤ budget`; hence `final settled ≤ budget + maxInFlight × (maxActionCost − reservation)`; worst-case sizer ⇒ 0 | `terminus/budget.ts` |
| **Reservation sizers** | worst-case (default) · estimate · learned (newsvendor critical fractile) · predictive; reservation clamped to `maxActionCost` | `terminus/reservation.ts` (ThrottleKit) |
| **Contact cap** | Rolling-window quota (default 3 per 7 days) on the shared store; peek, then consume | `terminus/caps.ts` |
| **Steer pricing** | `netFailureDelta` = change in per-attempt failure probability across all traffic (rescued − collateral), with elasticity 0.35 and abandonment-on-suppress 0.08; act only if negative beyond `minBenefitPerAttempt` 0.0005 and corroborated over 2 windows | `policy/evaluate.ts`, `config.ts` |
| **Holdout** | `stableDraw(customer, incident)` hash in [0,1) < 0.10 (sticky per customer per incident); recovery control is hashed on casualty id | `domain`, `policy/holdout.ts`, `recover/decide.ts` |
| **Lift + CI** | Loss-rate delta (control − treated) per exposed / collateral / overall population, with a 95% half-width; intervals that span zero are labelled noise | `apps/bench/src/prevent.ts` |
| **Recovery probability** | Hierarchical Beta-Binomial over (action, class, railHealthy, ordinal bucket 0/1/2+), κ = 12, decay 0.999, cold start 0.15, clamp [0.005, 0.95]; classification confidence shrinks toward the base rate | `recover/probability.ts` |
| **EV gate** | `net = p × amount × 0.30 − sendPaise − min(1, optOutRate × (1 + 0.6·priorContacts)) × ₹200`; act if `net > 50 paise`; SMS 20p (worst 60p), WhatsApp 12p, email 2p, retry 1p | `recover/decide.ts` |
| **Calibration** | 10-bin expected calibration error, Brier score, Brier skill vs base rate | `recover/calibration.ts` |
| **SMS cost** | GSM-7 160/153 chars per segment vs UCS-2 70/67, encoding decided on the *rendered* message | `domain/segments.ts`, `reason/segment.ts` |
| **Gate tolerance** | `tolerance = ⌈3·sd⌉` (sample sd, 8 seeds); bands never widened by `bless` | `proof/variance.ts`, `metric.ts` |

## Core features & public API/commands

| Capability | Surface | Code |
| --- | --- | --- |
| Ask what checkout to render (Kairos + Razorpay `config`) | `POST /plan` on sentry (`:8080`) | `apps/sentry/src/server.ts`, `policy/plan.ts`, `razorpay/checkout.ts` |
| Report outcomes in batch | `POST /outcomes` | sentry |
| Ingest Razorpay webhooks directly | `POST /webhooks/razorpay` (needs `RAZORPAY_WEBHOOK_SECRET` + `LEDGER_PII_HASH_KEY`) | `sentry/razorpay-webhook.ts`, `razorpay/inbound.ts` |
| Operator view, audit chain, metrics | `GET /` · `/ledger` · `/health` · `/metrics` | sentry `public/index.html` |
| Drain casualties (dry-run) | `pnpm --filter @kairos/recover-worker run start`; `:9464 /health /ready /metrics` | `apps/recover-worker` |
| Author / sign / verify a mandate | `kairos-mandate form` (loopback `:8181`) · `seal` · `explain` · `verify` | `apps/mandate` |
| Out-of-band stop | `kairos-mandate status` · `stop "<reason>"` · `resume` (needs DB, not the signing key) | `apps/mandate/src/stop.ts`, `terminus/stop-switch.ts` |
| Stepped simulation console | `:8788` `GET /api/scenarios`, `POST /api/scenario`, `POST /api/step`, `GET /api/snapshot`; 6 scenarios (calm, issuer-outage, invisible-issuer, two-at-once, budget-exhaustion, kill-switch) | `apps/console` |
| "Why did Kairos do that?" | `pnpm --filter @kairos/console run explain -- <target>` | `packages/explain`, `reason/honesty.ts` |
| Whole stack in one command | `pnpm demo` / `demo:logs` / `demo:down` (Compose, 60× clock) | `scripts/demo.mjs` |
| Regenerate the copy library | `pnpm --filter @kairos/scribe run compose [--dry-run]`, `record` | `apps/scribe` |
| Live gateway probe | `pnpm razorpay:probe [--link]` → `docs/razorpay-probe.json` | `adapters/razorpay/scripts/probe.mjs` |

Binding axes, in admission order (`domain/admission.ts`): `mandate-signature`, `kill-switch`, `mandate-validity`, `action-not-allowed`, `stop-rule`, `expected-value`, `quiet-hours`, `budget`, `concurrency`, `contact-cap`, `audit`. Action kinds: `steer`, `retry`, `contact-sms`, `contact-whatsapp`, `contact-email`, `escalate`, `reason` (an LLM call is an admitted, priced action).

## Payment-gateway & model integrations

- **Razorpay: real but partial.** `docs/razorpay-probe.json` records three live **test-mode** calls on 2026-09-03: `POST /v1/orders` 200, `GET /v1/payments/<nonexistent>` 400 → `BAD_REQUEST_ERROR, retryable: false`, `POST /v1/payment_links` 200. The inbound webhook path is implemented and unit-tested; ARCHITECTURE says a real signed webhook was verified end to end, but no transcript of that is committed. **Modelled, never called:** Payment Downtime API, Subscriptions, Invoices, Settlements. Checkout `config` is emitted but never rendered in a real Checkout.
- **Outbound actions are not real.** The worker accepts only `KAIROS_DELIVERY=dry-run`, and any other mode throws, since no live gateway charge-with-token or DLT-registered SMS/WhatsApp sender is wired. `.env.example` lists `SMS_PROVIDER_KEY`, `WHATSAPP_PROVIDER_KEY` and `REDIS_URL`, but no code reads them; the only stores wired are `MemoryStore` and `PostgresStore`.
- **Gemini.** `adapters/reasoner-gemini` drives composer, classifier and explainer ports over `generateContent`. It reads quota waits from the `google.rpc.RetryInfo` body (Gemini sends no `Retry-After`), treats thinking tokens as output tokens (found to be a 23× under-report otherwise), prices calls at list rate even on the free tier, and replays a committed cassette in tests, so CI never makes a call.

## Tech stack & build tooling

- **Language:** TypeScript ^7.0.2 with a strict base config (`noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noPropertyAccessFromIndexSignature`, `verbatimModuleSyntax`, NodeNext). Per-package `tsc` builds to `dist/`; no bundler, including for the site.
- **Runtime:** Node ≥ 22, Fastify 5 (sentry, console), `pg` 8, zod 4 for boundary validation.
- **Tests:** Vitest 4 against `src/` through aliases, so a clean checkout needs no build; v8 coverage with thresholds lines/functions/statements 85, branches 80; PGlite for Postgres-backed tests.
- **Lint/format:** Biome. commitlint (conventional) runs on PRs. `scripts/check-deps.mjs` checks that manifests, tsconfig paths and Vitest aliases agree.
- **CI** (`.github/workflows/`): `ci.yml` runs lint, check:deps, typecheck and build, then tests with coverage on Node **22 and 24**. `bench.yml` runs the scorecard **regression gate** (`--profile=quick --gate`) against `docs/results/baseline-quick.json` on every push/PR. `pages.yml` builds and deploys the site. `security.yml` runs CodeQL, gitleaks over history, and `pnpm audit`. Dependabot covers npm and Actions.
- **Containers:** multi-stage `Dockerfile` (node:22-alpine), and a `docker-compose.yml` with Postgres 18 plus four services.

## Build / dev / test (actual commands)

```sh
pnpm install            # Node ≥ 22, pnpm ≥ 11
pnpm build              # tsc in every workspace package
pnpm test               # vitest run (against source)
pnpm test:coverage      # with thresholds
pnpm lint               # biome check .
pnpm check              # lint + check:deps + typecheck + test

pnpm bench:detect | bench:spend | bench:prevent | bench:recover   # each has a :quick variant
pnpm bench:scorecard    # full profile → docs/results/scorecard-full.*
pnpm bench:gate         # quick profile, judged against the committed baseline (what CI runs)
pnpm bench:variance     # 8 seeds, derives bands
pnpm bench:bless        # rewrite the baseline (never widens a band)

pnpm demo               # Postgres + sentry + worker + traffic + console under Compose; open :8080
KAIROS_MANDATE_SECRET=$(openssl rand -hex 32) pnpm --filter @kairos/sentry run start
pnpm --filter @kairos/postgres run schema | psql "$DATABASE_URL"
```

No API key is needed for anything except `scribe compose`/`record` (Gemini) and `razorpay:probe` (Razorpay test keys).

## Tests

**71 `.test.ts` files, ~1,184 test cases, 15,772 LOC**: about 34% of the TypeScript is tests. Every package has some, and the money-moving core is the most heavily tested: `terminus` has 2,031 test lines against 1,800 source (e.g. "holds the budget against a fleet all admitting at once", "treats a kill switch it cannot read as engaged", "refuses the action when the decision cannot be recorded", "throws on a settlement it cannot record…"), and `recover` has 2,010 against 2,198. Other notable suites: `signature.test.ts` asserts that perturbing *any* mandate field changes the signature; `honesty`/`gauntlet` tests run over the whole committed copy library; the Gemini adapter replays a recorded cassette; the Postgres queue tests run on PGlite; and `site.test.ts` derives figures the site states (e.g. the 59× ratio and the film size) from committed files so they cannot drift. The bench gate in CI also works as an end-to-end regression test over the full simulated system. I did not run the tests (read-only task). On the last commit (`f16c31e`, 2026-09-05), **CI, Bench and Pages are green on GitHub**. The scheduled **Security** workflow is **failing as of 2026-10-12**, on its `pnpm audit` job only; CodeQL and gitleaks pass.

## Measured results — checked against `docs/results/`

All figures are from **seeded simulated traffic** (no live merchant, no real customer ever contacted). I checked each against the committed blessed outputs (`scorecard-full.json` at config `1715c07b4cfff993`, revision `5cfa489`, and `recovery.txt`):

| Claim | Committed value | Note |
| --- | --- | --- |
| Median detection latency at h=12 | **92.8 s** | 0.143 false alarms/h (**2 alarms in 14 h**), 83.3% detected (20/24 trials; `thin-slice` 0/4) |
| Issuer collapse caught | **5.3 s** median | HDFC UPI → 55% instantly |
| Median incident resolution | **2.5 min** (p90 15.8 min) | 2 of 20 trials lost cover mid-outage |
| Naive worker overspend, ₹500 budget, 64 workers | **₹100**, **271** contact-cap violations | out of 346 sends |
| Terminus, worst-case sizer, 1–64 workers | **₹0 over**, 0 violations, ≥ **99.6%** of budget used | adaptive sizers ≤ **₹7** over, each within its own computed bound |
| Suppression lift (netbanking-hdfc, scorecard) | **49.8 pp** | control arm is only **16 attempts**; 95% half-width ±24 pp |
| UPI demotion lift (scorecard) | **+11.0 pp** exposed, **−2.7 pp** collateral | collateral harm is reported, not netted away |
| Recovery: 5,556 casualties, ₹48,04,504 | do-nothing recovers ₹11,39,929 (**23.7%**) | 2,331 casualties held out |
| Kairos + generated copy vs fixed ladder | **₹6,82,069 vs ₹5,84,006** incremental (+16.8%); 5,749 vs 10,681 msgs; 54 vs 132 opt-outs; true cost ₹11,651 vs ₹28,495 (−59%) | ₹6,82,069 / ₹11,651 ≈ **58.5×** (the site's "59×") |
| Kairos + **template** copy vs ladder | +5.1% incremental, −56.3% true cost, −45.5% msgs | the comparison with no LLM involved |
| Calibration | ECE **1.84%**, Brier skill 0.252, 6,068 predictions | |

## Status, completeness & notable gaps

- **Status: a finished competition entry, inactive since 2026-09-05.** No commits have landed on `main` since then; five Dependabot PR branches are open, and the security audit job is now red. The README still says "Under active development."
- **The engineering is well above typical hackathon quality.** It has a strict pure core, ports and adapters, ten ADRs, seeded and reproducible benchmarks, variance-derived CI gates, holdout controls throughout, and docs that record their own retractions ("design claims that did not survive contact with measurement").
- **Simulation boundary:** every number comes from synthetic traffic, and the customer behaviour model (elasticity, spontaneous-return rates, opt-out rates, a 55/25/10/10 language mix, the ₹200 opt-out price) is invented and stated as such. Detection samples are small: 24 trials and 2 false alarms per threshold.
- **Not built, or not wired:**
  - **Live outbound:** the worker runs dry-run only.
  - **The audit ledger is in memory, per process** (`MemoryLedger`). A fleet produces N separate chains, and nothing persists across a restart. `recover-worker/main.ts` says plainly that "a shared appender… is not built."
  - **The generated copy library is used only in the benchmark.** The deployable `recover-worker` constructs `RecoveryExecutor` without `copy`, so it sends English templates.
  - **The Gemini residual classifier** is implemented but only invoked by `scribe record`.
  - **No exploration bandit:** variant choice is a deterministic hash, which `reason/source.ts` acknowledges.
  - **e-BH multiplicity control** (ARCHITECTURE §5) does not exist; only an `eValue()` helper and rollup do.
  - **Console UI and a demo checkout/storefront:** neither is built (ARCHITECTURE §17).
  - **"Chaos demonstrations we ship"** (ARCHITECTURE §12): no chaos script exists; failure handling is covered by unit tests instead.
  - **The film's render pipeline is not in the repo.** The 41.6 MB MP4 is committed, but not the source that made it.
- **Conditions on the "provable" bound:** the overspend bound is a short argument in `budget.ts` comments plus tests and an empirical sweep, not a formal proof (unlike ThrottleKit's TLA+ specs). It holds only if (a) adapter-reported costs never exceed `maxActionCost`, which is flagged as `exceededActionCap` but not clamped, (b) no reservation lapses before settlement, since orphans are booked unbounded but counted (asserted 0 in the bench), and (c) the backing store's `apply` is atomic. The bench's concurrency is cooperative and single-threaded (MEASUREMENT says so); the budget has not been measured against a multi-process shared store.
- **Known blind spot:** a degradation confined to a slice with ~4 attempts a minute (`thin-slice`) is never detected. This is documented as an evidence limit rather than a bug.

## README vs. code

Most spot-checks matched the committed results: 93 s, 0.14/h, 83%, 5 s, 2.5 min, ₹100 / 271 / ₹0 / ≥99.6%, "17% more / half the messages / 54 vs 132 / 59% lower true cost", 1.8% calibration, 24% unaided, ₹68,226 / 68 fewer messages / 4 fewer opt-outs, 6 console scenarios, 10 ADRs, and the 3-call probe. The concrete discrepancies:

1. **Invariant and band counts are wrong everywhere they appear.** The README says "seventeen claims… exactly… twenty-one… carry a band… **Nine of the twenty-three** bands are wider than the value", which contradicts itself (21 vs 23). MEASUREMENT.md and `llms.txt` say "eight of twenty-one". The committed baseline has **19 invariants and 27 banded metrics**. The gate's own criterion (`report.ts`) is "wider than **half** the value", which flags **12 of 27**; strictly wider than the value is 5.
2. **The generated-copy sweep is quoted from an older run.** The README's "₹10,367 against ₹1.05 lakh… on a range across seeds that straddles zero" comes from a four-seed sweep at `f163eb1`. The committed `recovery.txt` (regenerated at `5cfa489`) shows **₹11,823.50 vs ₹95,272.25**, and its worst seed at penalty 1.00 is **+₹728**, so it does *not* straddle zero. The qualitative conclusion (the gain comes only from readability and is near zero at penalty 1) still holds.
3. **The "17% more money" headline uses the generated-copy arm**, whose advantage the README itself says rests on an unmeasured readability weight. With templates only, Kairos is +5.1% incremental at 56% lower true cost. That still wins on every axis, but by much less.
4. **A stale pointer:** "MEASUREMENT.md… records the arm where a simpler baseline recovers more money than Kairos does." After the last commit, no committed arm beats Kairos; MEASUREMENT only keeps the history of the reversal.
5. **"The library is now read at send time"** is true of the benchmark harness only. The shipped worker sends English templates, as the README's own demo output shows.
6. **The steering figures (41%→12%, +13 pp, −1.85 pp) come from the Phase 3 prevent bench**, whose output is not committed under `docs/results/`. The current scorecard's UPI-demotion row reads +11.0 pp and −2.7 pp.
7. **Minor numeric drift:** "180 calls" (the library provenance says **190 calls, ₹8.15**; the dev section says "about ₹10"); "₹8" adaptive overspend (₹8 in the quick baseline, ₹7 in the full scorecard); the README's probe sample (165 ms, `order_TXhjDUA94yteWS`) comes from a different run than the committed JSON (235 ms, `order_TXiGidd0dxx2gS`).
8. **MEASUREMENT.md contradicts itself in places.** The Phase 4 calibration (ECE 1.3%, 6,472 predictions) and holdout (2,172 / ₹5,56,747) figures predate the committed run (1.84%, 6,068; 2,331 / ₹5,75,874). The window-sweep section says both "the default is now at the peak" and "the sweep says it is too short", and the committed `recovery.txt` has 120 min ahead of the 45-min default. Phase 5.5 still says "nothing consumes the library at runtime yet." The docs flag most of this drift themselves (open question 16).
9. **ARCHITECTURE.md describes features the code lacks:** e-BH false-discovery control, a `maxSteeredFraction` blast-radius parameter (the code uses a count of concurrent steers), an exploration bandit, and shipped chaos demos.
10. **`honesty.ts` checks digits only.** Its doc comment says a model "cannot say 'three of those four attempts' unless the source said 'three'", but spelled-out numbers are never extracted, so a fabricated "three" would pass the check.

None of these changes the core engineering claims: the admission ordering, commitment accounting, fail-closed kill switch, holdout-based incrementality and hash-chained refusals are all implemented as described.

---

*Doc generated 2026-10-12 by reading the source at C:\Users\ameya\Documents\razorpay\kairos. Source of truth: https://github.com/AmeyaBorkar/kairos*
