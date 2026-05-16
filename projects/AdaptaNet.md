# AdaptaNet

> Modular visual-recognition framework combining a Dynamic CNN backbone (deformable conv, selective kernels, coordinate attention) with an Adaptive Transformer (head pruning, MoE FFN) and a 20+ loss library.

**Repository:** [`AmeyaBorkar/AdaptaNet`](https://github.com/AmeyaBorkar/AdaptaNet)  
**Category:** ML Framework / Computer Vision  
**Visibility:** Public  
**Primary language:** Python  
**Default branch:** `main`  
**License:** —  
**Created:** 2026-03-03  
**Last pushed:** 2026-03-05  
**Metadata updated:** 2026-05-10  
**Size (GitHub reported):** 141 KB  

---

## What it is (one-paragraph version)

Modular visual-recognition framework combining a Dynamic CNN backbone (deformable conv, selective kernels, coordinate attention) with an Adaptive Transformer (head pruning, MoE FFN) and a 20+ loss library.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Python | 528,987 | 100.0% |

## File tree

- Total entries indexed: **42** (37 files, 5 directories)

```
.gitignore  (679 B)
README.md  (7 KB)
ROADMAP.md  (7 KB)
__init__.py  (295 B)
model.py  (7 KB)
requirements.txt  (118 B)
dynamic_cnn/    [6 files]
  dynamic_cnn/__init__.py
  dynamic_cnn/attention.py
  dynamic_cnn/backbone.py
  dynamic_cnn/blocks.py
  dynamic_cnn/layers.py
  dynamic_cnn/norm.py
losses/    [7 files]
  losses/__init__.py
  losses/classification.py
  losses/consistency.py
  losses/integration.py
  losses/representation.py
  losses/utils.py
  losses/weighting.py
scripts/    [6 files]
  scripts/ablate.py
  scripts/benchmark.py
  scripts/download_datasets.py
  scripts/preprocess_datasets.py
  scripts/train.py
  scripts/visualize.py
tests/    [2 files]
  tests/test_fixes.py
  tests/test_model.py
transformer/    [10 files]
  transformer/__init__.py
  transformer/attention.py
  transformer/config.py
  transformer/encoder.py
  transformer/hetero_attention.py
  transformer/integration.py
  transformer/layers.py
  transformer/moe.py
  transformer/norm.py
  transformer/pyramid.py
```

## README (verbatim)

# AdaptaNet

> Adaptive multi-scale deep learning architecture for visual recognition.

---

## Overview

AdaptaNet is a modular deep learning framework that combines a **Dynamic CNN backbone** with an **Adaptive Transformer** and a comprehensive **loss system**. The architecture is designed to be general-purpose and can be applied to classification, retrieval, metric learning, and other visual recognition tasks.

### Key Features

- **Dynamic CNN** with deformable convolutions, selective kernels, and coordinate attention
- **Adaptive Transformer** with head pruning, mixture-of-experts FFN, and multi-scale pyramid processing
- **20+ purpose-built loss functions** spanning classification, representation learning, consistency, and dynamic weighting
- **Unified model wrapper** (`AdaptaNet`) with gated CNN→Transformer projection and attentive pooling
- Fully modular — each component is independently importable and configurable

---

## Architecture

```
Input Image [B, 3, H, W]
    │
    ▼
┌──────────────────────────────┐
│   Dynamic CNN Backbone       │  Multi-scale, deformable convolutions,
│   (dynamic_cnn/)             │  coordinate attention, log-polar branch
└──────────┬───────────────────┘
           │ multi-scale features
           ▼
┌──────────────────────────────┐
│  CrossStageFeatureHierarchy  │  Fuses stem + stage_1..4 features
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  AdaptiveChannelProjection   │  Content-aware gated 1x1 conv
│  (model.py)                  │  CNN channels → transformer dim
└──────────┬───────────────────┘
           │ [B, D, H', W'] → flatten → [B, L, D]
           ▼
┌──────────────────────────────┐
│  Adaptive Transformer        │  Adaptive heads, MoE FFN, context-
│  (transformer/)              │  conditioned norm, hetero attention
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  AttentivePool → LayerNorm   │  Learned attention-weighted pooling
│  → Linear Head               │  → embeddings [B, D] + logits [B, C]
└──────────────────────────────┘
```

---

## Project Structure

```
├── model.py                     # AdaptaNet unified model wrapper
│
├── dynamic_cnn/                 # Dynamic CNN backbone
│   ├── norm.py                  # AdaptiveNorm, ScaleAdaptiveNorm
│   ├── attention.py             # Coordinate attention, SE blocks, cross-scale fusion
│   ├── layers.py                # Deformable conv, selective kernel, log-polar conv
│   ├── blocks.py                # Dynamic residual blocks, expert mixture
│   └── backbone.py              # DynamicCNNBackbone, MultiScaleDynamicCNN
│
├── transformer/                 # Adaptive Transformer
│   ├── config.py                # AdaptiveTransformerConfig
│   ├── attention.py             # Efficient self-attention, axial attention
│   ├── layers.py                # Layer norm, SwiGLU, encoder layers
│   ├── encoder.py               # Adaptive encoder with head pruning
│   ├── moe.py                   # Mixture-of-Experts FFN
│   ├── norm.py                  # Context-conditioned layer norm
│   ├── hetero_attention.py      # Heterogeneous attention (standard + linear + adaptive)
│   ├── pyramid.py               # Multi-scale pyramid transformer
│   └── integration.py           # Unified transformer combining encoder + pyramid
│
├── losses/                      # Comprehensive loss system
│   ├── classification.py        # ArcFace, CosFace, AdaCos, Circle
│   ├── representation.py        # Supervised contrastive, triplet, Barlow Twins, VICReg
│   ├── consistency.py           # Viewpoint, illumination, temporal, structural
│   ├── weighting.py             # Uncertainty, gradient norm, curriculum, prioritization
│   ├── integration.py           # LossIntegrationSystem
│   └── utils.py                 # Margin annealing, hard negative mining, temp scheduling
│
├── scripts/                     # Dataset download & preprocessing
│   ├── download_datasets.py     # Download CIFAR-100, CUB-200, SOP
│   └── preprocess_datasets.py   # Resize, normalize, save as .pt tensors
│
├── data/                        # Dataset storage (git-ignored)
│   ├── raw/                     # Downloaded originals
│   │   ├── cifar100/
│   │   ├── cub200/
│   │   └── sop/
│   └── processed/               # Cached tensors for training
│       ├── cifar100/
│       ├── cub200/
│       └── sop/
│
├── tests/                       # Unit tests
│   ├── test_model.py            # AdaptaNet model tests
│   └── test_fixes.py            # Component-level regression tests
│
├── requirements.txt
├── ROADMAP.md
└── .gitignore
```

---

## Quick Start

```python
import torch
from model import AdaptaNet

# Create model
model = AdaptaNet(
    in_channels=3,
    num_classes=200,
    embed_dim=256,
    base_width=64,
    transformer_depth=6,
    num_heads=8,
)

# Forward pass
x = torch.randn(2, 3, 224, 224)
out = model(x)

print(out["logits"].shape)       # [2, 200]
print(out["embeddings"].shape)   # [2, 256]
```

---

## Data Setup

AdaptaNet includes scripts to download and preprocess three benchmark datasets.

### 1. Download raw data

```bash
# Download all datasets
python scripts/download_datasets.py

# Or select specific ones
python scripts/download_datasets.py --dataset cifar100 cub200
```

### 2. Preprocess into tensors

```bash
# Preprocess all (resize to 224x224, normalize with ImageNet stats)
python scripts/preprocess_datasets.py

# Or select specific ones
python scripts/preprocess_datasets.py --dataset cifar100
```

Processed files are saved as `.pt` tensors in `data/processed/` with format `{"images": [N,3,224,224], "labels": [N]}`.

---

## Requirements

- Python >= 3.9
- PyTorch >= 2.0
- NumPy >= 1.24

```bash
pip install -r requirements.txt
```

---

## License

This project is for research and educational purposes.

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/AdaptaNet*
