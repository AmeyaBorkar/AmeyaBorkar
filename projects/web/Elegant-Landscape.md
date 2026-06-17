# Elegant Landscape

> A live, single-page marketing/portfolio site for a Pune-based landscape architecture studio (elegantlandscape.in). Built in vanilla HTML/CSS/JavaScript with no framework, no package manager, and no build step. The hero is a scroll-driven `<canvas>` animation that scrubs a 192-frame WebP image sequence as the user scrolls, with staged text overlays; below it sits a static magazine-style site (About / Services / Projects / Contact) plus six dedicated case-study pages with lightboxes. Deployed on Vercel with a hardened security-header config.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/ElegantLandscape |
| **Visibility** | Private |
| **Category** | web |
| **Primary language(s)** | HTML, CSS, JavaScript (vanilla, zero framework) |
| **Local path** | `C:\Users\ameya\Documents\ElegantWebsite` |
| **Default branch** | `main` |
| **Lines of code (computed)** | ~4,740 lines of source (HTML 2,098 across 10 files · CSS 2,054 in one file · JS 588 across 2 files). Plus ~340 lines of config/text (vercel.json 105, robots.txt 139, sitemap.xml 57, llms.txt 39). |
| **Source files (computed)** | 10 HTML, 1 CSS, 2 JS = 13 hand-written code files. 258 files tracked in git total (the rest are 192 WebP frames + 38 project images + 1 og-image.jpg). No `package.json`, no `node_modules`. |
| **Live URL** | https://elegantlandscape.in/ |
| **Hosting** | Vercel (static), domain via GoDaddy per `DEPLOYMENT.md` |
| **License** | Proprietary — "all rights reserved" (custom `LICENSE`); repo is *not* open source |
| **Last commit** | `e810af2` — "Bump cache-bust to v=6 after scroll-animation rollback" (2026-04-20) |

> Working-tree note: two untracked items at read time — `.claude/` and `ELEGANT LANDSCAPE.jpeg`. No staged/unstaged modifications to source files.

---

## What it actually is

A small static brochure site for **Elegant Landscape**, a landscape-architecture proprietorship led by Rohan Vivek Dalvi in Pune, Maharashtra. The site is a marketing/portfolio front for the studio: a long-scroll home page whose hero is a frame-by-frame canvas animation (a "garden revealed from macro" video pre-extracted to 192 stills), followed by a conventional content site, and six per-project case-study pages.

There is **no backend, no framework, no bundler, and no package manager** — verified: no `package.json` and no `node_modules` exist. Everything is plain files served statically. The only runtime "dependency" is a Google Fonts `@import` in `styles.css` (Cormorant Garamond + Inter); there are no JS libraries at all. The "zero build step" claim is accurate — the working tree *is* the deployed artifact, with cache-busting done by hand via `?v=6` query strings on the CSS/JS links.

The "192-frame" claim is **confirmed in code**: `script.js` declares `const TOTAL_FRAMES = 192`, and the filesystem holds exactly 192 frames each in `ImagesVideos/WebPFrames/` (`frame_0001.webp`..`frame_0192.webp`) and `ImagesVideos/ExtractedFrames/` (the PNG originals, gitignored). The runtime loads the **WebP** set (`FRAME_PATH = 'ImagesVideos/WebPFrames/frame_'`, `FRAME_EXT = '.webp'`).

---

## Architecture & how it's structured

**Multi-page**, but with one rich primary page and several lightweight secondary pages. All pages share one stylesheet (`styles.css`) and the shared `nav.js` hamburger script; only `index.html` loads the animation engine `script.js`.

Annotated working tree (assets collapsed):

```
ElegantWebsite/
├── index.html              Home — navbar + scroll-animation zone + static site (About/Services/Projects/Contact) + footer
├── project-1.html … project-6.html   Six case-study pages: hero + magazine content rows + lightbox gallery
├── 404.html                Custom not-found page (inline <style>, noindex)
├── privacy-policy.html      DPDP Act 2023 / IT Act privacy policy
├── terms.html              Terms of service
│
├── styles.css              2,054-line design system: tokens (:root vars), all sections, 6 @media blocks
├── script.js               559-line home-page engine: canvas frame animation, stages, project cards, nav active-state, counters
├── nav.js                  29-line shared mobile hamburger toggle (used on every page)
│
├── vercel.json             cleanUrls, redirects, security headers, per-asset cache rules
├── robots.txt              Allow search + AI answer-engines; block AI-training + SEO crawlers
├── sitemap.xml             9 <loc> URLs
├── llms.txt                llmstxt.org summary for LLM answer engines
├── og-image.jpg            1200×630 social card (tracked)
├── README.md, DEPLOYMENT.md, LICENSE
├── .gitignore, .vercelignore
│
├── ImagesVideos/
│   ├── WebPFrames/         192 .webp frames  ← loaded by the animation engine
│   ├── ExtractedFrames/    192 .png originals (gitignored, not deployed)
│   ├── Projects/           5 thumbnail PNGs (courtyard/garden/pool/rooftop/zen)
│   └── *.mp4, *.png        Source videos/renders (gitignored, not deployed)
│
├── Projects/Project1 … Project6/   Per-project render/plan images (38 tracked) used by project pages
└── Legal/                  Business-registration PII (PDFs) — gitignored AND vercelignored
```

Asset layout note: the `.gitignore` deliberately keeps the heavy PNG frames and source `.mp4`s out of git (only the optimized WebP frames ship), and `Legal/` (Aadhaar/PAN docs) is excluded from both git and deployment. `.vercelignore` additionally strips `README.md`/`DEPLOYMENT.md` from the deploy.

---

## The scroll-driven canvas animation — exactly how it works in code

All logic lives in `script.js` (an IIFE, `'use strict'`). The pattern is the classic "Apple AirPods-style" scroll-scrub: a single `<canvas id="frameCanvas">` is pinned (CSS-sticky) inside a tall `#scrollZone`, and the scroll position maps to a frame index that is drawn onto the canvas.

**Configuration (top of file):**
- `TOTAL_FRAMES = 192`; frames are `ImagesVideos/WebPFrames/frame_####.webp` (zero-padded to 4 digits via `padNum`).
- Device tiering: `IS_MOBILE = matchMedia('(max-width: 768px)')`. On mobile the scroll zone is `380vh` tall, desktop `600vh` (`SCROLL_HEIGHT_VH`). `MAX_DPR` is capped at 1 on mobile / 2 on desktop. `USE_BACKDROP_BLUR` is disabled on mobile.

**Preloading (not all 192 up front):**
- `loadFrames()` first loads only **key frames** — every 8th frame (`KEY_FRAME_STEP = 8`), plus frame 1 and 192 — in batches of 6 via `Promise.all`, updating the loading-bar width as `loadedCount / TOTAL_FRAMES`.
- Once key frames are in, it sets `animationReady = true`, hides the loading screen, draws frame 0, attaches the scroll listener, then **lazily fills the remaining frames in the background** in batches of 6. Each frame is an `Image()`; `onload` stores it into `frames[index-1]`, `onerror` still increments the counter so loading can never hang.

**Scroll → frame mapping:**
- `getScrollFraction()` computes `clamp(-rect.top / (scrollZone.offsetHeight - innerHeight), 0, 1)` from the sticky zone's bounding rect — i.e. how far through the tall zone you are (0→1).
- In `updateOnScroll()`: the animation is mapped to the **first 72%** of the scroll zone — `animFrac = clamp(scrollFrac / 0.72, 0, 1)` — and the frame drawn is `Math.round(animFrac * (TOTAL_FRAMES - 1))`. The last 28% of the zone is reserved for the blur/brand reveal overlays.

**Drawing (`drawFrame`):** clamps the index; if that exact frame isn't loaded yet it walks outward to the nearest loaded neighbor (graceful degradation during background fill); computes a cover-fit (`object-fit: cover` math) for the image against the DPR-scaled canvas; clears and draws; then paints a **radial vignette gradient** (transparent center → `rgba(0,0,0,0.45)` edge) on top.

**requestAnimationFrame:** yes — the scroll handler is rAF-throttled with a `ticking` flag (`startScrollListener`): on `scroll` (passive), if not already ticking it schedules one `requestAnimationFrame(updateOnScroll)`. This coalesces scroll events to one paint per frame. The same rAF-throttle pattern is reused for the navbar active-state listener. The stat counters also animate via a `requestAnimationFrame` count-up loop with cubic ease-out.

**IntersectionObserver:** used for the *static* part of the page (not for the canvas) — `initFadeInObserver()` and the project-card observer add `.in-view` when sections scroll into view; a third observer fires the About stat counters once. So: rAF drives the canvas, IntersectionObserver drives the static-section reveals.

**Staged overlays:** `STAGES` defines scroll-fraction windows for `intro` (0.00–0.08), `cards` (0.06–0.35), `info` (0.33–0.68), `blur` (0.72–1.00), `brand` (0.78–1.00). `updateStage` toggles a `.visible` class per stage based on `stageFraction`. The brand-reveal text ("Elegant Landscape") is built letter-by-letter in `initBrandText()`, each `<span>` given a staggered `transition-delay`. The blur stage interpolates `backdrop-filter: blur(0→30px)` and a cream overlay on desktop; on mobile it skips the expensive backdrop-filter and just fades the cream background opacity (commented as "10x cheaper on GPU").

**Resize handling:** `resizeCanvas()` rescales for DPR and re-`setScrollZoneHeight()`; a `resize` listener re-draws the current frame.

---

## Page sections & content

**Home (`index.html`):**
1. Scroll progress bar (fixed top) + loading screen with animated bar.
2. Glassmorphed floating navbar (logo + hamburger + 5 links: Home/About/Services/Projects/Contact) with scroll-based active highlighting.
3. **Scroll-animation zone** — sticky canvas with overlay stages: intro line + tagline "Where nature meets artistry"; three service teaser cards (Design / Craft / Nurture); info text "Where Ecology Meets Artistry"; scroll hint; brand reveal.
4. **About** — quote, metadata sidebar with animated count-up stats (`data-count` → 6 Landscapes, 3 Typologies), two prose paragraphs, three value pillars (Ecology-First / Refined Restraint / Site-Sensitive Craft).
5. **Services** — 6 cards: Landscape Design, Master Planning, Ecological Planting, Residential Gardens, Commercial & Hospitality, Hardscape & Art Integration.
6. **Projects** — grid populated by JS from the `PROJECTS` array (6 entries), each card clickable to its detail page, with 3-D mouse-tilt hover.
7. **Contact** — email/phone/studio channels and a mailto CTA.
8. **Footer** — brand, Explore/Legal/Contact link columns, © 2026.

**Project pages (`project-1.html` … `project-6.html`):** each is a self-contained magazine layout — full-bleed hero (image + type + title + location), a back-to-projects bar, alternating text/media content rows (including paired and full-width image rows), and a **lightbox** with prev/next, counter, caption, click-outside-to-close, and keyboard nav (Esc / ←/→). The lightbox JS is inlined per page (it indexes `.gallery-item` elements). The six projects: Bungalow Farmhouse (Pune), Garden Restaurant (Hinjewadi), Terrace Garden (Ravet), 5-Star Hotel (Goa), Resort (Pavana Dam), Wellness Resort (Hadshi).

**Legal pages:** `privacy-policy.html` (DPDP Act 2023 / IT Act framing), `terms.html`, and a styled `404.html` (noindex).

---

## Forms / contact / integrations

There is **no form and no form-handling backend** — consistent with the static, no-backend architecture. Contact is entirely via native links: `mailto:elegantlandscape2498@gmail.com` and `tel:+919175862498`, surfaced as contact channels and a "Start a Conversation" CTA on the home page and in every footer. No analytics tags, no third-party trackers, no email/CRM integration found in any HTML or JS. The only external network call is the Google Fonts stylesheet `@import`. The CSP in `vercel.json` is correspondingly tight (`connect-src 'self'`, `form-action 'self'`, `object-src 'none'`), only opening `style-src`/`font-src` to `fonts.googleapis.com`/`fonts.gstatic.com`.

---

## Styling & responsive design

One 2,054-line `styles.css`. A `:root` design-token system defines the palette (warm cream `--bg:#F5ECDD`, olive/sage greens, gold accent `--gold:#C5B748`), the two font stacks (`--font-heading` Cormorant Garamond serif, `--font-body` Inter), radius scale, three shadow levels, and three cubic-bezier transition speeds. Fonts come from a single Google Fonts `@import` at the top.

Responsive behavior uses **6 `@media` blocks**: breakpoints at **1100px** (×2), **900px**, **600px**, and **520px** (×2). The navbar collapses to a hamburger at `max-width: 900px` (matches the README's "below 900px" claim), and the grids (service cards, about body, services) collapse to single columns there. The animation engine itself branches at **768px** (`IS_MOBILE`) for the cheaper render path described above. The site is built mobile-aware: smaller DPR cap, shorter scroll zone, and the GPU-friendly blur fallback on phones.

Note: a project-wide search for `prefers-reduced-motion` found **no matches** — the scroll animation does not currently honor reduced-motion preferences (an accessibility gap).

---

## SEO / meta / performance considerations

Strong, deliberate SEO setup:
- Per-page `<title>`, `description`, canonical URL, `theme-color`, and `robots: index, follow, max-image-preview:large`.
- Full **Open Graph** + **Twitter Card** tags (1200×630 `og-image.jpg`); project pages use `og:type=article`.
- **JSON-LD** `ProfessionalService` structured data on the home page (name, address, phone, email, `areaServed`, `founder`, `foundingDate`).
- `sitemap.xml` (9 URLs: home + 6 projects + privacy + terms) and a hand-tuned `robots.txt` that **allows** search engines + AI *answer-engine* fetchers (ChatGPT-User, OAI-SearchBot, PerplexityBot, etc.) but **blocks** AI *training* crawlers (GPTBot, ClaudeBot, Google-Extended, CCBot, Bytespider…) and SEO-spam bots (Ahrefs/Semrush/MJ12).
- `llms.txt` (llmstxt.org format) — a markdown summary of the studio for LLM answer engines.
- Favicon is an inline SVG data-URI (🌿 emoji) — zero extra request.

Performance: WebP frame set instead of PNG; key-frame-first progressive load; lazy background fill; `loading="lazy"` on project/gallery images; DPR cap; mobile-specific render path; long-lived immutable cache headers for `/ImagesVideos/*` and `/Projects/*` (1 year) vs short revalidating cache for HTML/CSS/JS.

---

## Deployment (Vercel config)

`vercel.json` (schema-validated) drives a pure static deploy:
- `cleanUrls: true`, `trailingSlash: false` — so `/project-1` serves `project-1.html`.
- **Redirects** (all permanent/301): `/index`, `/index.html`, `/home` → `/`; `/privacy` → `/privacy-policy`; `/terms-of-service`, `/tos` → `/terms`; `/projects/1..6` → `/project-1..6`.
- **Security headers** on `/(.*)`: HSTS (2-year, preload), `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Referrer-Policy: strict-origin-when-cross-origin`, a long locked-down `Permissions-Policy`, `X-XSS-Protection: 0`, a strict **CSP**, COOP `same-origin`, CORP `same-origin`, and `X-Robots-Tag`.
- **Cache-Control** tuned per asset class: images immutable 1-year; `styles.css`/`script.js` 1-hour must-revalidate; HTML `max-age=0` must-revalidate; robots/sitemap 1-day (sitemap also gets an explicit XML content-type).

Deploy flow per `README.md`/`DEPLOYMENT.md`: Vercel is wired to the repo, every push to `main` auto-deploys; the runbook covers buying the domain on GoDaddy and pointing DNS at Vercel (with Vercel's free SSL). No CI config beyond Vercel's git integration.

---

## Status, completeness & notable gaps

The site is **complete and live in production**. All five top-level pages, six project pages, legal pages, and the 404 render and are wired together. The animation engine, lightboxes, nav, counters, and tilt effects are all implemented and functional.

Notable gaps / observations:
- **No `prefers-reduced-motion` handling** — the scroll-scrub canvas runs regardless of OS motion settings (accessibility gap).
- **No automated tests, no CI** beyond Vercel git auto-deploy, and **no build pipeline** — WebP generation and PNG→WebP conversion are done out-of-band (the PNG originals and source MP4s are gitignored).
- **Hand-managed cache-busting** (`?v=6`) — easy to forget to bump; the last commit message literally is a cache-bust bump after a "scroll-animation rollback," hinting the animation has been iterated/reverted.
- **Manual frame index** — `TOTAL_FRAMES = 192` is a hard-coded constant that must match the directory; nothing validates the count at runtime (a missing frame just degrades to the nearest loaded neighbor).
- Lightbox JS is duplicated inline across all six project pages rather than shared in a file.

---

## README vs. code

The README is largely accurate. Verified-true claims: vanilla stack with **no framework / no build / no package manager** (no `package.json`, no `node_modules`); **192 WebP frames**; navbar collapses to hamburger **below 900px** (CSS `@media (max-width: 900px)`); CSP/HSTS/X-Frame/Referrer/Permissions/COOP/CORP headers and per-asset cache rules (all present in `vercel.json`); clean URLs; magazine project pages with keyboard-nav lightbox; robots.txt tuned to allow answer-engines and block training crawlers; **sitemap "9 public URLs"** (the sitemap does contain 9 `<loc>` entries — home + 6 projects + 2 legal); JSON-LD `ProfessionalService`; hosted on Vercel, DNS via GoDaddy.

Discrepancies / nuances to flag:
- **Founding year conflict (in code, not README):** the home-page prose and `llms.txt` both say the studio was *"established in 2022,"* but the JSON-LD `foundingDate` is **`2023-06-23`**. These contradict each other; the README itself doesn't state a year, so this is an internal code inconsistency to fix one way or the other.
- **README repo-visibility wording:** the README says *"The repository is public for transparency and portfolio reference"* and the `LICENSE` says *"made public… for portfolio reference."* But the documentation task specifies the repo is **Private**. So the code/README's "public" framing does not match the repo's actual current visibility — likely the repo was flipped to private after these files were written.
- **README "No framework, no build step, no package manager"** is true, but note it omits the one external runtime dependency: the **Google Fonts `@import`** in `styles.css`. The site is not fully self-contained/offline-capable.
- README's repo tree lists files accurately; `og-image.jpg` is tracked while the heavy PNG/MP4 sources are intentionally gitignored — consistent with `.gitignore`/`.vercelignore`.

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\ElegantWebsite. Source of truth: https://github.com/AmeyaBorkar/ElegantLandscape*
