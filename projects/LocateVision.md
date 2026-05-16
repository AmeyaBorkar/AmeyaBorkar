# LocateVision

> Web application that identifies campus locations from photographs using a hybrid ResNet-50 + Swin Transformer ensemble with gated attention — 99.76% classification accuracy.

**Repository:** [`AmeyaBorkar/LocateVision`](https://github.com/AmeyaBorkar/LocateVision)  
**Category:** ML Application / Computer Vision  
**Visibility:** Public  
**Primary language:** HTML  
**Default branch:** `main`  
**License:** MIT License  
**Created:** 2026-02-11  
**Last pushed:** 2026-02-11  
**Metadata updated:** 2026-02-11  
**Size (GitHub reported):** 38 KB  

---

## What it is (one-paragraph version)

Web application that identifies campus locations from photographs using a hybrid ResNet-50 + Swin Transformer ensemble with gated attention — 99.76% classification accuracy.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| HTML | 144,881 | 86.2% |
| Python | 23,215 | 13.8% |

## File tree

- Total entries indexed: **20** (16 files, 4 directories)

```
LICENSE  (1 KB)
README.md  (3 KB)
requirements.txt  (247 B)
frontend/    [10 files]
  frontend/about.html
  frontend/contact.html
  frontend/coverage.html
  frontend/explore.html
  frontend/index.html
  frontend/locations/fruit-canteen.html
  frontend/locations/kiosk.html
  frontend/locations/staff-canteen.html
  frontend/locations/vit-canteen.html
  frontend/places.html
model/    [2 files]
  model/deploy.py
  model/train.py
server/    [1 files]
  server/image_classifier_server.py
```

## README (verbatim)

# LocateVision

**AI-Powered Image-to-Location Finder** — Upload an image and let our hybrid deep learning model identify the location on campus.

## Overview

LocateVision is a web application that uses a **Hybrid CNN-Transformer classifier** (ResNet-50 + Swin Transformer with Gated Attention) to classify campus locations from photographs. Users upload an image through the frontend, the server runs inference, and the predicted location is displayed along with navigation details.

### Key Features
- **Hybrid Architecture**: ResNet-50 (CNN) + Swin Transformer combined with Gated Attention mechanism
- **99.76% Accuracy** on campus location classification
- **Flask REST API** for real-time image classification
- **Responsive Frontend** with user authentication, dark mode, history tracking, and plan management

## Project Structure

```
ASEPojectRoot/
├── server/
│   └── image_classifier_server.py   # Flask API for image prediction
├── model/
│   ├── train.py                     # Training script (ResNet-50 + Swin Transformer)
│   ├── deploy.py                    # Deployment / testing script
│   └── weights/                     # Model weights (.pth) — gitignored
├── frontend/
│   ├── index.html                   # Landing page
│   ├── explore.html                 # Image upload & prediction UI
│   ├── about.html                   # About Us
│   ├── contact.html                 # Contact form
│   ├── coverage.html                # Coverage information
│   ├── places.html                  # Campus places overview
│   └── locations/
│       ├── fruit-canteen.html       # Location detail pages
│       ├── vit-canteen.html
│       ├── staff-canteen.html
│       └── kiosk.html
├── .gitignore
├── requirements.txt
└── README.md
```

## Tech Stack

| Component | Technology |
|-----------|-----------|
| **ML Model** | PyTorch, timm (Swin Transformer), torchvision (ResNet-50) |
| **Training** | Albumentations, AdamW + CosineAnnealingWarmRestarts |
| **Server** | Flask, Flask-CORS |
| **Frontend** | HTML5, CSS3, JavaScript, Bootstrap 5 |

## Setup

### Prerequisites
- Python 3.8+
- CUDA-compatible GPU (recommended for training)

### Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/LocateVision.git
cd LocateVision

# Install dependencies
pip install -r requirements.txt
```

### Running the Server

```bash
cd server
python image_classifier_server.py
```

The server will start on `http://localhost:5000`. Open `frontend/explore.html` in a browser to use the application.

### Training (Optional)

```bash
cd model
# Set data directories via environment variables or edit train.py defaults
export RAW_DATA_DIR="path/to/raw/images"
export PROCESSED_DATA_DIR="path/to/processed/images"
python train.py
```

## Model Architecture

The classifier uses a **hybrid approach** combining:
1. **ResNet-50** — CNN-based spatial feature extraction
2. **Swin Transformer** — Attention-based global feature extraction
3. **Gated Attention Mechanism** — Adaptive feature fusion
4. **Swish Activation** — Smooth, non-dying activation function

Features from both backbones are concatenated and passed through a gated attention layer before final classification.

## License

© 2025 LocateVision | All Rights Reserved

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/LocateVision*
