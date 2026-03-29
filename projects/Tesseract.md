# Tesseract

> FastAPI service that uses Qwen2.5-VL-7B-Instruct to inspect X-ray cargo scans for contraband, producing risk scores, object classifications, and natural-language explanations.

**Repository:** [`AmeyaBorkar/Tesseract`](https://github.com/AmeyaBorkar/Tesseract)  
**Category:** ML Application / Vision-Language Models  
**Visibility:** Public  
**Primary language:** Python  
**Default branch:** `main`  
**License:** —  
**Created:** 2026-03-23  
**Last pushed:** 2026-03-28  
**Metadata updated:** 2026-03-28  
**Size (GitHub reported):** 163 KB  

---

## What it is (one-paragraph version)

FastAPI service that uses Qwen2.5-VL-7B-Instruct to inspect X-ray cargo scans for contraband, producing risk scores, object classifications, and natural-language explanations.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Python | 40,638 | 35.1% |
| CSS | 31,869 | 27.5% |
| HTML | 24,137 | 20.8% |
| JavaScript | 19,293 | 16.6% |

## File tree

- Total entries indexed: **34** (27 files, 7 directories)

```
.gitignore  (164 B)
Problem Statement - CIIBS.pdf  (166 KB)
README.md  (7 KB)
backend/    [21 files]
  backend/.env.example
  backend/.gitignore
  backend/README.md
  backend/app/__init__.py
  backend/app/config.py
  backend/app/dependencies.py
  backend/app/main.py
  backend/app/middleware.py
  backend/app/models/__init__.py
  backend/app/models/schemas.py
  backend/app/routes/__init__.py
  backend/app/routes/analysis.py
  backend/app/routes/comparison.py
  backend/app/routes/health.py
  backend/app/services/__init__.py
  ... and 6 more under backend/
frontend/    [3 files]
  frontend/app.js
  frontend/index.html
  frontend/styles.css
```

## README (verbatim)

# Tesseract

**AI-Powered Cargo Image Analysis for Customs & Border Security**

Tesseract uses Vision Language Models (VLMs) to analyze cargo scan and X-ray images, detecting suspicious, mis-declared, or prohibited objects to assist customs officers in making faster, more accurate decisions.

---

## Features

| Capability | Description |
|---|---|
| **Image Analysis** | Detect suspicious, concealed, or prohibited items in cargo scans |
| **Image Comparison** | Compare reference vs current scans to spot tampering or mis-declaration |
| **Risk Scoring** | Confidence-based risk scores (0.0-1.0) with LOW / MEDIUM / HIGH / CRITICAL levels |
| **Object Classification** | Categorize detected items as normal, suspicious, restricted, or prohibited |
| **Explainability** | Natural language explanations of why an image was flagged |
| **Recommendations** | Actionable next steps for customs officers |
| **Interactive Dashboard** | Dark-themed security dashboard with drag-and-drop uploads, animated risk gauges, and real-time status |

## Tech Stack

- **Frontend:** Vanilla HTML / CSS / JavaScript (zero build dependencies)
- **Backend:** FastAPI (Python 3.10+)
- **AI Model:** Qwen2.5-VL-7B-Instruct (via HuggingFace Inference API)
- **Image Processing:** Pillow (preprocessing, contrast enhancement, X-ray optimization)
- **Data Validation:** Pydantic

## Project Structure

```
Tesseract/
├── frontend/
│   ├── index.html               # Dashboard UI
│   ├── styles.css               # Dark security-ops theme
│   └── app.js                   # Client-side logic
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI entry point + frontend serving
│   │   ├── config.py            # Settings & environment config
│   │   ├── middleware.py        # Security headers, rate limiting, request tracing
│   │   ├── dependencies.py      # API key authentication
│   │   ├── routes/
│   │   │   ├── health.py        # GET  /api/v1/health
│   │   │   ├── analysis.py      # POST /api/v1/analyze
│   │   │   └── comparison.py    # POST /api/v1/compare
│   │   ├── services/
│   │   │   ├── vlm_service.py   # HuggingFace VLM integration
│   │   │   ├── image_processor.py # Image preprocessing pipeline
│   │   │   └── risk_scorer.py   # Structured risk scoring engine
│   │   ├── models/
│   │   │   └── schemas.py       # Pydantic request/response models
│   │   └── utils/
│   │       └── image_utils.py   # Image validation & conversion helpers
│   ├── .env.example             # Environment variable template
│   ├── requirements.txt         # Python dependencies
│   └── README.md
└── README.md                    # This file
```

## Quick Start

### Prerequisites

- Python 3.10+
- A free [HuggingFace API token](https://huggingface.co/settings/tokens)

### Setup

```bash
# Clone the repository
git clone https://github.com/AmeyaBorkar/Tesseract.git
cd Tesseract/backend

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env and add your HuggingFace API token
```

### Run

```bash
uvicorn app.main:app --reload --port 8000
```

Open **http://localhost:8000** to use the dashboard UI.

Set `DEBUG=True` in `.env` to enable the Swagger docs at `/docs`.

## Dashboard UI

The frontend is a dark-themed security dashboard served directly by FastAPI at the root URL.

**Views:**
- **Dashboard** - System status, model info, quick-action cards
- **Analyze Scan** - Drag-and-drop single image upload with animated risk gauge, detected objects, and recommendations
- **Compare Scans** - Side-by-side two-image upload with difference detection

**Features:**
- Drag-and-drop and click-to-browse file uploads
- Client-side file validation (type and size)
- Animated circular risk gauge with color-coded thresholds
- Detected object cards with category badges and confidence bars
- Toast notifications for success and error states
- Responsive layout with mobile sidebar
- Optional API key input in the topbar

## API Endpoints

### `GET /api/v1/health`
Returns server status and model configuration.

### `POST /api/v1/analyze`
Upload a single cargo/X-ray image for analysis.

**Request:** `multipart/form-data` with `file` field

**Response:**
```json
{
  "risk_score": 0.75,
  "risk_level": "HIGH",
  "summary": "Suspicious dense region detected...",
  "detected_objects": [
    {
      "name": "Concealed metallic object",
      "category": "suspicious",
      "confidence": 0.85,
      "description": "Dense metallic shape inconsistent with declared cargo"
    }
  ],
  "explanation": "Detailed analysis...",
  "recommendations": "IMMEDIATE REVIEW RECOMMENDED..."
}
```

### `POST /api/v1/compare`
Upload two cargo images for comparison analysis.

**Request:** `multipart/form-data` with `file1` and `file2` fields

**Response:**
```json
{
  "risk_score": 0.6,
  "risk_level": "MEDIUM",
  "differences_found": true,
  "summary": "Significant differences detected...",
  "differences": ["Item present in scan 1 missing from scan 2"],
  "explanation": "Detailed comparison..."
}
```

## Production Guardrails

| Guardrail | Description |
|---|---|
| **API Key Auth** | Optional `X-API-Key` header authentication (set `API_KEY` env var) |
| **Rate Limiting** | Sliding-window per-IP rate limiter (configurable requests/window) |
| **Security Headers** | HSTS, X-Content-Type-Options, X-Frame-Options, Referrer-Policy |
| **Request Tracing** | Unique `X-Request-ID` on every request for log correlation |
| **File Validation** | Extension check, magic-byte content verification, size limits |
| **Decompression Bomb Protection** | PIL pixel limit to prevent memory exhaustion |
| **CORS Control** | Configurable allowed origins (defaults to `*` for dev) |
| **Non-blocking VLM** | `asyncio.to_thread` prevents blocking the event loop |
| **VLM Timeout** | Configurable timeout for HuggingFace API calls |
| **Global Exception Handler** | Catches unhandled errors without leaking internals |
| **Structured Logging** | Timestamped, leveled logs replacing print statements |

## Environment Variables

| Variable | Description | Default |
|---|---|---|
| `HF_API_TOKEN` | HuggingFace API token (**required**) | - |
| `MODEL_ID` | VLM model identifier | `Qwen/Qwen2.5-VL-7B-Instruct` |
| `MAX_IMAGE_SIZE` | Max image dimension in pixels | `1920` |
| `MAX_FILE_SIZE` | Max upload size in bytes | `20971520` (20 MB) |
| `API_KEY` | API key for endpoint auth (empty = no auth) | - |
| `ALLOWED_ORIGINS` | Comma-separated CORS origins | `*` |
| `RATE_LIMIT_REQUESTS` | Max requests per rate-limit window | `30` |
| `RATE_LIMIT_WINDOW` | Rate-limit window in seconds | `60` |
| `VLM_TIMEOUT` | VLM API timeout in seconds | `120` |
| `DEBUG` | Enable `/docs` and `/redoc` | `False` |
| `LOG_LEVEL` | Logging level (DEBUG, INFO, WARNING, ERROR) | `INFO` |

## How It Works

1. **Image Upload** - User uploads a cargo scan via the dashboard or API
2. **Validation** - File extension, magic bytes, and size are verified
3. **Preprocessing** - Image is resized, contrast-enhanced, and sharpened (optimized for X-ray scans)
4. **VLM Analysis** - Processed image is sent to Qwen2.5-VL with a customs-analyst prompt
5. **Risk Parsing** - VLM text output is parsed into structured risk scores, object detections, and categories
6. **Response** - Structured JSON response with risk level, explanations, and recommendations

## License

MIT

---

*Generated 2026-05-02 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/Tesseract*
