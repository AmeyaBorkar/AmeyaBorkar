# Projects

Detailed, **code-derived** documentation for 29 of Ameya Borkar's projects, organized by category.

Unlike a typical project list, every doc here was written by **reading the actual source code** — not the README. Each one includes:

- **Computed statistics** — real line counts and source-file counts measured from the tree (not the GitHub-reported numbers), verified parameter counts for ML models where possible.
- **A code walkthrough** — architecture, key modules, and the actual algorithms as implemented.
- **A "README vs. code" section** — concrete discrepancies between what each project claims and what the committed code actually does (stubbed features, dead code, mislabels, build breakage, over/under-counts).

The aim is accuracy over marketing: where a feature is aspirational, scaffolded, or wired-but-unused, the docs say so.

---

## Categories

### ⚙️ [systems/](systems/) — OS, kernels, concurrency, data structures
[VeridianOS](systems/VeridianOS.md) · [TaskForge-OS](systems/TaskForge-OS.md) · [SyncUp](systems/SyncUp.md) · [GridWatch](systems/GridWatch.md) · [OS-Exam-Prep](systems/OS-Exam-Prep.md) · [Smart-Dictionary](systems/Smart-Dictionary.md)

The flagship is **VeridianOS** — a ~70,800-line bare-metal x86-64 type-1 hypervisor microkernel where every OS component runs as a hardware-isolated VM-guest domain.

### 🧠 [ml-research/](ml-research/) — vision, audio, NLP, VLMs, LLM memory
[PROMETHEUS](ml-research/PROMETHEUS.md) · [AdaptaNet](ml-research/AdaptaNet.md) · [SGEE](ml-research/SGEE.md) · [EdgeCAAI-Net](ml-research/EdgeCAAI-Net.md) · [Engram](ml-research/Engram.md) · [LocateVision](ml-research/LocateVision.md) · [Tesseract](ml-research/Tesseract.md)

Research-grade PyTorch work. Highlights: **PROMETHEUS** confirms in code that a physics-informed Neural ODE *hurts* binarization; **EdgeCAAI-Net** measures real artist-leakage (559/765 test artists leak under track-disjoint splits); **Engram** ships on PyPI as `engrampy`.

### 💹 [finance/](finance/) — pricing, basis surfaces, research agents, filing analysis
[trader-edge](finance/trader-edge.md) · [Nifty-Pricing-Mirror](finance/Nifty-Pricing-Mirror.md) · [PRISM](finance/PRISM.md) · [LexDrift](finance/LexDrift.md)

**trader-edge** is a no-LLM pre-trade risk engine (every number is closed-form or numerical); **PRISM** is an autonomous ReAct stock-analyst agent (Groq-backed) with an ML ensemble; **LexDrift** detects linguistic drift in SEC 10-Ks.

### 🚦 [infra/](infra/) — the ThrottleKit family
[throttlekit](infra/throttlekit.md) (TS core) · [throttlekit-py](infra/throttlekit-py.md) (Python client) · [throttlekit-site](infra/throttlekit-site.md) (Astro site)

One rate-limiting product across three repos (plus a 4th component, `throttlekit-server`, inside the core repo). The unifying claim — **bit-identical decisions across in-memory / Redis / Postgres / gRPC / Python** — is backed by shared golden vectors and **5 TLA+ specs with JS twins in CI**. See the [family overview](infra/README.md).

### 🌐 [web/](web/) — front-end & browser tooling
[proximap](web/proximap.md) · [markdown-viewer](web/markdown-viewer.md) · [FontVisualizer](web/FontVisualizer.md) · [Elegant-Landscape](web/Elegant-Landscape.md)

**proximap** is a headless OpenStreetMap geospatial engine (3 npm packages + an MCP server, with a Held-Karp errand planner); **Elegant-Landscape** is a live, zero-build-step studio site with a 192-frame scroll-scrubbed canvas.

### 📱 [apps/](apps/) — end-user applications & platforms
[SARVA](apps/SARVA.md) · [BankSystem](apps/BankSystem.md) · [JavaChat](apps/JavaChat.md) · [ExcelLinker](apps/ExcelLinker.md) · [Personal-Health-Assistant](apps/Personal-Health-Assistant.md)

**SARVA** is a pan-India services platform in foundation phase (strong auth + Postgres RLS, pillars still in `docs/`); the rest range from a C banking system to an AI health coach.

---

*Docs generated 2026-06-16 by reading each project's source tree. Each file footer names the exact local path read and the canonical GitHub repo (source of truth).*
