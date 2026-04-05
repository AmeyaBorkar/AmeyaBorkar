# Elegant Landscape

> Production website for a Pune landscape-architecture studio. Vanilla HTML/CSS/JS, no build step. 192-frame scroll-driven canvas hero, six magazine-style case-study pages. Live at elegantlandscape.in.

**Repository:** [`AmeyaBorkar/ElegantLandscape`](https://github.com/AmeyaBorkar/ElegantLandscape)  
**Live / Homepage:** <https://elegant-landscape.vercel.app>  
**Category:** Web / Production  
**Visibility:** Private  
**Primary language:** HTML  
**Default branch:** `main`  
**License:** Other  
**Created:** 2026-03-05  
**Last pushed:** 2026-04-19  
**Metadata updated:** 2026-04-19  
**Size (GitHub reported):** 176,035 KB  

---

## What it is (one-paragraph version)

Production website for a Pune landscape-architecture studio. Vanilla HTML/CSS/JS, no build step. 192-frame scroll-driven canvas hero, six magazine-style case-study pages. Live at elegantlandscape.in.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| HTML | 108,762 | 64.2% |
| CSS | 39,252 | 23.2% |
| JavaScript | 21,460 | 12.7% |

## File tree

- Total entries indexed: **268** (258 files, 10 directories)

```
.gitignore  (408 B)
.vercelignore  (208 B)
404.html  (3 KB)
DEPLOYMENT.md  (15 KB)
LICENSE  (2 KB)
README.md  (3 KB)
index.html  (15 KB)
llms.txt  (4 KB)
nav.js  (781 B)
og-image.jpg  (151 KB)
privacy-policy.html  (12 KB)
project-1.html  (11 KB)
project-2.html  (9 KB)
project-3.html  (10 KB)
project-4.html  (11 KB)
project-5.html  (10 KB)
project-6.html  (12 KB)
robots.txt  (2 KB)
script.js  (20 KB)
sitemap.xml  (2 KB)
styles.css  (38 KB)
terms.html  (13 KB)
vercel.json  (4 KB)
ImagesVideos/    [197 files]
  ImagesVideos/Projects/courtyard.png
  ImagesVideos/Projects/garden.png
  ImagesVideos/Projects/pool.png
  ImagesVideos/Projects/rooftop.png
  ImagesVideos/Projects/zen.png
  ImagesVideos/WebPFrames/frame_0001.webp
  ImagesVideos/WebPFrames/frame_0002.webp
  ImagesVideos/WebPFrames/frame_0003.webp
  ImagesVideos/WebPFrames/frame_0004.webp
  ImagesVideos/WebPFrames/frame_0005.webp
  ImagesVideos/WebPFrames/frame_0006.webp
  ImagesVideos/WebPFrames/frame_0007.webp
  ImagesVideos/WebPFrames/frame_0008.webp
  ImagesVideos/WebPFrames/frame_0009.webp
  ImagesVideos/WebPFrames/frame_0010.webp
  ... and 182 more under ImagesVideos/
Projects/    [38 files]
  Projects/Project1/Gazebo.png
  Projects/Project1/Info.txt
  Projects/Project1/Inviting backyard pool with lounge chair.png
  Projects/Project1/Lush villa garden with swimming pool.png
  Projects/Project1/Master Plan.jpg
  Projects/Project1/Sunset glow at SHREE 55 entrance.png
  Projects/Project1/Tropical backyard with pool and lounge.png
  Projects/Project1/Tulsi vrindavan.png
  Projects/Project1/Twilight garden with Parijat tree.png
  Projects/Project1/Twilight poolside retreat with tropical flair.png
  Projects/Project1/Vibrant entryway with blooming crepe myrtle.png
  Projects/Project2/Evening garden retreat with lanterns.png
  Projects/Project2/Hotel garden layout and design.png
  Projects/Project2/Info.txt
  Projects/Project2/Minimalist garden with wooden decks.png
  ... and 23 more under Projects/
```

## README (verbatim)

# Elegant Landscape

Website for **Elegant Landscape** — a Pune-based landscape architecture practice. Live at <https://elegantlandscape.in>.

Single-page portfolio with a scroll-driven canvas animation as the hero, dedicated case-study pages for each project, and a magazine-style content layout.

## Stack

Vanilla HTML, CSS, and JavaScript. No framework, no build step, no package manager. Hosted on Vercel; DNS via GoDaddy.

## What's in the repo

```
index.html                    Home: scroll animation + About / Services / Projects / Contact
project-1.html ... project-6.html
                              Individual case-study pages with hero + gallery + lightbox
privacy-policy.html           DPDP Act 2023 / IT Act compliant privacy policy
terms.html                    Terms of service
404.html                      Custom not-found page

styles.css                    Design system, sections, responsive breakpoints
script.js                     Home-page scroll animation engine + nav active state
nav.js                        Shared mobile hamburger toggle (all pages)

ImagesVideos/WebPFrames/      192 WebP frames for the scroll animation
Projects/Project{1..6}/       Renders and drawings per project

vercel.json                   Clean URLs, security headers (CSP, HSTS, X-Frame, etc.), cache rules
robots.txt                    Allows search + retrieval AI crawlers; blocks training crawlers
sitemap.xml                   9 public URLs
llms.txt                      Markdown summary for LLM answer engines (llmstxt.org)
og-image.jpg                  1200x630 social preview card

DEPLOYMENT.md                 Full deployment runbook: GoDaddy → Vercel → DNS → handoff
```

## Key features

- **Scroll-driven animation engine** (`script.js`): 192-frame WebP sequence rendered on canvas, synced to scroll position with key-frame preloading.
- **Staged overlay system**: intro line, service cards, info text, brand reveal — timed against scroll progress.
- **Responsive navbar**: glassmorphed floating pill on desktop, collapses to hamburger below 900px, with scroll-based active-tab highlighting.
- **Magazine project pages**: side-by-side text + image rows, paired and full-width image rows, a lightbox with keyboard navigation.
- **Production hardening**: CSP, HSTS, X-Frame-Options, Referrer-Policy, Permissions-Policy, COOP/CORP headers, clean URLs, cache-control per asset class.
- **SEO + social**: OG / Twitter meta, canonical URLs, JSON-LD `ProfessionalService` schema, sitemap, robots.txt tuned for AI answer engines (allowed) vs training crawlers (blocked).

## Local development

```bash
npx serve .
# open http://localhost:3000
```

No install, no build. Edit any file and reload.

## Deploying changes

Vercel is wired to this repo. Every push to `main` triggers an auto-deploy.

```bash
git add .
git commit -m "what changed"
git push origin main
```

Full deployment, domain, and handoff runbook: [DEPLOYMENT.md](DEPLOYMENT.md).

## License

This is **not** an open-source project. See [LICENSE](LICENSE) — all rights reserved to Elegant Landscape. The repository is public for transparency and portfolio reference; code, imagery, copy, and brand elements are proprietary and may not be copied, redistributed, or used to train AI models without written permission.

Licensing inquiries: [elegantlandscape2498@gmail.com](mailto:elegantlandscape2498@gmail.com)

---

Website development: [Ameya Borkar](https://github.com/AmeyaBorkar)
Studio: Elegant Landscape (Rohan Vivek Dalvi), Pune, Maharashtra, India

---

*Generated 2026-05-02 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/ElegantLandscape*
