# mosaic

> A Rust + WebAssembly + TypeScript **platform substrate for community-authored "visual methods"** (called *Facets*) that turn media into text art: ASCII density/edge art, glyph shape-matching, error-diffusion dithering, braille, half-block colour pixel art, and audio spectrograms. Facets are untrusted code, either WebAssembly modules or bytecode programs written in a small expression language Ameya calls *Glint*. Both kinds run with **zero imports** and with fuel, memory and wall-clock limits. The server runs them on wasmtime; the browser runs the same Facets on its own WebAssembly engine. Golden-vector conformance tests and per-Facet certificates (bound to the bytes by SHA-256) make "browser preview == server render" a property that gets checked, not just assumed. The code ships as a 13-crate Cargo workspace (two engines, sandbox, bytecode VM, DSL compiler, certifier, registry, axum HTTP server, wasm-bindgen bridge) plus a TypeScript browser host. The web UI it is designed for has **not** been built.

| | |
| --- | --- |
| **Repository** | https://github.com/AmeyaBorkar/mosaic |
| **Visibility** | Public (0 stars; 11 open Dependabot PRs at time of reading) |
| **Category** | systems |
| **Primary language(s)** | Rust (~87% of code) + TypeScript (~13%). Rust edition 2024 for the workspace (2021 for the guest Facets); toolchain pinned to **1.97.1** in `rust-toolchain.toml` |
| **Local path** | `C:\Users\ameya\Documents\Mosiac` (note the directory is spelled "Mosiac") |
| **Default branch** | `main` |
| **Lines of code (computed)** | **Rust 15,411**: 11,207 in crate `src/` (includes inline `#[cfg(test)]` modules), 1,925 in crate integration tests, 1,395 in examples and golden emitters, 448 in guest Facets, 436 in a standalone tool. **TypeScript 2,293**: 1,120 src + 1,173 test, plus a 36-line `.mjs` helper. Total **≈17,740 LOC of code**. Also tracked: 5,354 lines of committed JSON golden vectors (7 files), ~2,700 lines of Markdown (12 files), 443 lines of TOML, 145 lines of CI YAML. Lockfiles excluded. |
| **Source files (computed)** | 52 `.rs` (25 crate src, 9 integration tests, 11 examples, 6 Facets, 1 tool), 7 TS src, 13 `.test.ts`, 1 `.mjs` = **73 code files**. Also 15 committed `.wasm` fixtures (copies of 6 distinct Facets, 93 B to 5.5 KB). **224 Rust test functions + 60 JS test cases.** |
| **Package / host platform** | Cargo workspace of 13 member crates, all `publish = false`. Six `no_std` `wasm32-unknown-unknown` Facet crates and one demo tool are excluded from the workspace. A pnpm workspace holds one private package, `@mosaic/facet-abi`, which is consumed as raw `.ts` through Node 24 type-stripping (no build step). The server deploys via a multi-stage `Dockerfile` (cargo-chef → `debian:bookworm-slim`, non-root user). **Nothing is published** to crates.io or npm. |
| **Key dependencies** | Sandbox: `wasmtime` 47 (Cranelift), `wasmparser` 0.252, `bytemuck`, `anyhow`. Engines and VM: `libm` 0.2 only. `glyph-atlas`, `dither` and `mosaic-core` have **zero** dependencies. Server: `axum` 0.8, `tokio` 1, `redb` 4.1, `sha2` 0.10, `uuid` 1, `base64` 0.22, `serde`/`serde_json`. Browser bridge: `wasm-bindgen` 0.2, `js-sys` 0.3. Tool: `image` 0.25. JS has **no runtime dependencies**; dev-only `typescript` ^5.9.2 and `@types/node` ^24.10, and tests use `node:test`. |
| **License** | Dual **MIT OR Apache-2.0** © 2026 Ameya Borkar (`LICENSE-MIT` + `LICENSE-APACHE`). GitHub's license detector shows only Apache-2.0. |
| **Last commit** | `f0528de`, "feat(vm,dsl): deterministic noise(x,y) hash opcode" (2026-07-28). First commit `bba3ccb`, "feat: scaffold workspace, licensing, docs, and domain contract" (2026-07-21). **104 commits**, all by Ameya, on three days: 41 on Jul 21, 52 on Jul 27, 11 on Jul 28. CI on `main` HEAD is green (checked via `gh run list`). |

---

## What it actually is (be concrete)

The vision (`vision.md`) is a "Shadertoy for media-to-art methods". Authors publish a *style*; anyone runs it on their own media. The code builds the **substrate** for that idea and leaves out the product. There is no web app, no gallery and no editor UI. What exists is the engine, the sandbox, the authoring language, the certification gate, the registry and the HTTP API, all heavily tested.

The vocabulary has three layers:

- **Mosaic (platform):** the domain-agnostic substrate (`mosaic-core`, `mosaic-runtime`, `mosaic-certify`, `mosaic-registry`, `mosaic-server`).
- **Tessera (engine):** one engine per media domain. Each fills a five-slot contract: Input · Unit · Feature vocabulary · Output primitive · Composition. Two engines are built:
  - `tessera-ascii`: RGBA image → grid of cells.
  - `tessera-spectral`: mono PCM audio → STFT time × log-frequency grid.
- **Facet (style):** a pure function from **a per-cell `f32` feature vector to one `u32` token** (a Unicode codepoint).

Every render follows the same pattern: **decompose → measure → map → recompose**.

1. The trusted engine measures features per cell.
2. The untrusted Facet maps each cell's features to a token, inside the sandbox.
3. The trusted composer validates and assembles the tokens into text.

Because a Facet only ever sees numbers, the *same Facet binary* runs on image and audio features. That cross-domain claim is a passing test (`tessera-spectral/tests/sandboxed_spectral.rs`), not just an assertion in the docs.

The "preview == render" guarantee comes from four pieces working together:

- **Shared engine code.** The engines are written once in Rust and compiled both natively (server) and to wasm (browser, via `mosaic-wasm`).
- **One implementation of each non-trivial routine.** The glyph matcher, Floyd–Steinberg and the bytecode VM are each a single `no_std` crate. That one crate is linked into both the native engine and the guest Facet, so there are no two copies to drift apart.
- **Determinism discipline.** Every transcendental goes through `libm`, there is no `mul_add`, wasmtime canonicalizes NaNs and disables relaxed-SIMD, and float reductions are written in a fixed order.
- **Committed golden vectors.** Seven golden JSON files emitted by the native host are replayed in Node against the wasm builds.

## Architecture & how it's structured

Annotated tree (`target/` and `node_modules/` omitted):

```
Mosiac/
├─ Cargo.toml                workspace: 13 member crates; facets/* and tools/* excluded
├─ rust-toolchain.toml       pinned 1.97.1 + wasm32-unknown-unknown (audit H4)
├─ package.json · pnpm-workspace.yaml   private root; workspace globs packages/* and apps/* (apps/ does not exist)
├─ Dockerfile                multi-stage production image for mosaic-server
├─ vision.md                 product vision + open questions O1–O5
├─ docs/
│  ├─ architecture.md        decision log D1–D21 (the real design record)
│  ├─ audit-2026-07-27.md    25-finding adversarial audit (C1, H1–H6, M1–M15, L1–L9), all marked resolved
│  ├─ capability-roadmap.md  ranked 11-item roadmap (items 1–5 now done)
│  ├─ roadmap-*-2026-07.md   server-side and authoring build plans + progress logs
│  └─ ui-builder-guide.md    spec for the (unbuilt) web UI: screens, SDK, API, Glint cheatsheet
├─ crates/
│  ├─ mosaic-core/      860  contract: FeatureSchema/FeatureType/Gather, compose_codepoints (untrusted-glyph mask),
│  │                         composite::Canvas (painter's algorithm, 4 blends), manifest::Manifest;
│  │                         input/unit/output/propagate are empty doc-only modules
│  ├─ glyph-atlas/      245  no_std 16-glyph 8×8 atlas + SSD matcher (shared by engine and Facet)
│  ├─ dither/           139  no_std 1-bit Floyd–Steinberg, power-of-two weights (shared by engine and Facet)
│  ├─ mosaic-runtime/   939  wasmtime sandbox: run_i32 / run_map / run_map_2d / run_program
│  ├─ mosaic-vm/        991  no_std, forbid-unsafe stack VM (28 opcodes, straight-line, statically validated)
│  ├─ mosaic-dsl/     1,187  Glint compiler: lexer → recursive-descent parser → AST → bytecode
│  ├─ tessera-ascii/  1,407  image engine: L0/L1/L2 features, u/v position, r/g/b colour,
│  │                         render_ascii/structural/dither/braille, half-block + tint colour
│  ├─ tessera-spectral/ 741  audio engine: Hann window + Goertzel filterbank, log-spaced bands
│  ├─ mosaic-compose/   471  declarative JSON Composition + LayerResolver seam (library-only)
│  ├─ mosaic-certify/ 1,448  static profile gate (wasmparser) + probe certificates for wasm and DSL programs;
│  │                         embeds facet_interp.wasm as INTERP_WASM
│  ├─ mosaic-registry/  675  Store trait + InMemoryStore + durable RedbStore; FacetArtifact = Wasm | Program
│  ├─ mosaic-server/  1,562  axum service: certify, render, registry, bearer auth, moderation (10 routes)
│  └─ mosaic-wasm/      542  wasm-bindgen bridge: extract*, compose, Canvas, compileDsl, renderHalfblock/Braille, extractColors
├─ facets/                   guest Facets (no_std Rust → wasm32, opt-level "s", LTO, 16 MiB --max-memory)
│  ├─ ramp/  structural/  dither/      real methods (gather, gather L2, propagation)
│  ├─ interp/                          the one trusted DSL interpreter Facet (wraps mosaic-vm)
│  └─ spin/  liar/                     adversarial fixtures (never returns; wild alloc pointer)
├─ packages/facet-abi/       @mosaic/facet-abi: browser Facet host (host.ts, sandbox.ts, worker.ts, certificate.ts, program.ts)
├─ tools/mosaic-pixel/  436  standalone hex-pixel-art PNG CLI; NOT part of the pipeline, not built by CI
└─ scripts/                  build-facets.sh (rebuild + copy .wasm), verify-fixtures.sh (re-emit 7 goldens + git diff)
```

**Entry points**

- Server binary: `crates/mosaic-server/src/main.rs`. It builds the `Sandbox`, compiles the embedded interpreter once, picks `RedbStore` when `MOSAIC_DB` is set (else in-memory), and binds `MOSAIC_ADDR` (default `127.0.0.1:8080`).
- Browser: `mosaic-wasm` (built with `wasm-pack`) plus `packages/facet-abi/src/index.ts`.
- Library consumers use the crates directly. The `examples/` directories double as the golden-vector emitters.

## Code walkthrough (the important parts)

**`mosaic-runtime`: the sandbox.**

- One `wasmtime::Engine` is configured with:
  - `consume_fuel(true)` and `epoch_interruption(true)`;
  - NaN canonicalization on;
  - relaxed-SIMD, threads, multi-memory and **memory64 explicitly off** (wasmtime 47 enables memory64 by default).
- A background thread ticks the epoch every 10 ms, so a wall-clock deadline is enforced without the guest cooperating.
- Default `Limits`: 100 M fuel, 16 MiB memory, 5,000 ms wall clock.
- Every execution gets a fresh `Store` with `StoreLimits` capping it at 1 memory, 1 table, 10,000 table elements and 1 instance. It is instantiated with an empty import list, so purity is structural rather than policed.
- Before Cranelift runs, a `wasmparser` pre-pass rejects modules over 8 MiB, with more than 4,096 functions, or with any function body over 256 KiB. This bounds compile cost (audit M9).
- `run_marshalled` is the single marshaller behind all three map ABIs:
  - it allocates through the guest's own `alloc` export;
  - it writes little-endian `f32` features with a zero-copy `bytemuck` cast;
  - it calls `run` (gather), `run2d` (propagation) or `load_program` + `run` (DSL);
  - it reads `u32` tokens back;
  - every size is checked against 32-bit wasm addressing before any host allocation.
- The epoch deadline exists because fuel charges one unit for an O(n) `memory.fill`. That was the audit's Critical finding (C1), and the test `epoch_bounds_bulk_memory_loop` is its regression test.

**`tessera-ascii`: the image engine.**

- `Grid` partitions the image with integer scaling (`col * width / cols`).
- `feature::extract` makes two passes:
  1. mean Rec. 709 luma per cell;
  2. a Sobel gradient over the *cell* luma grid (a radius-1 gather), using `libm::atan2f` for orientation.
- It then appends the normalized cell centre `u, v` and the mean colour `r, g, b`. The colour reuses the same integer `mean_color` as the tint render. Each cell gets **8 slots**: `[luma, grad_mag, grad_dir, u, v, r, g, b]`.
- `extract_structural` emits an 8×8 sub-cell luma patch instead (**stride 64**). Its consumer, `glyph_atlas::match_glyph`, does nearest-glyph SSD over a 16-glyph atlas whose ink table is precomputed at compile time.
- Feature extraction is byte-budgeted at `MAX_FEATURE_BYTES = 8 MiB`, so the stride-64 path cannot drive a multi-GB allocation.
- Facet-free render modes:
  - `render_braille`: a 2×4 dot threshold per cell, packed into U+2800–U+28FF;
  - `color::render_halfblock`: `▀` with top and bottom mean colours.
- `density_glyph` deliberately uses `(l*(n-1)+0.5) as usize`, not `.round()`. The two disagree at exactly one `f32` luma (`0x3D638E38`), which broke native≡Facet parity (audit M5).

**`tessera-spectral`: the second engine.**

- Columns are STFT frames and rows are log-spaced bands between `fmin` and `fmax`, with band 0 drawn at the bottom.
- Each frame is Hann-windowed; each band's energy comes from a Goertzel filter whose final magnitude is accumulated in `f64` (audit M1: f32 cancelled catastrophically near DC). The grid is then peak-normalized to `[0, 1]`.
- Work is capped by `MAX_WINDOW = 65,536`, `MAX_CELLS = 8 M` and `MAX_STFT_OPS = 2^30`.
- Each cell exposes one scalar, so `facet_ramp` (gather) and `facet_dither` (propagation) run on audio unmodified.

**`mosaic-vm` + `mosaic-dsl`: Glint.**

The bytecode format is: header (`MVM1` magic, `u16` stride, `n_params`, `n_tables`) · `f32` params · `u32` codepoint tables · code section.

`validate` makes one static pass that:

- bounds-checks every `LOADF`, `LOADP` and `TABLE` operand;
- rejects surrogate and out-of-range table codepoints;
- simulates the stack effect to rule out underflow, overflow past 64, or anything other than exactly one value at `END`.

Because v1 has **no jumps**, every program terminates. The 28 opcodes are const/load/dup/pop, arithmetic, comparisons and logic, min/max/abs/clamp/floor/trunc, branchless `SELECT`, `TABLE` lookup, and `HASH`. `HASH` is PCG (Jarzynski & Olano 2020) over the raw IEEE bits, after canonicalizing NaN and `-0.0` so it is bit-identical on native and wasm.

The compiler supports: identifiers resolved against a caller-supplied `Schema`; numbers; char and string literals; the full operator set and ternary; `let` bindings; and the builtins `abs floor trunc min max clamp select mix remap smoothstep noise ramp glyph`.

- Bindings and desugared multi-use arguments are stored once and referenced via `Expr::LetRef`, so nested `mix`/`smoothstep` cannot blow up the AST exponentially.
- `emit` still caps the emitted code at the VM's 64 KiB limit.
- Parse depth is capped at 256 so hostile input cannot overflow the host stack.

`patch_params` rewrites the fixed-offset params section in place. That is how a live control can change a value without recompiling.

**`mosaic-certify`: the authoritative gate.**

`check_profile` walks the module with `wasmparser` and admits only:

- zero imports;
- one bounded, non-shared, 32-bit memory of ≤ 256 pages;
- at most one table with ≤ 10,000 elements;
- `memory` and `alloc` exports;
- **exactly one** of `run` or `run2d`.

Failures map to one of 31 machine-stable `RejectionCode`s.

`certify` then compiles the module in the real sandbox and runs a deterministic, seeded probe suite:

- gather Facets: strides 1–8 × 24 cells;
- propagation Facets: 8×8, 6×4 and 5×3 grids.

It records `Tokens` or `Trapped` per probe and emits a `Certificate` bound by SHA-256 to the exact bytes. A Facet that traps on every probe is rejected.

`certify_program` does the same for bytecode: native `validate` first (precise codes), then four probes run through the embedded `facet_interp.wasm`. The docs are honest that the probes are "a representative sample, not a proof."

**`mosaic-server`: the HTTP authority.**

The axum router has 10 routes. CPU work (compile, certify, render, redb I/O) runs on `spawn_blocking`.

`POST /v1/render`:

- is discriminated by `engine`: `ascii`, `ascii-structural`, `spectral`, `halfblock` or `braille`;
- takes raw base64 RGBA8 or f32 PCM, so there is no codec ambiguity;
- resolves the Facet either inline (runs `check_profile` first) or by registry `id` (only `Published` Facets render by id);
- then runs extract → sandbox → `compose_codepoints`, the same source the browser compiles.

Registry and auth:

- `POST /v1/facets` requires the author role. It certifies the submission, either a wasm module or a `program` + `engine` with a stride check, and stores it as `Certified`.
- `POST /v1/facets/{id}/moderate` requires the moderator role. The only allowed transitions are `Certified → Published | Rejected`; anything else returns 409.
- Unpublished Facets return 404 to everyone except their author and moderators.
- Bearer tokens come from a JSON file named by `MOSAIC_TOKENS` and are stored as `SHA-256(token) → Principal`.

**`packages/facet-abi`: the browser host.**

- `host.ts` mirrors `run_map`, `run_map_2d` and `run_program` exactly. That includes hand-parsing the memory and table sections from raw bytes, because `WebAssembly.Module` reflection does not expose limits. It also has an overflow-safe LEB128 reader (audit H2).
- `sandbox.ts` runs Facets in a Web Worker under a wall-clock timeout (default 250 ms) and calls `terminate()` on overrun. This is the browser's substitute for fuel.
- `certificate.ts` `verifyCertificate` / `verifyProgramCertificate` re-hash the bytes and replay every probe, requiring identical tokens or an identical trap.

## Core features & public API/commands

| Capability | Server (`mosaic-server`) | Browser | Rust crate |
| --- | --- | --- | --- |
| Image → ASCII via a gather Facet (stride 8) | `POST /v1/render` `engine: ascii` | `extract_features` → `runFacetSandboxed` → `compose` | `tessera_ascii::feature::extract` + `Sandbox::run_map` |
| L2 structural glyph matching (stride 64) | `engine: ascii-structural` | `extract_structural_features` | `render_structural`, `glyph_atlas::match_glyph` |
| Error-diffusion dithering (propagation ABI) | inline/registered `run2d` Facet | `runFacetSandboxed2d` | `Sandbox::run_map_2d`, `dither::floyd_steinberg` |
| Audio → spectrogram art (stride 1) | `engine: spectral` | `extract_spectral_features` | `tessera_spectral::feature::extract` |
| Braille (~8× resolution, no Facet) | `engine: braille` | `renderBraille` | `render_braille` |
| Half-block colour pixel art / glyph tint | `engine: halfblock`; `params.color: true` | `renderHalfblock`, `extractColors` | `color::render_halfblock`, `extract_cell_colors` |
| Author a Facet in Glint | publish as `program` + `engine` | `compileDsl(engine, src, params)` → `runFacetProgramSandboxed` | `mosaic_dsl::compile`, `Sandbox::run_program` |
| Live parameter controls | — | `applyParams` (patch bytecode in place) | `mosaic_vm::patch_params` |
| Certify (gate + golden probes) | `POST /v1/certify` | `verifyCertificate`, `verifyProgramCertificate` | `certify`, `certify_program`, `check_profile` |
| Registry + moderation | `GET/POST /v1/facets`, `GET /v1/facets/{id}[/wasm\|/program]`, `POST …/moderate` | — | `Store`, `InMemoryStore`, `RedbStore` |
| Identity | `GET /v1/whoami`, `GET /healthz` | — | `AuthConfig` |
| Composition (overlay/tile/mask/stipple) | — (no endpoint) | `new Canvas(...).place(...)` | `mosaic_core::composite`, `mosaic_compose::render` |

**Feature names a Glint program can read:** `ascii` → `luma grad_mag grad_dir u v r g b`; `spectral` → `band_energy`. The browser compiler has no schema for `ascii-structural`, although the server accepts stride-64 programs for that engine.

## Sandbox & determinism guarantees (as implemented)

| Bound | Native (wasmtime) | Browser (`facet-abi`) |
| --- | --- | --- |
| Imports | 0 (instantiate with `&[]`) | 0 (reject if `WebAssembly.Module.imports` is non-empty) |
| CPU | 100 M fuel + 5 s epoch deadline | 250 ms Worker timeout + `terminate()` |
| Memory | `StoreLimits` 16 MiB, 1 memory | declared max ≤ 256 pages (16 MiB), 1 memory; Facets built with `--max-memory` |
| Tables | 1 table, ≤ 10,000 elements | same, parsed from raw bytes |
| Module size / compile cost | ≤ 8 MiB, ≤ 4,096 fns, ≤ 256 KiB per body | ≤ 8 MiB |
| memory64 / shared / multi-memory | disabled in `Config` | rejected by the byte parser |
| NaN payloads, relaxed-SIMD | canonicalized / disabled | **cannot be enforced** (documented); the server render is authoritative |
| Untrusted output | `compose_codepoints` masks C0/C1, bidi overrides, U+2028/2029, zero-width chars and U+0300–036F combining marks to U+FFFD | same function, compiled to wasm |

The documented residual gaps (audit M6) are that the browser checks only export *arity*, not wasm value types, and accepts atomics on non-shared memory. Both are contained by the server being the render of record.

## Tech stack & build tooling

- **Rust 1.97.1** (pinned), edition 2024; every library crate except `mosaic-wasm` carries `#![forbid(unsafe_code)]`. `mosaic-wasm` has one `unsafe` block, a zero-copy `Float32Array::view`, with a SAFETY comment. Guest Facets use `unsafe` only for raw-pointer ABI marshalling and a static bump arena (12 MiB) that returns `-1` when exhausted.
- **wasm-pack** builds `mosaic-wasm` for `--target nodejs` (as used in CI). Facets are built with `RUSTFLAGS="-C link-arg=--max-memory=16777216"`.
- **TypeScript** is type-checked with `tsc --noEmit` under `strict`, `noUncheckedIndexedAccess` and `verbatimModuleSyntax`. Tests run on Node 24's built-in `node --test` with TypeScript type-stripping, with no test framework.
- **CI** (`.github/workflows/ci.yml`) has three jobs:
  1. **rust**: `fmt --check`, `clippy --locked --all-targets --all-features -D warnings`, `cargo test --locked --workspace`, a wasm32 build of engine + VM + bridge, per-Facet fmt/clippy, `build-facets.sh` followed by `git diff --exit-code '*.wasm'` (committed binaries must match their source), `verify-fixtures.sh` (all 7 goldens re-emitted and diffed), and a release build of `mosaic-server`;
  2. **web**: wasm-pack, `pnpm install --frozen-lockfile`, `tsc`, and the 13 `node --test` files;
  3. **audit**: `cargo audit`.
- Dependabot is configured for Cargo and GitHub Actions.
- **Container:** `Dockerfile` (cargo-chef dependency caching, `--locked`, non-root `mosaic` user, `EXPOSE 8080`, liveness at `/healthz`). CD (image push and deploy) is explicitly left unwired.

## Build / dev / test (actual commands)

```sh
cargo test --workspace                                   # 224 Rust tests (needs MSVC build tools on Windows)
cargo clippy --all-targets --all-features -- -D warnings
bash scripts/build-facets.sh                             # rebuild the 6 Facets → 15 committed .wasm copies
bash scripts/verify-fixtures.sh                          # re-emit 7 golden JSONs; fail on any diff

cargo install wasm-pack
wasm-pack build crates/mosaic-wasm --target nodejs --dev --out-dir pkg
pnpm install && pnpm run typecheck && pnpm test          # root script runs 12 of the 13 JS test files

cargo run -p mosaic-server                               # 127.0.0.1:8080; MOSAIC_DB / MOSAIC_TOKENS / MOSAIC_ADDR
cargo run -p tessera-ascii --example demo                # synthetic images → ASCII in the terminal
cargo run -p mosaic-runtime --example dsl_pixel_art -- luma.txt 80 40
cargo run --manifest-path tools/mosaic-pixel/Cargo.toml -- img.png --style all --palette pico8
```

## Tests

**Rust: 224 test functions** (191 `#[test]` + 33 `#[tokio::test]`).

- Most are colocated in `src/`, led by the runtime (24), `tessera-ascii` (23), `mosaic-certify::profile` (20), the VM (15), the DSL (15) and `mosaic-core` (15).
- The other 9 integration-test files hold 55, of which 33 are in `mosaic-server/tests/http.rs`. Those drive the router in-process with `tower::ServiceExt::oneshot` and cover certify/render happy and error paths, 401/403/404/409 semantics, the moderation state machine, and render-by-id for both wasm and program Facets.

Notable test styles:

- **Adversarial sandbox fixtures in WAT:** infinite loop, `memory.fill` bulk loop (epoch), wild `alloc` pointer, memory bomb, table bomb, multi-memory, shared memory, memory64, more than 4,096 functions, oversized module, missing exports, wrong signatures, and imported memory.
- **Randomized native≡sandboxed sweeps** using deterministic xorshift PRNGs: 64 random images each for the ramp, structural and dither Facets, and 32 random signals for the spectral engine.
- **A 4,096-sample bucket-distribution test** for `noise`.

**JS: 13 test files, 56 `test()` declarations → 60 test cases** (two declarations loop over 2 wasm certificates and 4 program certificates). They cover:

- conformance against the native goldens;
- adversarial admission;
- the real-module conformance gate (made non-vacuous after audit M13);
- Worker timeout killing the `spin` Facet;
- DSL in the browser;
- certificate and program-certificate replay, including tampered-token, hash-mismatch and patched-bytecode rejection;
- the full browser pipelines for ascii, structural, dither, braille, spectral and composite.

`tools/mosaic-pixel` has **no tests** and is not built in CI. No benchmarks exist anywhere in the repository.

## Status, completeness & notable gaps

- **Status: an engineering-complete backend substrate, pre-product, unpublished.**
  - Every vision open question (O1–O5, O4.1) is closed by a recorded decision (D5–D14).
  - The server-side build-out (D15–D17) and capability-roadmap items 1–5 (D18–D21) are implemented and CI-green.
  - All crates are `0.0.0` with `publish = false`.
- **Development cadence:** three intensive days. Jul 21 built the core, the second engine, composition and the DSL. Jul 27 was a 25-finding adversarial self-audit plus the fixes for all of it, followed by certify, server and registry. Jul 28 added program publishing, colour features, braille and `noise`. There have been no commits since 2026-07-28. Eleven Dependabot PRs are open, and the CI runs on the two newest (`wat`, `js-sys`) failed. The `wasmtime 47 → 49` bump passes CI but is unmerged.
- **Not built:**
  - **The web shell.** `apps/web` is referenced in `pnpm-workspace.yaml`, `architecture.md` and the 449-line `ui-builder-guide.md`, but there is no `apps/` directory. "Auto-generated controls" therefore exist only as data: `mosaic_core::manifest::Manifest` is constructed by the engines' `render::manifest()` and asserted in tests, but nothing consumes it. The browser-facing manifest is a separate `{name, value, index}` JSON from `compileDsl`.
  - **More engines.** ANSI, halftone and data→art Tesserae are not built.
  - **Roadmap items 6–11** are not built: bounded loops, DSL-level dithering, `sqrt`, video/live engines, a node editor, and composition authoring.
  - **Declarative Compositions are library-only.** The server has no endpoint for them and the registry stores only `Wasm | Program` artifacts.
- **Scaffolding in the contract crate:** `mosaic_core::{input, unit, output, propagate}` are empty modules carrying only doc comments. The propagation mechanism is actually realized through the separate `run2d` ABI, not through these modules.
- **Browser Worker path is unexercised in a real browser.** `sandbox.ts` spawns `new Worker(new URL("./worker.ts", …))`, which needs a bundler to load `.ts`. The Node tests exercise an equivalent policy through `test/helpers/facet-worker.mjs` and `node:worker_threads`, not `sandbox.ts` itself.
- **Observed gap (not in the audit):** `verifyCertificate` replays probes *synchronously on the caller's thread* (`runFacetMap`), with no Worker or timeout. A Facet can be certified while trapping (for example, exhausting fuel) on some probes, as long as at least one probe produces tokens. Such a Facet would hang the browser verifier on the probe where the server recorded `Trapped`. `ui-builder-guide.md` recommends running this verifier for a "verified" badge.
- **Operational:** the server has no rate limiting, and token rotation is "edit the JSON file." Both are reasonable for a service with no UI and no deployment yet.

## README vs. code

The architecture record and audit are unusually rigorous and mostly accurate. The **root README has drifted** behind the code, and the drift mostly *under-claims*:

- **Test counts are stale.** The README says "127 Rust tests + 39 JS tests". The code has **224** Rust test functions and **60** JS test cases. Audit L9 flagged this same class of drift, and it has come back.
- **"`pnpm test` — all eight node --test suites".** There are 13 test files. The root `package.json` script runs **12** and omits `program_cert.test.ts`. CI runs all 13, and the `facet-abi` package script runs its 8. This is a regression of audit L8: the local command no longer matches CI.
- **"Regenerate and verify all five golden vectors".** `verify-fixtures.sh` regenerates and diffs **seven**.
- **The Status section contradicts itself.** It lists "the conformance gate, the registry (+auth), an authoritative server render endpoint" as *what's left*, while its intro and `crates/mosaic-server` show all three built. The repository-layout block also omits `mosaic-certify`, `mosaic-registry`, `mosaic-server` and `tools/mosaic-pixel`. The README cites decisions "D1–D16", but `architecture.md` runs to **D21**.
- **"Provably identical" (README tagline and GitHub description)** overstates what is shown. Equivalence is established by golden vectors and a seeded *sample* of probes, and `architecture.md` D15 itself says "not a proof over all inputs." The browser also cannot enforce NaN canonicalization or relaxed-SIMD rejection (D9 "known parity limits").
- **Stale metadata in other files:**
  - `SECURITY.md` puts "the not-yet-built registry and web UI" out of scope, but the registry and bearer auth exist.
  - The `mosaic-server` Cargo description says "(later) the Facet registry"; the registry is built.
  - The `tessera-ascii` Cargo description says "(L0 luminance slice)"; the crate actually has L0, L1, L2, dithering, braille and colour.
  - `architecture.md` lists `verifyCertificate` under `mosaic-wasm`, but it lives in the TS `@mosaic/facet-abi`.
  - `architecture.md` D2 says TypeScript/Next.js "owns … the registry", but the registry is Rust.
  - `architecture.md` D8 calls `facet-ramp` "499 bytes"; the committed binary is 535 bytes.
- **Accurate claims, spot-checked against code:**
  - zero-import instantiation;
  - fuel plus epoch metering, and every disabled wasm feature;
  - the same Facet binary rendering image and audio (`sandboxed_spectral.rs`, 32-signal sweep);
  - 64-image native≡sandboxed sweeps;
  - CI rebuilding Facets from source and diffing the committed `.wasm`;
  - the pinned toolchain, `--locked` and `cargo audit`;
  - every audit fix cited in commit messages (C1 epoch, H6 `MAX_WINDOW`, M1 f64 Goertzel, M12 `MAX_LAYERS`, M5 ramp index, and the others).

---

*Doc generated 2026-10-12 by reading the source at C:\Users\ameya\Documents\Mosiac. Source of truth: https://github.com/AmeyaBorkar/mosaic*
