# SGEE

> Multi-stage framework that anchors transformer representations of text inside VAD (Valence-Arousal-Dominance) psycholinguistic space via contrastive refinement after supervised pretraining.

**Repository:** [`AmeyaBorkar/SGEE-Semantically-Grounded-Emotion-Embeddings`](https://github.com/AmeyaBorkar/SGEE-Semantically-Grounded-Emotion-Embeddings)  
**Category:** ML Research / NLP  
**Visibility:** Public  
**Primary language:** Python  
**Default branch:** `main`  
**License:** —  
**Created:** 2026-02-27  
**Last pushed:** 2026-02-27  
**Metadata updated:** 2026-02-27  
**Size (GitHub reported):** 1,697 KB  

---

## What it is (one-paragraph version)

Multi-stage framework that anchors transformer representations of text inside VAD (Valence-Arousal-Dominance) psycholinguistic space via contrastive refinement after supervised pretraining.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Python | 392,612 | 100.0% |

## File tree

- Total entries indexed: **60** (48 files, 12 directories)

```
.gitignore  (2 KB)
README.md  (8 KB)
requirements.txt  (68 B)
testRuns.py  (6 KB)
configs/    [1 files]
  configs/base_config.yaml
data/    [19 files]
  data/lexicons/NRC-VAD-Lexicon-v2.1/MWE/mwe-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/MWE/mwe-arousal-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/MWE/mwe-dominance-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/MWE/mwe-valence-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/OneFilePerDimension/PolarSubset/arousal-polar-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/OneFilePerDimension/PolarSubset/dominance-polar-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/OneFilePerDimension/PolarSubset/valence-polar-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/OneFilePerDimension/arousal-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/OneFilePerDimension/dominance-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/OneFilePerDimension/valence-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/README.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/Unigrams/unigrams-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/Unigrams/unigrams-arousal-NRC-VAD-Lexicon-v2.1.txt
  data/lexicons/NRC-VAD-Lexicon-v2.1/Unigrams/unigrams-dominance-NRC-VAD-Lexicon-v2.1.txt
  ... and 4 more under data/
src/    [24 files]
  src/config.py
  src/data/datasetsProj.py
  src/data/features.py
  src/data/preprocessing.py
  src/training/evaluation.py
  src/training/losses.py
  src/training/run_multi_seed.py
  src/training/run_sgee_seeds.py
  src/training/stage2_losses.py
  src/training/stage2_trainer.py
  src/training/trainStage1.py
  src/training/trainStage2.py
  src/training/train_baseline_E.py
  src/training/train_baseline_F.py
  src/training/train_baseline_G.py
  ... and 9 more under src/
```

## README (verbatim)

# SGEE — Semantically Grounded Emotion Embeddings

A multi-stage framework for learning emotion-aware text representations by grounding transformer embeddings in psycholinguistic VAD (Valence-Arousal-Dominance) space.

> **This release covers Stage 1 (Emotion Classification) and Stage 2 (VAD Grounding with Contrastive Learning).**

---

## Overview

SGEE extends a RoBERTa backbone with **lexicon-derived** and **surface-level** features, then projects the concatenated representation into a unified **SGEE embedding space**. Training proceeds in stages:

| Stage | Objective | Dataset | Key Losses |
|-------|-----------|---------|------------|
| **Stage 1** | Multi-label emotion classification | GoEmotions (27+1 classes) | BCE, VAD regression |
| **Stage 2** | VAD grounding + contrastive alignment | GoEmotions | Prototype-contrastive, weighted VAD, adaptive emotion |

### Architecture

```
Text ─→ RoBERTa Backbone ─→ [CLS] hidden state
                                     │
Lexicon Features ──────────────────→ ╋ ─→ SGEE Projection ─→ SGEE Embedding (512d)
Surface Features ──────────────────→ ╋           │                    │
                                                 │                    │
                                          Emotion Head          VAD Head
                                        (28-class BCE)      (3d regression)
```

**Stage 2** adds:
- **VAD Prototypes** — cluster centers in VAD space (e.g. *positive-calm*, *negative-intense*, *neutral-balanced*)
- **Contrastive Learning** — aligns SGEE embeddings with their nearest VAD prototype
- **Adaptive Loss Balancing** — dynamically weights emotion vs VAD losses

---

## Repository Structure

```
sgee_aca_project/
├── src/
│   ├── config.py                    # Configuration dataclasses (model, training, paths)
│   ├── models/
│   │   ├── backbone.py              # RoBERTa backbone wrapper
│   │   ├── full_model.py            # Complete SGEEModel
│   │   ├── heads.py                 # Emotion (multi-label) + VAD (regression) heads
│   │   ├── prototypes.py            # VAD prototype manager
│   │   └── sgee_projection.py       # SGEE projection layer
│   ├── data/
│   │   ├── datasetsProj.py          # Dataset classes & dataloader creation
│   │   ├── features.py              # Lexicon (NRC-VAD) + surface feature extraction
│   │   └── preprocessing.py         # Text cleaning pipeline
│   ├── training/
│   │   ├── trainStage1.py           # Stage 1 training script
│   │   ├── trainStage2.py           # Stage 2 training script
│   │   ├── trainer.py               # Core SGEETrainer (metrics, checkpointing)
│   │   ├── losses.py                # Multi-task loss (Stage 1)
│   │   ├── stage2_losses.py         # Contrastive + VAD losses (Stage 2)
│   │   ├── stage2_trainer.py        # Stage 2 trainer with prototype management
│   │   ├── run_multi_seed.py        # Multi-seed evaluation runner
│   │   ├── run_sgee_seeds.py        # SGEE seed runner
│   │   ├── train_stage1_enhanced.py # Enhanced Stage 1 with extra monitoring
│   │   └── train_baseline_*.py      # Ablation baselines (E, F, G, H, RoBERTa, etc.)
│   └── utils/
│       ├── config.py                # Config utilities
│       └── logging.py               # Logging setup
├── configs/
│   └── base_config.yaml             # Base YAML configuration
├── data/                            # (not tracked — see Data Setup below)
│   └── lexicons/                    # NRC-VAD Lexicon v2.1
├── models/                          # (not tracked — trained checkpoints)
├── testRuns.py                      # Stage 1 → Stage 2 weight compatibility checker
├── .gitignore
└── README.md
```

---

## Setup

### Prerequisites

- Python 3.9+
- CUDA-capable GPU recommended (training is feasible on CPU but slow)

### Installation

```bash
git clone https://github.com/<your-username>/sgee_aca_project.git
cd sgee_aca_project
pip install -r requirements.txt
```

### Dependencies

```
torch>=2.0
transformers>=4.30
numpy
pandas
scikit-learn
tqdm
pyyaml
```

> A `requirements.txt` will be generated separately. Core dependencies are PyTorch, HuggingFace Transformers, and scikit-learn.

### Data Setup

**GoEmotions** (required for both stages):

1. Download the [GoEmotions dataset](https://github.com/google-research/google-research/tree/master/goemotions)
2. Run the preprocessing script:
   ```bash
   python data/preprocess_goemotions.py
   ```
3. Preprocessed files will be saved to `data/processed/goemotionsPreprocessed/`

**NRC-VAD Lexicon** (required for feature extraction):

1. Download the [NRC-VAD Lexicon v2.1](https://saifmohammad.com/WebPages/nrc-vad.html)
2. Place it in `data/lexicons/NRC-VAD-Lexicon-v2.1/`

---

## Training

### Stage 1 — Emotion Classification

```bash
python src/training/trainStage1.py
```

The script offers interactive mode selection:
- **Quick test** — 100 samples, 2 epochs (~5 min)
- **Development** — 2000 samples, 3 epochs (~30 min)
- **Full training** — entire GoEmotions, 6 epochs (~3-5 hours on GPU)

Trained checkpoints are saved to `models/`.

### Stage 2 — VAD Grounding with Contrastive Learning

```bash
python src/training/trainStage2.py
```

Stage 2 loads the Stage 1 checkpoint and adds:
- VAD prototype contrastive alignment
- Enhanced VAD regression with dimension weighting
- Adaptive emotion loss balancing

> **Note:** Update the `stage1_model_path` in `trainStage2.py` to point to your trained Stage 1 checkpoint.

### Baselines

Run ablation baselines for comparison:

```bash
python src/training/train_baseline_roberta_plain.py    # Plain RoBERTa
python src/training/train_baseline_roberta_multitask.py # RoBERTa multitask
python src/training/train_baseline_E.py                 # Baseline E
python src/training/train_baseline_F.py                 # Baseline F
python src/training/train_baseline_G.py                 # Baseline G
python src/training/train_baseline_H.py                 # Baseline H
```

---

## Configuration

All hyperparameters are managed through dataclasses in `src/config.py`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `backbone_name` | `roberta-base` | Transformer backbone |
| `sgee_embedding_dim` | `512` | SGEE embedding dimensionality |
| `batch_size` | `16` | Training batch size |
| `backbone_lr` | `2e-5` | Backbone learning rate |
| `heads_lr` | `1e-4` | Head learning rate |
| `max_epochs` | `6` | Maximum training epochs |
| `emotion_num_classes` | `28` | 27 GoEmotions + neutral |

Override defaults via `configs/base_config.yaml` or programmatically.

---

## Key Design Decisions

- **Lexicon + Surface features**: Raw transformer representations are concatenated with NRC-VAD lexicon scores and surface-level persuasion indicators before projection.
- **Multi-stage training**: Stage 1 establishes emotion classification capability; Stage 2 grounds the embedding space in psycholinguistic VAD geometry.
- **Prototype-based contrastive learning**: VAD prototypes define meaningful regions in affective space. Contrastive loss encourages embeddings to cluster near their assigned prototype.
- **Adaptive loss balancing**: Stage 2 dynamically adjusts the relative weight of emotion and VAD losses based on training progress.

---

## License

This project is for academic and research purposes.

---

## Acknowledgments

- [GoEmotions](https://github.com/google-research/google-research/tree/master/goemotions) — Google Research
- [NRC-VAD Lexicon](https://saifmohammad.com/WebPages/nrc-vad.html) — Saif Mohammad
- [RoBERTa](https://huggingface.co/roberta-base) — Meta AI / HuggingFace

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/SGEE-Semantically-Grounded-Emotion-Embeddings*
