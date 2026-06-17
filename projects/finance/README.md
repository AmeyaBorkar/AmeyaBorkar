# finance

Markets, trading, and financial-NLP projects — option pricing, basis surfaces, autonomous research agents, and SEC-filing analysis.

Each linked doc below was written by **reading the actual source**, not the README — with computed line counts and an explicit "README vs. code" section.

| Project | What it actually is | Stack / size | Notable code-level finding |
|---------|---------------------|--------------|----------------------------|
| [trader-edge](trader-edge.md) | A **pre-trade risk engine** with no LLMs — every number is closed-form or a numerical op on observable prices | Python · NumPy/SciPy · 2,929 LOC | Verified math: Black-Scholes + Newton→bisection IV inversion, reflection-principle barrier first-passage, a 25k-path Monte Carlo, Breeden-Litzenberger terminal PDF, EV net of costs. Genuinely no LLM. **38** tests (README says 31); `CubicSpline` imported but unused; engine is long-only. |
| [Nifty-Pricing-Mirror](Nifty-Pricing-Mirror.md) | A **live spot-vs-futures basis surface** for NSE indices, refreshed every ~3s (Rich TUI + web dashboard) | Python · Groww API · 1,425 LOC | `basis = future − spot`, annualised by `365/dte`; CONTANGO/BACKWARDATION when one bucket exceeds the other by 1.5×. Mislabel bug: the table title and docstring say "Nifty 50" but the default universe is **Nifty 200** (which also makes 8 API calls/refresh, not the claimed 2). |
| [PRISM](PRISM.md) | An **autonomous stock-analyst agent** — a ReAct loop that picks its own research tools and runs them live, plus an ML ensemble | Python · `openai` SDK · TF/XGBoost/Prophet · 5,120 LOC | Runs on **Groq** (`llama-3.3-70b`) via the OpenAI-compatible SDK; the declared `anthropic` dependency is **never imported**. 13 real tools (yfinance, EDGAR, news, VADER). LSTM/XGBoost/Prophet/SARIMAX ensemble **retrains on every call** (saved weights never loaded). A large "static pipeline" is dead code; no tests. |
| [LexDrift](LexDrift.md) | An **SEC 10-K "linguistic drift" radar** — detects when a company suddenly starts sounding different | FastAPI + Next.js 16 · ~19,107 src LOC | EDGAR iXBRL → 3-pass regex parse → **`bge-small-en-v1.5`** embeddings (README wrongly says `all-MiniLM-L6-v2`) → per-section cosine/Jaccard + sentiment deltas judged by company-specific z-scores. Several runtime deps (transformers/sklearn/torch/groq…) are imported but missing from `pyproject.toml`. |
