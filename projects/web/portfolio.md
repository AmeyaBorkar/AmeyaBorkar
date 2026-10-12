# portfolio (AX-01)

> Ameya's personal portfolio, live at **ameyaborkar.in**, built as one scroll-driven pixel-art rocket launch. The page is a single long scroll from the launch pad past the Kármán line to deep space, and **altitude is time**: each `<section class="act">` declares the altitude it ends at, so scrolling further down the CV also means flying higher. Behind the DOM sits a hand-written **software framebuffer** (`Uint32Array` at 1/3–1/5 screen resolution, nearest-neighbour upscaled) with 8×8 Bayer dithering, 17 character-map sprites, a Lambert-shaded Earth that goes from horizon to pale blue dot, and an "orbital field" where the six featured projects are satellites you can click. It is vanilla HTML/CSS/ES modules with no framework, no `package.json`, no build step and no image files, deployed as static files on Vercel behind a strict CSP. A 73-assertion headless-Chrome (CDP) test harness ships alongside it.

| | |
| --- | --- |
| **Repository** | https://github.com/AmeyaBorkar/portfolio |
| **Visibility** | Private |
| **Category** | web |
| **Primary language(s)** | JavaScript (vanilla ES modules), CSS, HTML. No TypeScript, no framework. |
| **Local path** | `C:\Users\ameya\Documents\PortfolioWebsite\NewPortfolio` |
| **Default branch** | `main` (tracks `origin/main`, in sync) |
| **Lines of code (computed)** | **3,933 lines of site source** (JS 2,619 across 7 modules · CSS 766 · HTML 548) + **721 lines of test harness** (10 `.mjs`) = **4,654 LOC**. Plus 248 lines of Markdown (2 READMEs) and 78 lines of config/text. Largest modules: `main.js` 955, `scene.js` 823, `engine.js` 227, `rocket.js` 203, `sprites.js` 202, `payloads.js` 128, `palette.js` 81. |
| **Source files (computed)** | 29 tracked files: 9 site source (7 JS, 1 CSS, 1 HTML), 10 test scripts (7 suites + runner + 2 screenshot tools), 2 Markdown, 6 config/text (`vercel.json`, `robots.txt`, `.editorconfig`, `.gitattributes`, `.gitignore`, `.vercelignore`), 1 PDF (the résumé), 1 `.gitkeep`. **0 image files** (no png/jpg/gif/webp/svg/ico tracked). No `package.json`, no `node_modules`. |
| **Host platform** | Vercel (static, no build). Domain `ameyaborkar.in` (GoDaddy DNS per README). Verified live 2026-10-12: apex returns **308 → `https://www.ameyaborkar.in/`**, which serves 200. |
| **Key dependencies** | **Runtime: none.** The only third-party fetch is Google Fonts (Silkscreen + VT323). **Tests:** Node ≥ 22 (global `fetch` + `WebSocket`, no npm packages) and a local Chrome (default path hard-coded to the Windows install). Local serving: any static server (README uses `python -m http.server`). |
| **License** | None declared (no `LICENSE` file). The repo is private. |
| **Last commit** | `979a6e0`, "Replace the contact form with a plain address" (2026-09-05). First commit `4f56e16`, "AX-01: pixel-art scroll-driven launch portfolio" (2026-09-01). **14 commits total**: 12 on 2026-09-01, 2 on 2026-09-05. |

> Working-tree note: the tree was clean when reading started. During the read, `AmeyaBorkar_Resume.pdf` was overwritten by something outside this session (mtime 2026-10-12 19:48:49 IST; 60,850 → 82,793 bytes), so it now shows as modified and uncommitted. The committed and live PDF is still the 60,850-byte version. Nothing else changed.

---

## What it actually is (be concrete)

This is a single-page portfolio whose whole concept is the scroll axis. `index.html` contains **10 acts**, and each one declares its flight-plan data in attributes: `data-label` (rail label), `data-alt` (altitude in km at the act's end), `data-side` (which side the text column sits on) and `--vh` (scroll runway in viewport heights). `main.js` reads these at boot. The altitude rail, the telemetry readout, the sky colour, Earth's curvature, which side the vehicle flies on, and when stage separation or the Kármán-line flash fire all come from those numbers.

| # | Section id | Rail label | `data-alt` (km) | `--vh` | Content |
| --- | --- | --- | --- | --- | --- |
| 0 | `act-pad` | PAD | 0 | 1.5 | Hero ("AMEYA BORKAR"), press-and-hold LAUNCH button |
| 1 | `act-ignition` | IGNITION | 2 | 1.6 | "TOWER CLEARED" one-paragraph intro |
| 2 | `act-origins` | ORIGINS | 12 | 2.6 | Background + education ticks |
| 3 | `act-maxq` | MAX Q | 20 | 2.0 | Alarm panel that must be ACKNOWLEDGED; the PROMETHEUS "the part that didn't work" story |
| 4 | `act-flight` | EXPERIENCE | 85 | 3.6 | 6 collapsible "beacons" (roles), oldest at the bottom |
| 5 | `act-karman` | KÁRMÁN | 100 | 1.7 | "100 km · YOU ARE NOW IN SPACE" card |
| 6 | `act-research` | RESEARCH | 300 | 2.2 | 2 papers (ICAIN 2025 Springer; ICICI 2026 in press) + 1 filed patent |
| 7 | `act-payload` | PROJECTS | 1,400 | 3.4 | 6-payload manifest + canvas orbital field + dossier panel |
| 8 | `act-deep` | NOW | 60,000 | 2.2 | "currently", pale-blue-dot button |
| 9 | `act-signal` | CONTACT | 400,000 | 1.9 | Static email + COPY, link "frequencies", DOSSIER-only keyword list, RE-ENTRY |

On desktop that adds up to about **22.7 viewport heights** of runway. Desktop panels are `position: sticky` at mid-viewport, so the world scrolls past while the text stays put. On phones (≤ 900 px) the panels flow normally instead.

The six projects (`<article class="mf">`, the single source of truth for the manifest, canvas field and dossier):

| Id | Name | Shell (`data-shell`) | Sprite (`data-art`) | Patch (`data-emblem`) |
| --- | --- | --- | --- | --- |
| SAT-01 | VeridianOS | GEO · flagship | `module` | `shield` |
| SAT-02 | ThrottleKit | GEO · flagship | `bus` | `gauge` |
| SAT-03 | Engram | MEO · shipped | `probe` | `orbit` |
| SAT-04 | proximap | MEO · shipped | `scope` | `grid` |
| SAT-05 | GridWatch | LEO · for fun | `cubesat` | `signal` |
| SAT-06 | BankSystem | LEO · for fun | `lander` | `rocket` |

All real content stays in the DOM, where it is selectable, indexable and screen-readable. The `<canvas id="stage">` is `aria-hidden` and purely atmospheric. Everything drawn on it is either duplicated in the DOM or decorative.

## Architecture & how it's structured

There is no bundler and no package manager. `index.html` loads one `<script type="module" src="js/main.js">`, which imports the other six modules. Annotated tree:

```
NewPortfolio/
├─ index.html          548 lines. All copy and structure; the 10 <section class="act"> ARE the flight plan.
│                      Inline-SVG pixel art for the boot rocket, HUD patch, favicon (data: URI).
├─ css/style.css       766 lines. "Flight deck" design system: :root tokens, notched clip-path panels,
│                      hard drop-shadows, sticky act panels, DOSSIER mode, ≤900px phone layout,
│                      prefers-reduced-motion rules, pointer-events "hit routing" to the canvas
├─ js/
│  ├─ palette.js        81  PAL (43 named hex colours) + abgr() packer + pre-packed C.* + mix()
│  ├─ engine.js        227  framebuffer G, px/pxA/rect/rectA/disc/ring, 8×8 Bayer + ditherBand,
│  │                        makeSprite, rotatedSprite (64 steps, cached), drawSprite, math, xorshift rng
│  ├─ sprites.js       202  11 char-map sprites: ROCKET 20×44, BOOSTER, GANTRY, birds, PLANE,
│  │                        BALLOON, SATELLITE, SAT_DISH, DEBRIS, METEOR
│  ├─ payloads.js      128  6 project-satellite sprites + ARRAY_ROWS (solar-array rows for the glint)
│  ├─ scene.js         823  sky, stars, aurora, clouds, ground/pad, Earth, ambient traffic, orbital
│  │                        hardware + Moon, warp streaks, hotspot brackets, payload orbital field
│  ├─ rocket.js        203  the vehicle, exhaust particle system, stage separation, screen shake, throttle
│  └─ main.js          955  state S, act parsing, scroll→altitude model, render loop, telemetry, audio,
│                           launch, warp, skill cross-reference, hit-testing, payload dossier,
│                           mission-log console, DOSSIER/audio toggles, boot sequence
├─ AmeyaBorkar_Resume.pdf   the one binary asset (opened by `resume` / the DOSSIER link)
├─ vercel.json         cleanUrls + security headers (CSP etc.) + cache rules
├─ robots.txt          Allow: / for everyone, explicitly including GPTBot/ClaudeBot/CCBot/Google-Extended
├─ .vercelignore       keeps test/, *.md, .editorconfig out of the deploy
├─ .gitattributes      force LF; *.pdf/*.png/*.ico binary
└─ test/
   ├─ run.mjs          runner: checks the server is up, spawns the 7 suites, tallies PASS/FAIL lines
   ├─ probe · launch · feat · log · pay · cmds · touch .mjs   the 7 CDP suites (73 assertions)
   ├─ shoot.mjs · mob.mjs   screenshot tools (desktop / phone), not part of the run
   ├─ shots/.gitkeep   screenshot output dir (PNGs gitignored)
   └─ README.md
```

**Module graph:** `main → engine, scene, payloads, rocket`. `scene → palette, engine, sprites`. `rocket → palette, engine, sprites, scene` (it imports `groundY` so the vehicle sits on the pad slab). `sprites`/`payloads → palette, engine`. State lives in one object `S` in `main.js`. Two module-level singletons are shared across files: `HOTSPOTS` (rebuilt each frame in `scene.js`) and `EARTH_GEOM` (published by `drawEarth` so the payload field can orbit the planet that was actually drawn).

**Frame order** (`main.js frame()`): sky → stars → aurora → Earth → ground → clouds → ambient traffic → orbital hardware → payload field → exhaust plume → ship → booster → hotspot brackets → warp streaks → `flip()` (one `putImageData`), then the DOM telemetry, panel cross-fade and audio rumble are updated.

## Code walkthrough (the important parts)

**Framebuffer engine (`engine.js`).** `resize()` chooses an integer pixel scale from the viewport width: **3** below 700 px, **4** below 1500 px, **5** otherwise. The canvas is then sized to `ceil(innerWidth/scale) × ceil(innerHeight/scale)` and stretched back to full size with CSS (`image-rendering: pixelated` + `imageSmoothingEnabled = false`). At 1920×1080 that is a 384×216 buffer (≈ 83k pixels), at 1440×900 it is 360×225 (81k), and on a 390×844 phone 130×282 (≈ 37k). `G.buf` is a `Uint32Array` view over the `ImageData` bytes, so every primitive writes one packed little-endian ABGR word (`palette.abgr()` does the packing). Primitives: `px`, `pxA` (per-pixel alpha blend against the existing word, decomposed into channels), `rect`/`rectA` (row `fill`), `disc` (filled circle by row spans), `ring` (midpoint circle). `flip()` is the single `putImageData` per frame. `rng(seed)` is a 32-bit xorshift, so the stars, clouds, cities and landmasses come out the same on every load.

**Ordered dithering.** `BAYER` is the classic 8×8 matrix normalised to [0,1). `ditherBand(y0, y1, cA, cB, tAt)` fills whole rows, choosing `cB` when `t > BAYER[(y&7)*8+(x&7)]` and `cA` otherwise. That gives a two-colour ordered dither with no intermediate colours, which is what makes the gradients band like hand-made pixel art. `drawSky(alt)` interpolates a **6-stop sky** across **9 altitude keyframes** (0, 3, 11, 26, 55, 98, 300, 4,000, 200,000 km) with smoothstep. It also moves the band boundaries from `LOW` to `HIGH` between 8 and 110 km, so the warm horizon shrinks to a thin limb as the atmosphere thins. The same Bayer threshold is reused for the aurora's alpha cutoff, the 5-step ocean ramp on Earth, and the polar-ice edge.

**Sprites (`sprites.js`, `payloads.js`).** `makeSprite(rows, key)` turns an array of strings into `{w, h, cols: Uint32Array}`. Each character maps through a per-sprite key to a packed palette colour, and `.`/space are transparent (stored as 0). There are 17 sprites in total. The rocket is 20×44, lit from the upper left via a `D L L L W W M M S S S D` column pattern that reads as a cylinder. The six payload sprites run from 18×14 (lander) to 32×13 (station module). `drawSprite` supports integer `scale`, `alpha`, `flipX` and `tint`. **`rotatedSprite`** never rotates smoothly. It quantises the angle to `ROT_STEPS = 64`, inverse-maps each destination pixel of a `ceil(hypot(w,h))`-square back into the source with nearest-neighbour sampling, and caches the result per sprite per step in a `WeakMap`, which behaves like a pre-drawn sprite sheet. The rocket, the tumbling booster and the debris all go through it. Separately, the DOM uses inline `<svg shape-rendering="crispEdges">` built from `<rect>`s for the boot rocket, the HUD patch, the favicon and the six mission-patch emblems (`EMBLEM` in `main.js`). These are vectors, not image files.

**Scroll model: altitude is time (`main.js`).** Each frame:
`pRaw = scrollY / (scrollHeight − innerHeight)`. `S.vel` is a smoothed `|Δp|/dt` ("scroll energy"). `S.pSmooth` follows `pRaw` exponentially at rate 8/s, or 1/s under `prefers-reduced-motion`. `altitudeAt(pSmooth·max)` takes the viewport **midpoint**, finds the act containing it, and smoothstep-interpolates from the previous act's `data-alt` to this act's. The result is a piecewise, monotonic map from document position to kilometres, with the readout speeding up through each act. `currentAct()` uses a 45% line to highlight the rail and, on the first move past act 0, auto-launches (`doLaunch(true)`), so scrolling alone starts the flight. Desktop panels cross-fade via a `--vis` custom property. The outgoing panel reaches 0 before the incoming one starts (fade windows at 14–42% of a viewport inside each act), so two panels are never legible at once.

`scene.js` separates the two axes on purpose. **Positions** of clouds, birds, balloons and orbital hardware come from scroll progress `p` times a per-object depth (`SPAN = 26` screen heights per unit of `p`). **Altitude** only decides colour, density and which layers exist. The exception is the ground: `groundY(alt)` is tied to altitude so the pad holds perfectly still for all of act 0.

**Flight dynamics.** `camPull` eases from the close pad camera (CAM 01) to the tracking camera (CAM 02) between 0.15 and 2 km. Gravity turn: `pitch = smooth(inv(36, 130, alt)) × 1.30 rad` (≈ 74.5°). Camera roll: `roll = smooth(inv(118, 520, alt)) × 1.30`. The drawn attitude is `pitch − roll`, so the rocket pitches over and then reads upright again once the camera has rolled to match. The starfield (1,100 stars) and Earth rotate by `roll`. The stars live in a square as wide as the frame's diagonal, so rotation never exposes empty corners. Throttle is 0 before launch, 1 while igniting (< 3 km) or warping, 0.30 while "reading" (a beacon or dossier is open), and otherwise `clamp(0.36 + vel·6)`, so the engine flares while you scroll and idles while you read. One-time beats: **stage separation at 22 km** (the booster tumbles away and can be clicked to "RECOVER"), and the **Kármán line at 100 km** (white flash, toast, log line). Meteors spawn between 55 and 1,200 km, at most 3 at a time.

**Earth (`drawEarth`).** The radius is driven by `log10(alt)`. It goes from `6×W` (an almost flat horizon) to `0.46×W` (a whole ball), then down to `0.035×W` between about 2,500 and 250,000 km (the pale blue dot). Shading is **Lambert off a true sphere normal** `nz = √(1−u²−v²)` with a fixed upper-left light, quantised down a **5-step ocean ramp** (`night → seaD → sea → seaL → seaB`) with Bayer dithering between steps. On top of that:
- polar ice where `|lat| > 0.905` against a tilted pole vector;
- 34 landmasses, each a union of 3–5 discs, with "arid" ones in sand tones;
- 46 city-light clusters drawn only where Lambert < 0.05 (the night side);
- 16 high-cloud wisps drawn only in sunlight;
- an atmosphere-glow annulus.

Close in, where only the top sliver of the sphere is visible, it cross-fades to depth-from-rim banding ("horizon mode"). The light and pole are fixed in *frame* space rather than rotating with the camera; the comment explains that rolling them swung the terminator off-screen.

**Payload field (`drawPayloads`).** Satellites fly on shells concentric with `EARTH_GEOM` at `R × {leo 1.13, meo 1.30, geo 1.49}`. `visibleArc()` samples 128 points around each shell and returns the widest contiguous run that is on stage (the left 60% of the frame, clear of the content column) **and** outside the rocket's bounding box. Each shell's two payloads are spread over 16–84% of that arc with a slight drift. When one is selected, it lerps to `(0.42W, 0.30H)` and steps to integer scale 2, the others dim to 22%, and the panel switches to a dossier: patch SVG, orbit note, and the `.mf-body` HTML copied across. Other details: dotted shell arcs, a 12-pixel orbit trail, a sunlight glint that sweeps the rows listed in `ARRAY_ROWS`, a blinking station-keeping beacon, and a hover leader line. On mobile (`G.mobile`, width < 900) the field is not drawn, and the manifest rows become the interface.

**Rocket & exhaust (`rocket.js`).** The world moves past a vehicle that stays roughly in place. `place()` lerps from standing on the slab to a flight position at 44% of frame height (24% on mobile), and the vehicle stays at scale 2 throughout. Exhaust is a particle system: cap of 260 particles, spawn rate `throttle × 470/s` from the rotated nozzle, spread that "blooms in vacuum" between 20 and 70 km, and a 7-step colour ramp from `hot5` (white-hot) down to `cloudS` (smoke), plus a bright core cone along the tail axis. A faint orange compression-heating box appears between 7 and 19 km. Screen shake is a decaying random offset applied as a CSS `transform` on the canvas.

**Interaction layer.** `HOTSPOTS` is cleared and rebuilt every frame by whatever got drawn, and `hitTest` pads every target to at least 44 CSS px. There are 8 canvas hotspot kinds: bird (scatter), plane (hail), weather balloon (release a personal fact), meteor (catch; **5 unlocks FTL** star-streaking), booster (recover), orbital relay/satellite/debris (hail), payload (open), Moon (range, past 3,000 km). `drawHints` gives every hotspot a quietly pulsing corner bracket that brightens on hover, and a DOM `#reticle` shows the verb. Clicking the rail **warps**: full throttle, star streaks along the flight axis, and a `behavior: 'instant'` jump 240 ms later, under cover of the effect.

**HUD & telemetry.** Fixed elements: the patch, the name card, three tools (DOSSIER / LOG / AUDIO), the altitude rail (built from `data-label`), and the telemetry block (ALT km, VEL m/s, THR %, MET, a progress bar and the camera label). Velocity is admitted in the source to be "honest-ish": `lerp(0, 7800, smooth(inv(0, 400, alt))) + vel·900`. MET counts from launch. **Audio** is off by default and uses no files. A 2 s white-noise buffer goes through a 180 Hz low-pass into a gain of `throttle × 0.16 × air`, where `air` fades to 0 between 60 and 100 km ("sound stops here"). UI blips are square, sawtooth or sine oscillators.

**Mission-log console.** Opened with `` ` ``/`~` or the LOG button. An unread badge (capped at "99+") counts lines logged while it is closed. The command path is `runCommand(raw)`, shared by typed input and the 6 command chips:

| Command | Effect |
| --- | --- |
| `help` | prints the 10-line help |
| `projects` / `projects <id\|name>` | lists the 6 payloads; with an argument it matches the id, an id suffix (`06`) or a name prefix (`engram`), warps to PROJECTS and opens that dossier |
| `resume` (alias `cv`) | logs a link and `window.open`s the PDF |
| `contact` | prints clickable mailto / GitHub / LinkedIn links |
| `goto <act>` | warps to `act-<arg>`; valid args are the section ids `pad ignition origins maxq flight karman research payload deep signal` |
| `dossier` · `launch` · `clear` · `whoami` · `abort` | toggle DOSSIER · ignite · wipe log · bio line · "not on my watch" + shake |

Tab completes a unique verb prefix (or lists the candidates), and ↑/↓ recall history. `open` is an undocumented alias of `projects`. User input is echoed with `textContent`, never `innerHTML`, and a test asserts that `<img src=x onerror=…>` does not create a node. `Esc` closes one layer per press, in order: dossier → skill query → console → open beacons.

**Skill cross-reference.** There is no skills section. Instead there are 28 `.chip` buttons covering 20 distinct skills; `python` is the most referenced (4 chips). `query(id)` lights every `.beacon` that contains that chip and every `.mf` whose `data-skills` includes it, then shows a bar such as `PYTHON — 3 ROLES · 1 PAYLOAD`.

**DOSSIER mode** is the accessibility and recruiter escape hatch. It toggles `body.dossier`, which hides the canvas, rail, telemetry and reticle, removes the sticky/clip-path styling, expands every beacon and manifest body, and reveals the `.kw-strip` keyword list. The render loop short-circuits while it is on.

**Contact & boot.** The contact form was removed in the last commit. The address is plain text with a mailto link and a COPY button: `navigator.clipboard.writeText`, falling back to selecting the text and showing a ⌘C/Ctrl+C hint. The boot screen is a 5-item pre-flight checklist that clears at 130 ms intervals while an SVG rocket climbs a dashed rail as the progress bar.

**Dev hook.** `window.__S`, `__HS` and `__G` are exposed only when `location.hostname` is `127.0.0.1` or `localhost`. The tests depend on them.

## Core features

- Scroll-driven flight from 0 to 400,000 km across 10 acts. Altitude, sky, Earth, camera, side swing and throttle are all derived from the HTML.
- Press-and-hold launch: the first press flips a safety cover, then a 1.5 s wall-clock hold ignites (measured in wall-clock time, not frame count). Releasing early aborts, and scrolling also launches.
- Gravity turn plus camera roll, stage separation, Kármán-line flash, meteors and FTL mode, the Moon, and orbital relays/debris with messages.
- Projects as an orbital field with mission-patch dossiers and LEO/MEO/GEO filter tabs (`role="tablist"`).
- Mission-log console with 10 verbs, chip shortcuts, Tab completion and history.
- Skill-chip cross-referencing across roles and projects.
- DOSSIER plain-résumé mode, optional synthesized audio, and a phone layout with 40–54 px tap targets and `env(safe-area-inset-*)` padding.

## Tech stack & deployment

- **Stack:** HTML5 + CSS (custom properties, `clip-path` notches, `position: sticky`) + vanilla JS ES modules, plus Canvas 2D (`putImageData` only) and Web Audio. Two typefaces, Silkscreen (UI) and VT323 (body), from Google Fonts. No build tooling, no `package.json`, no cache-busting query strings.
- **`vercel.json`:** `cleanUrls: true`, `trailingSlash: false`. All routes get `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()`, and this CSP:
  `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src https://fonts.gstatic.com; img-src 'self' data:; connect-src 'self'; form-action 'self'; base-uri 'self'; frame-ancestors 'none'; object-src 'none'`.
  Cache rules: `/css/*` and `/js/*` get `max-age=600, stale-while-revalidate=86400`, and the résumé PDF gets `max-age=3600`.
- **CSP vs. code:** the policy fits the code closely. There are no inline scripts, no `eval`, and one same-origin module entry point. `style-src 'unsafe-inline'` is needed because every act carries an inline `style="--vh:…"`. `img-src data:` covers the SVG data-URI favicon. `connect-src 'self'` is more than enough: the JS makes **no network requests at all** (no `fetch`, no storage APIs). `frame-ancestors 'none'` does the job of `X-Frame-Options`.
- **Live verification (2026-10-12):** `https://ameyaborkar.in` → **308** → `https://www.ameyaborkar.in/` → **200**, `Server: Vercel` (edge `bom1`).
  - The served CSP, Permissions-Policy, Referrer-Policy and nosniff headers match `vercel.json` byte for byte. HSTS (`max-age=63072000`) is also present, coming from Vercel's default rather than `vercel.json`.
  - The live `index.html` and `js/main.js` are byte-identical to HEAD. `js/engine.js` is identical apart from line endings (the local checkout has CRLF).
  - `/README.md` and `/test/run.mjs` return 404, so `.vercelignore` works.
  - The live résumé is the committed 60,850-byte PDF.
- **SEO/meta:** title "ameya builds things", a description, `canonical` + `og:url` = `https://ameyaborkar.in/`, and `og:title`/`og:description`/`og:type`/`og:site_name`. There is no `og:image` (acknowledged in the README), no Twitter card, no JSON-LD and no sitemap. `robots.txt` explicitly *allows* AI crawlers (GPTBot, ClaudeBot, PerplexityBot, Google-Extended, CCBot), which is the opposite of Elegant-Landscape's block-the-trainers policy.

## Build / dev / test (actual commands)

```bash
# serve (ES modules need http://, not file://) — no install, no build
python -m http.server 8123            # then open http://127.0.0.1:8123

# tests: real headless Chrome over the DevTools Protocol (Node ≥ 22, no npm deps)
node test/run.mjs                     # with the server above running; all 7 suites
CHROME_PATH=/path/to/chrome node test/run.mjs
BASE_URL=https://www.ameyaborkar.in/ node test/run.mjs   # see caveat under Tests

# auxiliary screenshot tools (not part of the run)
node test/shoot.mjs <url> <outdir> '[[0.5,"name"]]'
node test/mob.mjs <outdir> <fraction> <name>

# deploy: push to main; Vercel serves the repo root as-is (Framework: Other, no build)
```

## Tests

**7 suites, 73 assertions, all integration tests through real headless Chrome over raw CDP WebSockets.** There is no jsdom and no unit-test framework. Each suite launches its own Chrome on a separate debugging port (9333–9344), disables the HTTP cache, sets a 1440×900 or 390×844 viewport, and prints `PASS`/`FAIL` lines. `run.mjs` counts those lines and also fails on `EXCEPTION`/`THREW`.

| Suite | Assertions | Covers |
| --- | --- | --- |
| `probe` | 17 | beacons, skill query (python → 3 roles · 1 payload), LEO filter, manifest dossier, MAX Q, console + **XSS echo check**, scroll auto-launch, DOSSIER mode, mobile horizontal overflow |
| `cmds` | 16 | `contact`/`projects`/`projects engram`/`projects 06`/`resume`, Tab, help, chips, static contact block, **real clipboard read-back** |
| `pay` | 13 | 6 payloads on station, hotspots, canvas-click → dossier, patch SVG, focus, shell filter, manifest-row parity |
| `launch` | 8 | cover arming, hold progress, early-release abort, full-hold ignition, MET |
| `feat` | 8 | upright at p=0.20, pitched > 0.6 rad at 0.52, rolled back < 0.25 at 0.80, 3 distinct sides, rail warp, orbital hotspots |
| `log` | 7 | unread badge, chips, Tab, ↑ history, bad-`goto` suggestions |
| `touch` | 4 | 3 synthetic touch drags + 1 mouse-wheel gesture on a phone viewport; the page must move > 40 px |

**Static count:** 17+16+13+8+8+7+4 = **73**, matching the README. **Executed for this doc** (Node 24.13.1, Chrome headless, `python -m http.server`, screenshots redirected to a scratch directory): **73/73 PASS, 0 exceptions.** Against production via `BASE_URL`: **54 pass, 8 fail, 11 never run.** `feat`, `pay` and `cmds` read `window.__S/__HS/__G`, which only exist on localhost, and `pay` aborts after two failures. The tests are timing-based (fixed `sleep`s) and default to the Windows Chrome path. The engine maths (dither, rotation, Earth shading) has no unit tests; the README says screenshots (`shoot.mjs`, `mob.mjs`) serve as the visual check. The commit history shows the harness growing with the fixes: 64 assertions / six suites → 68 / seven → 73 / seven.

## Status, completeness & notable gaps

- **Complete and live.** Built in a concentrated burst: 12 commits on 2026-09-01 (prototype → flight dynamics → realistic Earth → payload lab → orbital field → real content → deploy prep → phone scrolling), then 2 on 2026-09-05 (canonical domain, removal of the contact form). No changes since. The deployed site matches HEAD.
- **Reduced motion is only partly handled.** `REDUCED` disables the flash, warp and smooth scrolling, and the CSS stops blink/pulse/toast/boot animations. The canvas still animates continuously (twinkle, plume, aurora) and **screen shake still fires**. DOSSIER is not switched on automatically.
- **Canvas toys are pointer-only** (the README acknowledges this). Their content is mirrored in the DOM, but the objects themselves cannot be reached with Tab.
- **Canonical/redirect mismatch.** `canonical` and `og:url` name the apex `https://ameyaborkar.in/`, but Vercel's primary domain is `www`, so the canonical URL itself 308-redirects. Making `www` the canonical, or the apex the primary domain, would fix it.
- **"Edit the HTML, no JS to touch" is mostly true.** Rail, altitude, sky and sides do follow the attributes. But the `goto` help/suggestion list (`ACTS_HELP`) is a hard-coded string, `act-payload` and `act-ignition` are referenced by id, and ambient/orbital objects (balloons at p≈0.21–0.29, relays at p 0.60–0.89, the Moon at 0.86) are placed by **global scroll fraction**, not by act. Adding or resizing acts would move them; a `scene.js` comment already warns about this for the station flyby.
- **Colour is defined in several places** (see README vs. code), and `scene.js` has its own `pack()` that duplicates `palette.abgr()`.
- **Continuous redraw.** The full framebuffer is redrawn every rAF tick even when idle (the cost is roughly 80k pixel writes plus overdraw). It is cheap at this resolution, but there is no idle throttling or `visibilitychange` handling beyond what the browser does itself.
- **No `og:image`**, which the README lists as a to-do.
- **Content note (not code):** the ORIGINS act lists "cgpa 8.94". Ameya's older profile docs list 9.05, so it is worth reconciling whichever is current.

## README vs. code

Accurate in most respects. Verified true:
- no framework, no build step, no image files (0 tracked images; the only binary is the résumé PDF);
- a `Uint32Array` framebuffer blitted once per frame and nearest-neighbour upscaled;
- 8×8 Bayer dithering;
- sprites as hand-authored character maps in `sprites.js`/`payloads.js`;
- 64-step cached nearest-neighbour rotation;
- a diagonal-square starfield;
- Lambert shading down a five-step ocean ramp, with polar ice, arid belts, sunlit-only cloud and night-side city lights;
- the gravity turn over 36–130 km and camera roll over 118–520 km (~75°);
- six payloads, two per shell, others dimmed to 22%;
- the field disabled on phones;
- the 1.5 s hold and the 5-meteor FTL unlock;
- the DOSSIER `.kw-strip`;
- the CSP exactly as described (and served live);
- `.vercelignore` excluding `test/` and Markdown;
- **"73 assertions across seven suites"**: counted statically and run, 73/73.

Discrepancies and nuances:
1. **Framebuffer size.** The README says "≈288×180 … roughly 50k pixel writes per frame." The scale is 3/4/5 by width (< 700 / < 1500 / ≥ 1500 px), so 1440×900 gives 360×225 (81k pixels) and 1080p gives 384×216 (83k). `engine.js`'s own header says "~130k pixel writes/frame at 1080p." 288×180 would only occur at scale 5 on a 1440×900 screen, which the code never selects. The order-of-magnitude point (versus ~2M) still holds.
2. **"Every colour in the project comes from here. Nothing else."** (`palette.js`, which also says "32 colours"). `PAL` has **43** entries (41 distinct: `glass` = `sig`, `city` = `hot4`). Colours are also hard-coded in `scene.js` (54 sky-keyframe hexes), `main.js` (6 mission-patch field colours), `style.css` (19 re-declared tokens, a literal `#141a2e` and rgba panel backgrounds) and the favicon SVG.
3. **`data-label` "and `goto origins` in the console".** `goto` actually uses the section **id** (`act-<arg>`), not the label. For 4 of the 10 acts they differ: EXPERIENCE is `goto flight`, PROJECTS is `goto payload`, NOW is `goto deep`, CONTACT is `goto signal`.
4. **DOSSIER "also covers `prefers-reduced-motion`".** It is only a manual toggle. Under reduced motion the canvas keeps animating and screen shake still fires (see Status).
5. **`touch.mjs` "at three points."** It runs three touch drags plus a fourth mouse-wheel gesture, and all four count among the 73. Its exit code checks only two of them.
6. **`test/README.md` says "Six suites"** while its own table and `run.mjs` list seven. The header was not updated when `touch` was added (`fd22770`).
7. **"`BASE_URL` points the run at a deployed URL."** Mechanically true, but against production 8 assertions fail and 11 never run because the inspection hook is gated to localhost (see Tests).
8. **"Esc closes everything."** It closes one layer per press, in priority order.
9. **Stale "four" comments.** `payloads.js` ("the four project satellites"), `main.js` ("the same four `<article class="mf">` elements") and `scene.js` ("three of four payloads") date from the "four working prototypes" commit. The code now has six.
10. **CSS header rules.** "No gradients" and "every measurement on a 4px grid" are aims rather than strict facts. There are two `repeating-linear-gradient` hard-stop stripe patterns (the launch cover and the boot rail), and many 3/5/7/9 px values. "No border-radius" and "no blur" do hold.
11. **"Served at ameyaborkar.in."** True via a 308 to `www.ameyaborkar.in`, which is the address that actually serves the page (see the canonical note above).

---

*Doc generated 2026-10-12 by reading the source at C:\Users\ameya\Documents\PortfolioWebsite\NewPortfolio. Source of truth: https://github.com/AmeyaBorkar/portfolio (private) · live at https://ameyaborkar.in*
