# PRISM

> AI-powered stock-research agent that autonomously decides what to investigate via a ReAct loop over 12 tools, streams its chain-of-thought into a live dashboard, and synthesizes equity-research reports.

**Repository:** [`AmeyaBorkar/PRISM`](https://github.com/AmeyaBorkar/PRISM)  
**Category:** ML Application / Agentic AI  
**Visibility:** Public  
**Primary language:** Python  
**Default branch:** `main`  
**License:** —  
**Created:** 2026-03-31  
**Last pushed:** 2026-03-31  
**Metadata updated:** 2026-03-31  
**Size (GitHub reported):** 92 KB  

---

## What it is (one-paragraph version)

AI-powered stock-research agent that autonomously decides what to investigate via a ReAct loop over 12 tools, streams its chain-of-thought into a live dashboard, and synthesizes equity-research reports.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Python | 179,473 | 89.9% |
| HTML | 20,204 | 10.1% |

## File tree

- Total entries indexed: **86** (64 files, 22 directories)

```
.env.example  (377 B)
.gitignore  (347 B)
README.md  (8 KB)
pyproject.toml  (2 KB)
config/    [4 files]
  config/default.yaml
  config/models.yaml
  config/prompts.yaml
  config/sources.yaml
src/    [53 files]
  src/prism/__init__.py
  src/prism/__main__.py
  src/prism/agent/__init__.py
  src/prism/agent/analyst.py
  src/prism/agent/prompts.py
  src/prism/agent/tools.py
  src/prism/analysis/__init__.py
  src/prism/analysis/historical.py
  src/prism/analysis/pattern_matcher.py
  src/prism/analysis/sentiment.py
  src/prism/analysis/technical.py
  src/prism/cli/__init__.py
  src/prism/cli/app.py
  src/prism/cli/commands/__init__.py
  src/prism/cli/commands/analyze.py
  ... and 38 more under src/
tests/    [3 files]
  tests/__init__.py
  tests/integration/__init__.py
  tests/unit/__init__.py
```

## README (verbatim)

# Prism

AI-powered stock analysis agent that researches stocks like a human analyst.

Give it a company name. It decides what to research, calls tools, reads articles, runs ML models, and writes a comprehensive equity research report — all dynamically, like a real analyst would.

![Prism UI](https://img.shields.io/badge/UI-Web%20Dashboard-blue) ![Python](https://img.shields.io/badge/Python-3.11+-green) ![License](https://img.shields.io/badge/License-MIT-yellow)

## How It Works

Prism uses a **ReAct agent loop** — the LLM decides what to do next based on what it discovers:

```
You: "Analyze Apple"

Agent: get_stock_price(AAPL)        → $246.63, down 10% in 90 days
Agent: get_company_profile(AAPL)    → $3.6T market cap, 26.5 P/E
Agent: search_news("Apple")        → 23 articles found
Agent: read_webpage(article_url)    → Reads full article via Jina Reader
Agent: "Price dropped — is this sector-wide?"
Agent: compare_stocks(AAPL, XLK)    → AAPL lagging tech sector
Agent: compute_technical_indicators → RSI 35.6, below all SMAs
Agent: run_prediction_model(all)    → LSTM: bullish, XGBoost: bullish, Prophet: neutral
Agent: search_web("Apple analyst ratings") → Consensus target $296.90
Agent: get_sec_filings(AAPL)        → Recent 10-Q filed
Agent: submit_report               → BULLISH, medium confidence
```

The agent is **not scripted** — it follows interesting leads, digs deeper when something looks unusual, and decides when it has enough information.

## Features

- **Dynamic AI Agent** — LLM decides the research flow via tool use (not a fixed pipeline)
- **12 Research Tools** — price data, company profiles, news search, article reading, technical analysis, sentiment analysis, ML predictions, stock comparison, web search, SEC filings, historical patterns
- **4 ML Models** — LSTM, XGBoost, Prophet, ARIMA with weighted ensemble voting
- **Real-time Web UI** — ChatGPT-style interface with live tool call streaming, dark mode
- **Web Scraping** — Jina Reader API for clean article extraction from any URL
- **News Aggregation** — Google News RSS + yfinance + optional NewsAPI
- **SEC Filings** — EDGAR API integration (free, no key needed)
- **Multi-provider LLM** — Works with Groq, Cerebras, OpenAI, Anthropic (any OpenAI-compatible API)
- **Free by default** — Groq (free) + yfinance + Google News + Jina Reader + SEC EDGAR

## Quick Start

```bash
# Clone
git clone https://github.com/AmeyaBorkar/PRISM.git
cd PRISM

# Install
pip install -e .

# Set your LLM API key (Groq is free — https://console.groq.com)
echo "GROQ_API_KEY=gsk_your_key_here" > .env

# Run the web UI
python -c "from dotenv import load_dotenv; load_dotenv(); import uvicorn; from prism.web.app import app; uvicorn.run(app, host='0.0.0.0', port=8000)"
```

Open **http://localhost:8000** and enter a company name.

### CLI Usage

```bash
python -m prism analyze "Apple"
python -m prism analyze "TSLA" --format html --output report.html
python -m prism analyze "NVIDIA" --horizon 60
python -m prism version
python -m prism cache clear
```

## Architecture

```
User Input (company name)
        |
        v
  Resolve Ticker (yfinance)
        |
        v
  ┌─────────────────────────┐
  │   Claude/Groq/LLM Agent │
  │   (ReAct tool-use loop) │
  │                         │
  │   Thinks → Acts →       │
  │   Observes → Repeats    │
  └────────┬────────────────┘
           |
   ┌───────┼───────────────────────────┐
   │       │       │        │          │
   v       v       v        v          v
 Price   News    Web     Technical    ML
 Data   Search  Scrape   Analysis   Models
(yfinance)(Google (Jina   (ta lib)  (LSTM,
          News)  Reader)           XGBoost,
                                  Prophet,
                                  ARIMA)
   │       │       │        │          │
   └───────┼───────┴────────┴──────────┘
           |
           v
   Agent synthesizes all data,
   calls submit_report when ready
           |
           v
   ┌─────────────────┐
   │  Analyst Report  │
   │  (MD / HTML / JSON)│
   └─────────────────┘
```

## Project Structure

```
PRISM/
├── config/
│   ├── default.yaml          # LLM provider, model, cache settings
│   ├── models.yaml           # ML hyperparameters & ensemble weights
│   ├── sources.yaml          # Data source registry
│   └── prompts.yaml          # Agent prompt customization
├── src/prism/
│   ├── agent/
│   │   ├── analyst.py        # ReAct agent loop (core engine)
│   │   └── tools.py          # 12 tool definitions + executor
│   ├── analysis/
│   │   ├── technical.py      # RSI, MACD, SMA, Bollinger
│   │   ├── sentiment.py      # VADER sentiment scoring
│   │   ├── historical.py     # Event-impact correlation
│   │   └── pattern_matcher.py # Cosine similarity patterns
│   ├── ml/
│   │   ├── models/           # LSTM, XGBoost, Prophet, ARIMA
│   │   ├── ensemble.py       # Weighted ensemble voting
│   │   └── preprocessing.py  # Feature engineering
│   ├── data/
│   │   ├── sources/          # yfinance, news, SEC EDGAR
│   │   └── cache/            # SQLite cache with per-type TTLs
│   ├── web/
│   │   ├── app.py            # FastAPI + SSE streaming
│   │   └── static/index.html # Chat UI (dark mode)
│   ├── cli/                  # Typer CLI
│   ├── reporting/            # Markdown/HTML/JSON report renderer
│   └── pipeline/             # Orchestrator
├── pyproject.toml
└── .env.example
```

## Configuration

### Switch LLM Provider

Edit `config/default.yaml`:

```yaml
llm:
  provider: "groq"                          # groq, cerebras, openai, anthropic
  model: "llama-3.3-70b-versatile"          # model ID
  base_url: "https://api.groq.com/openai/v1"
```

Set the matching API key in `.env`:

```
GROQ_API_KEY=gsk_...       # Groq (free)
CEREBRAS_API_KEY=csk_...   # Cerebras (free)
ANTHROPIC_API_KEY=sk-...   # Anthropic (paid)
```

### ML Model Weights

Edit `config/models.yaml`:

```yaml
ensemble:
  method: "weighted_average"
  weights:
    lstm: 0.30
    xgboost: 0.30
    prophet: 0.25
    arima: 0.15
```

### Add Premium Data Sources

```
NEWSAPI_KEY=...        # newsapi.org — better news coverage
ALPHA_VANTAGE_KEY=...  # alphavantage.co — financial statements
```

## Research Tools

| Tool | Source | What it does |
|------|--------|-------------|
| `get_stock_price` | yfinance | Price history, returns, 52-week range |
| `get_company_profile` | yfinance | Fundamentals, P/E, margins, analyst targets |
| `search_news` | Google News RSS | Recent articles with titles and summaries |
| `read_webpage` | Jina Reader API | Extract full article text from any URL |
| `search_web` | DuckDuckGo | General web search |
| `get_financial_events` | yfinance | Earnings, dividends, splits, analyst recs |
| `compute_technical_indicators` | `ta` library | RSI, MACD, SMA, Bollinger, ATR |
| `analyze_news_sentiment` | VADER | Sentiment scoring on collected articles |
| `find_historical_patterns` | Cosine similarity | Similar past price patterns |
| `run_prediction_model` | TensorFlow, XGBoost, Prophet, statsmodels | ML ensemble predictions |
| `compare_stocks` | yfinance | Compare performance vs market/sector/peers |
| `get_sec_filings` | SEC EDGAR | Recent 10-K, 10-Q, 8-K filings |

## Tech Stack

**Agent**: Groq/Cerebras/OpenAI-compatible API with function calling
**ML**: TensorFlow/Keras (LSTM), XGBoost, Prophet, statsmodels (ARIMA)
**Data**: yfinance, Google News RSS, Jina Reader, SEC EDGAR, DuckDuckGo
**Web**: FastAPI + SSE + vanilla JS
**Analysis**: pandas, numpy, scikit-learn, ta, vaderSentiment

## Disclaimer

This tool is for **informational and educational purposes only**. It is not financial advice. Always do your own research and consult a qualified financial advisor before making investment decisions. AI predictions are inherently uncertain.

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/PRISM*
