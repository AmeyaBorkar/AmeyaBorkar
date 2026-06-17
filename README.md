<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0d1117,50:1a1a2e,100:16213e&height=180&section=header&text=Ameya%20Borkar&fontSize=58&fontColor=e6edf3&fontAlignY=38&desc=i%20build%20things.%20some%20of%20them%20even%20work.&descAlignY=56&descSize=18&animation=fadeIn" width="100%"/>

[![Typing SVG](https://readme-typing-svg.demolab.com?font=JetBrains+Mono&weight=500&size=16&duration=3000&pause=800&color=58A6FF&center=true&vCenter=true&width=800&lines=wrote+a+70k-line+hypervisor+microkernel+·+for+fun+·+it+boots.;taught+a+drone+to+see+things.+it+won+nationally.;yes%2C+i+beatbox.+no%2C+you+cannot+hear+it.;barbell+in+the+morning+·+keyboard+at+night.)](https://git.io/typing-svg)

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/ameya-borkar-69b947381)
[![Portfolio](https://img.shields.io/badge/Portfolio-000000?style=flat-square&logo=firefox&logoColor=58a6ff)](https://ameyaborkar-60d8c.web.app)
[![Email](https://img.shields.io/badge/Email-EA4335?style=flat-square&logo=gmail&logoColor=white)](mailto:ameyaborkar17@gmail.com)
[![Profile Views](https://komarev.com/ghpvc/?username=AmeyaBorkar&color=58a6ff&style=flat-square&label=people+who+ended+up+here)](https://github.com/AmeyaBorkar)

</div>

<br>

## hey.

i'm ameya. i write code, lift heavy things, ride motorcycles, and beatbox — not necessarily in that order, never at the same time.

my computer vision stack once helped a drone win a national championship. my barbell still doesn't care.

when i learn something, my first thought is *"okay but what can i build with this?"* — which is how i ended up publishing research, deploying backends for real companies, and accidentally becoming the person who beatboxes at tech events.

> wrote a banking system in C — stock market and all — just to see if i could. i could.
> so i wrote a 70,000-line hypervisor microkernel next. it boots on three different laptops.

🌐 &nbsp;[ameyaborkar-60d8c.web.app](https://ameyaborkar-60d8c.web.app)

<br>

## things i've built

sorted the way the repo is — by what they actually are. each one links to a detailed, **code-derived** writeup under [`projects/`](projects/): i read the source (not the readmes), counted the lines myself, and noted where the two disagree.

<br>

### ⚙️ &nbsp;[systems](projects/systems/) &nbsp;·&nbsp; *low-level things that probably shouldn't work, but do*

| | what | with what | the twist |
|:--:|------|-----------|-----------|
| ⚙️ | **[VeridianOS](projects/systems/VeridianOS.md)** — type-1 hypervisor microkernel | C · assembly · VT-x / AMD-V · IOMMU · EPT/NPT | 70,788 lines of bare-metal C. every OS component runs inside its own hardware-isolated VM domain. ships its own tcp/ip stack. boots end-to-end on three laptops spanning Kaby Lake, Tiger Lake *and* Zen 5. i'm still processing this. |
| 🧱 | **[TaskForge-OS](projects/systems/TaskForge-OS.md)** — an OS in miniature | C · POSIX threads · Win32 | a banking app that talks to a tiny kernel through "system calls." schedulers, paging, deadlock detection — the whole OS-course greatest-hits album. |
| 🔄 | **[SyncUp](projects/systems/SyncUp.md)** — concurrent file backup | C · pthreads · slab allocator | a directory mirror that's secretly an entire operating-systems course. worker pool, custom allocator, resource-allocation-graph deadlock tracking. |
| 🚓 | **[GridWatch](projects/systems/GridWatch.md)** — emergency dispatch sim | C11 · Fibonacci heaps · suffix arrays · BK-trees · TUI + Flask | sends ambulances around a grid city. every number on screen is powered by a different advanced data structure. the whole DS syllabus, cosplaying as a 911 dispatcher. |
| 📚 | **[OS-Exam-Prep](projects/systems/OS-Exam-Prep.md)** — study SPA | stdlib Python → static HTML/JS | compiles markdown notes into a zero-dependency single-page site. 62 model answers, 20 mind maps, not a single framework. |
| 🌲 | **[Smart-Dictionary](projects/systems/Smart-Dictionary.md)** — autocomplete + tree benchmark | C · BST · AVL · threaded BT | benchmarked three tree structures behind a word autocompleter. picked a winner. (it's AVL. it's usually AVL.) |

### 🧠 &nbsp;[ml-research](projects/ml-research/) &nbsp;·&nbsp; *teaching machines to do things, then writing down what didn't work*

| | what | with what | the twist |
|:--:|------|-----------|-----------|
| 📜 | **[PROMETHEUS](projects/ml-research/PROMETHEUS.md)** — binarization study | PyTorch · Neural ODEs · DINOv2 · ASPP | ablated every fancy idea on DIBCO. the physics-informed ODE made it *worse*. wrote that down. the boring patch-extraction trick is what actually won. |
| 🧠 | **[AdaptaNet](projects/ml-research/AdaptaNet.md)** — adaptive vision framework | PyTorch · deformable CNN · MoE transformer | dynamic backbone + mixture-of-experts transformer + a dozen custom losses. more config knobs than sense. |
| 🎭 | **[SGEE](projects/ml-research/SGEE.md)** — emotion embeddings | PyTorch · RoBERTa · NRC-VAD | anchors transformer representations in valence-arousal-dominance space. contrastive grounding stacked on supervised heads. |
| 🎵 | **[EdgeCAAI-Net](projects/ml-research/EdgeCAAI-Net.md)** — music genre classifier | PyTorch · torchaudio · gradient reversal | 1.42M params. accidentally proved the benchmarks were leaking artists between train and test — 559 of 765 of them. oops. |
| 🧬 | **[Engram](projects/ml-research/Engram.md)** — LLM memory that *consolidates* | Python · PyPI (`engrampy`) · clustering · principled decay | raw events → summaries → abstractions. use strengthens, redundancy decays on a closed-form curve. because "vector db with a search bar" is not memory. |
| 🖼️ | **[LocateVision](projects/ml-research/LocateVision.md)** — campus classifier | ResNet-50 · Swin Transformer · Flask | fuses a CNN and a transformer through a learned gate. served behind a small web UI. |
| 📦 | **[Tesseract](projects/ml-research/Tesseract.md)** — cargo X-ray VLM | FastAPI · Qwen2.5-VL-7B · HuggingFace | a vision-language model for customs officers. natural-language contraband reasoning, with a python risk-scorer doing the math the model won't. |

### 💹 &nbsp;[finance](projects/finance/) &nbsp;·&nbsp; *money, but make it numerical*

| | what | with what | the twist |
|:--:|------|-----------|-----------|
| 📊 | **[trader-edge](projects/finance/trader-edge.md)** — pre-trade risk engine | Python · NumPy · SciPy · option-chain IV · Monte Carlo | no LLMs in the loop. black-scholes, IV inversion, barrier first-passage, a 25k-path monte carlo. tells you when your "1:3 R:R" is actually 1:3 noise. |
| 📈 | **[Nifty-Pricing-Mirror](projects/finance/Nifty-Pricing-Mirror.md)** — basis surface | Python · Groww API · rich · Flask | live spot-vs-futures basis for NSE indices, refreshed every 3s. premium, discount, or flat — in a terminal table. |
| 🔮 | **[PRISM](projects/finance/PRISM.md)** — autonomous stock analyst | LLM ReAct loop · Groq · XGBoost · Prophet | an agent that picks its own research tools and runs them live, then an ML ensemble argues with it. watch it think in real time. |
| 💹 | **[LexDrift](projects/finance/LexDrift.md)** — SEC filing radar | FastAPI · Next.js · sentence-transformers | detects when companies *suddenly* start sounding different in their 10-Ks. drift, by cosine distance. finance with extra steps. |

### 🚦 &nbsp;[infra](projects/infra/) &nbsp;·&nbsp; *one product, three repos, a formal proof*

| | what | with what | the twist |
|:--:|------|-----------|-----------|
| 🚦 | **[throttlekit](projects/infra/throttlekit.md)** — rate limiting for node & the web | TypeScript · GCRA · Redis · Postgres · TLA+ | one decision transform, proven bit-identical across in-memory / Redis / Postgres. fixed-memory DDoS sketches. a fleet-overshoot bound checked in *TLA+*. zero deps, on npm. |
| 🐍 | **[throttlekit-py](projects/infra/throttlekit-py.md)** — the python client | Python · gRPC · Redis Lua | same decisions from python — over gRPC to the node core, or running the core's own Lua directly. conformance-proven on shared golden vectors. |
| 🌗 | **[throttlekit-site](projects/infra/throttlekit-site.md)** — the pitch | Astro | "meter what your LLM spends, prove what your fleet admits." docs synced straight out of the monorepo. |

### 🌐 &nbsp;[web](projects/web/) &nbsp;·&nbsp; *things that run in a browser, mostly without a build step*

| | what | with what | the twist |
|:--:|------|-----------|-----------|
| 🗺️ | **[proximap](projects/web/proximap.md)** — headless geospatial engine | TypeScript · OpenStreetMap · MCP | not a map widget — an engine. nearby-ranking, isochrones, walkability, and an errand planner that solves a generalized TSP with exact held-karp DP. ships as a library, a CLI, *and* an MCP server. |
| 📝 | **[markdown-viewer](projects/web/markdown-viewer.md)** — one package, three surfaces | Python · FastAPI · tkinter · pure-stdlib markdown engine | rest api, desktop app, website — all off a hand-written markdown engine. no external markdown lib. four hand-tuned themes. |
| 🔤 | **[FontVisualizer](projects/web/FontVisualizer.md)** — type specimen tool | vanilla JS | 132 google fonts, live preview, an HSV color wheel, and a full-screen "studio board." one html file, no build. |
| 🌾 | **[elegantlandscape.in](projects/web/Elegant-Landscape.md)** — live studio site | vanilla HTML/CSS/JS · canvas · Vercel | a real site for a Pune landscape studio. 192-frame scroll-driven canvas animation. zero frameworks, zero build step. |

### 📱 &nbsp;[apps](projects/apps/) &nbsp;·&nbsp; *the ones real people might actually open*

| | what | with what | the twist |
|:--:|------|-----------|-----------|
| 🇮🇳 | **[SARVA](projects/apps/SARVA.md)** — pan-india life utility | Go · Flutter · PostgreSQL + RLS | sab kuch. sabke liye. four pillars — samay · parivar · adhikar · awaaz. foundation's laid: RS256 auth, argon2id OTPs, row-level security on every table. the rest is going up like real users land tomorrow — because they will. |
| 🏦 | **[BankSystem](projects/apps/BankSystem.md)** — banking in a terminal | C · Win32 console · multithreading | a whole bank with its own stock market and a background price ticker. in C. for fun. yes, really. |
| 💬 | **[JavaChat](projects/apps/JavaChat.md)** — whatsapp clone | Java · sockets · Swing · MySQL | wrote a custom TCP protocol because apparently that's a thing i do. thread-per-client, member-scoped delivery, offline reconciliation. |
| 📑 | **[ExcelLinker](projects/apps/ExcelLinker.md)** — sheet joiner | Python · pandas · tkinter | point it at two spreadsheets, pick the key columns, get one dataset. a friendly GUI over a pandas join. |
| 🩺 | **[Personal-Health-Assistant](projects/apps/Personal-Health-Assistant.md)** — AI nutrition coach | Flask · React · Cerebras · Llama-4 | a full-stack coach that plans meals and workouts. JWT auth, SQLite, an LLM doing the talking. |

<br>

## what i'm doing rn

- ⚙️ &nbsp;writing a microkernel in my spare time *(it boots. i'm as surprised as you are.)*
- 🚦 &nbsp;shipping a rate limiter with a formal proof stapled to it *(yes, TLA+. yes, for a rate limiter.)*
- 🏋️ &nbsp;5am lifts. yes, by choice. no, i don't know why either.
- 🤖 &nbsp;making machines smarter than me *(which is, admittedly, a little concerning)*
- 🏍️ &nbsp;solving hard problems at 80 km/h
- 🚁 &nbsp;teaching drones to see things
- 🥊 &nbsp;training MMA *(the only thing that hits harder than a segfault.)*
- 📄 &nbsp;published a paper. mentioned it once. here it is again.
- 🥁 &nbsp;beatboxing when no one's watching *(and sometimes when they are)*

<br>

## stack

<div align="center">

[![Languages](https://skillicons.dev/icons?i=python,c,cpp,java,go,ts,js,mysql&theme=dark)](https://skillicons.dev)

[![Web/Backend](https://skillicons.dev/icons?i=nextjs,react,nodejs,express,fastapi,flask,astro,redis,postgres,docker&theme=dark)](https://skillicons.dev)

[![AI/ML](https://skillicons.dev/icons?i=pytorch,tensorflow,opencv&theme=dark)](https://skillicons.dev)

[![Tools](https://skillicons.dev/icons?i=git,linux,github&theme=dark)](https://skillicons.dev)

</div>

<br>

## i also

```
train MMA  ·  play tabla  ·  powerlift  ·  play piano  ·  ride motorcycles  ·  beatbox  ·  swim  ·  play badminton
```

*the beatboxing always catches people off guard. that's the point.*

<br>

## stats

<div align="center">

<img src="https://streak-stats.demolab.com?user=AmeyaBorkar&theme=github-dark-blue&hide_border=true&date_format=M%20j%5B%2C%20Y%5D" />

</div>

<div align="center">

<img src="https://github-readme-activity-graph.vercel.app/graph?username=AmeyaBorkar&theme=github-compact&hide_border=true&area=true" width="100%"/>

</div>

<br>

<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:16213e,50:1a1a2e,100:0d1117&height=100&section=footer&animation=fadeIn" width="100%"/>

</div>
