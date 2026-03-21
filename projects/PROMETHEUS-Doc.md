# PROMETHEUS-Doc

> Document-image binarization on the DIBCO benchmark — a seven-phase investigation of physics-informed Neural ODEs, ASPP, DINOv2 features, FFT branches, and two-stage boundary refinement.

**Repository:** [`AmeyaBorkar/PROMETHEUS`](https://github.com/AmeyaBorkar/PROMETHEUS)  
**Category:** ML Research / Computer Vision  
**Visibility:** Private  
**Primary language:** Python  
**Default branch:** `main`  
**License:** —  
**Created:** 2026-02-27  
**Last pushed:** 2026-03-31  
**Metadata updated:** 2026-03-31  
**Size (GitHub reported):** 517,373 KB  

---

## What it is (one-paragraph version)

Document-image binarization on the DIBCO benchmark — a seven-phase investigation of physics-informed Neural ODEs, ASPP, DINOv2 features, FFT branches, and two-stage boundary refinement.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Python | 1,019,177 | 97.0% |
| HTML | 25,523 | 2.4% |
| Jupyter Notebook | 5,918 | 0.6% |
| Dockerfile | 243 | 0.0% |

## File tree

- Total entries indexed: **14354** (14146 files, 208 directories)

```
.gitignore  (622 B)
README.md  (11 KB)
ROADMAP.md  (10 KB)
pyproject.toml  (495 B)
requirements.txt  (1 KB)
run_finetune_experiments.py  (9 KB)
run_overnight.py  (7 KB)
setup_project.py  (46 KB)
checkpoints/    [42 files]
  checkpoints/cafr/metrics.csv
  checkpoints/cafr_eval/metrics.csv
  checkpoints/dgi/metrics.csv
  checkpoints/dps/metrics.csv
  checkpoints/dps_enhanced/dps_config.json
  checkpoints/dps_enhanced/metrics.csv
  checkpoints/dps_enhanced_direct/dps_config.json
  checkpoints/dps_enhanced_direct/metrics.csv
  checkpoints/nre/metrics.csv
  checkpoints/pretrained/.gitkeep
  checkpoints/stage_b/.gitkeep
  checkpoints/stage_b_dde/dps_config.json
  checkpoints/stage_b_dde/metrics.csv
  checkpoints/stage_b_dibco/dps_config.json
  checkpoints/stage_b_dibco/metrics.csv
  ... and 27 more under checkpoints/
colab/    [1 files]
  colab/COLAB_PLAN.md
configs/    [4 files]
  configs/evaluation/benchmarks.yaml
  configs/inference/default.yaml
  configs/training/stage_b_pretrain.yaml
  configs/training/stage_d_joint.yaml
data/    [17 files]
  data/__init__.py
  data/augmentation/__init__.py
  data/augmentation/online_degradation.py
  data/degradation/__init__.py
  data/degradation/physics_models/__init__.py
  data/degradation/physics_models/degradation_engine.py
  data/loaders/__init__.py
  data/loaders/dataset_loaders.py
  data/loaders/patch_dataset.py
  data/loaders/unified_loader.py
  data/preprocessing/__init__.py
  data/preprocessing/cache.py
  data/preprocessing/transforms.py
  data/synthetic/__init__.py
  data/synthetic/generators/__init__.py
  ... and 2 more under data/
deploy/    [3 files]
  deploy/Dockerfile
  deploy/api_server.py
  deploy/static/index.html
docs/    [8 files]
  docs/api/.gitkeep
  docs/paper/METRICS.md
  docs/paper/PAPER_STRUCTURE.md
  docs/paper/PROJECT_JOURNEY.md
  docs/paper/TRAINING_LOG.md
  docs/paper/figures/.gitkeep
  docs/paper/references.bib
  docs/paper/tables/.gitkeep
eval/    [13907 files]
  eval/__init__.py
  eval/ablations/.gitkeep
  eval/benchmarks/.gitkeep
  eval/eval_ensemble.py
  eval/eval_fullres.py
  eval/generate_predictions.py
  eval/human_study/.gitkeep
  eval/metrics.py
  eval/results/ablation_summary.json
  eval/results/stage_b_dps_dibco/ground_truth/0000.png
  eval/results/stage_b_dps_dibco/ground_truth/0001.png
  eval/results/stage_b_dps_dibco/ground_truth/0002.png
  eval/results/stage_b_dps_dibco/ground_truth/0003.png
  eval/results/stage_b_dps_dibco/ground_truth/0004.png
  eval/results/stage_b_dps_dibco/ground_truth/0005.png
  ... and 13892 more under eval/
genome/    [6 files]
  genome/renderers/__init__.py
  genome/renderers/classical_renderer.py
  genome/schema/__init__.py
  genome/schema/document_genome.py
  genome/validators/__init__.py
  genome/validators/schema_validator.py
inference/    [5 files]
  inference/__init__.py
  inference/batch_restore.py
  inference/export_genome.py
  inference/export_layout.py
  inference/restore.py
models/    [82 files]
  models/__init__.py
  models/aoss/__init__.py
  models/aoss/cycle_consistency/__init__.py
  models/aoss/cycle_consistency/cycle.py
  models/aoss/discriminators/__init__.py
  models/aoss/discriminators/oracle.py
  models/aoss/model.py
  models/binarization/__init__.py
  models/binarization/binarization_head.py
  models/cafr/__init__.py
  models/cafr/confidence/__init__.py
  models/cafr/confidence/estimator.py
  models/cafr/model.py
  models/cafr/router/__init__.py
  models/cafr/router/routing_policy.py
  ... and 67 more under models/
notebooks/    [1 files]
  notebooks/stage_b_training.ipynb
scripts/    [11 files]
  scripts/cache_dps_outputs.py
  scripts/cache_raw_passthrough.py
  scripts/diagnose_adjoint.py
  scripts/download_datasets/download_all.py
  scripts/overnight_stage_d.py
  scripts/overnight_train.py
  scripts/preprocess_augmented.py
  scripts/preprocess_datasets.py
  scripts/rebuild_manifests.py
  scripts/validate_direct_v3.py
  scripts/visualization/.gitkeep
tests/    [28 files]
  tests/__init__.py
  tests/integration/__init__.py
  tests/integration/test_api.py
  tests/integration/test_inference.py
  tests/test_prometheus_unet.py
  tests/test_prometheus_v3.py
  tests/unit/__init__.py
  tests/unit/test_data/__init__.py
  tests/unit/test_data/test_degradation.py
  tests/unit/test_data/test_genome_sampler.py
  tests/unit/test_data/test_pipeline.py
  tests/unit/test_eval/__init__.py
  tests/unit/test_eval/test_metrics.py
  tests/unit/test_genome/__init__.py
  tests/unit/test_genome/test_renderer.py
  ... and 13 more under tests/
training/    [23 files]
  training/__init__.py
  training/callbacks/__init__.py
  training/callbacks/checkpointing.py
  training/callbacks/early_stopping.py
  training/callbacks/lr_monitor.py
  training/loggers/__init__.py
  training/loggers/tb_logger.py
  training/loggers/wandb_logger.py
  training/stage_a_data.py
  training/stage_b_pretrain.py
  training/stage_c_physics.py
  training/stage_d_binarize.py
  training/stage_d_fast.py
  training/stage_d_joint.py
  training/stage_e_selfsup.py
  ... and 8 more under training/
```

## README (verbatim)

<p align="center">
  <h1 align="center">PROMETHEUS-Doc</h1>
  <p align="center">
    <strong>Physics-Reversible Omni-Modal Encoding via Temporal Hierarchical Emergent Understanding of Symbolic Documents</strong>
  </p>
  <p align="center">
    <em>Document Binarization | Differentiable Physics | Neural ODE | Boundary-Aware Refinement</em>
  </p>
  <p align="center">
    <a href="#architecture">Architecture</a> |
    <a href="#results">Results</a> |
    <a href="#key-findings">Key Findings</a> |
    <a href="#quick-start">Quick Start</a> |
    <a href="#project-structure">Project Structure</a>
  </p>
</p>

---

## Overview

**PROMETHEUS-Doc** is a comprehensive investigation into document image binarization on the DIBCO benchmark. The project systematically evaluates physics-informed approaches (Neural ODEs), modern segmentation architectures (ASPP, D-LinkNet), foundation model features (DINOv2), frequency-domain analysis (FFT), and two-stage boundary refinement -- providing the most thorough ablation study of what actually matters for document binarization.

**Best result: 19.74 dB PSNR** on DIBCO 2017 (fine-tuned DP-LinkNet with our augmentation pipeline). Our from-scratch model (PrometheusV3) achieves **18.06 dB**.

### The Journey (7 Phases)

| Phase | Approach | PSNR | Key Insight |
|:-----:|----------|-----:|-------------|
| 1-3 | Pixel-space Neural ODE + Sauvola | 15.97 | ODE destroys spatial information |
| 4 | ODE-as-bottleneck U-Net | 15.21 | 512->64ch compression kills capacity |
| **5** | **ASPP + dense patches (V3)** | **18.06** | **Dense patches are the #1 factor (+2.72 dB)** |
| 5 | V3 + ODE side-branch | 17.86 | Physics provides no benefit (-0.07 dB) |
| 5 | V3 + Freq + DINOv2 | 17.68 | Multi-view features don't help (-0.25 dB) |
| 6 | Fine-tuned DP-LinkNet | 19.74 | Training recipe matters more than architecture |
| **7** | **Two-stage boundary refinement** | **TBD** | **BARRN targets boundary errors (in progress)** |

## Architecture

### PrometheusV3 (From-Scratch Best: 18.06 dB)

```
INPUT: degraded document (B, 3, 256, 256)

ENCODER (ResNet34 pretrained, Output Stride 16):
  stem+pool  -> (B,  64,  H/4,  W/4)   [e1] skip
  layer2     -> (B, 128,  H/8,  W/8)   [e2] skip
  layer3     -> (B, 256, H/16, W/16)   [e3] skip
  layer4     -> (B, 512, H/16, W/16)   [e4] (stride removed for OS16)

ASPP BOTTLENECK (DeepLabV3+ style):
  1x1 conv + 3x3 dilated @6,12,18 + GAP -> (B, 256, H/16, W/16)

U-NET DECODER:
  cat(bottleneck, e3) -> 256 -> up + cat(e2) -> 128 -> up + cat(e1) -> 64

BINARY HEAD:
  Conv(64,32,3) + BN + ReLU + Conv(32,1,1) -> sigmoid
```

**27.9M params** | BCE + Dice loss | 4,994 dense 256x256 patches/epoch

### Two-Stage Refinement: BARRN (In Progress)

```
Stage 1: Pretrained DP-LinkNet (frozen, 19.62 dB)
  -> initial binary probability map

Stage 2: BARRN (481K params, trainable)
  Input: RGB image + Stage 1 probability (4 channels)
  Encoder: 3-level CNN (32/64/128 channels)
  Bottleneck: Sobel boundary detection -> spatial attention
  Decoder: skip-connected upsampling
  Output: residual DELTA correction
  
  refined_logit = stage1_logit + delta
```

Novel contribution: boundary-aware spatial attention focuses corrections on text edges where DRD errors concentrate.

### D-LinkNet (Reimplemented: 28.78M params)

Exact reimplementation of DP-LinkNet architecture:
- **HDC**: Cascaded dilated convolutions (rates 1, 2, 4) with dense additive skip
- **SPP**: 3-scale max pooling (2x2, 3x3, 5x5), each reduced to 1 channel
- **LinkNet decoder**: Additive skip connections (not concatenation), transposed conv upsampling

## Results

### DIBCO 2017 Benchmark (20 images, held-out)

| Method | PSNR (dB) | F-measure | Notes |
|--------|:---------:|:---------:|-------|
| DP-LinkNet (published) | 20.83 | 0.928 | Includes 8-fold TTA |
| **DP-LinkNet (our eval, no TTA)** | **19.62** | **0.939** | **Pretrained weights, our pipeline** |
| DP-LinkNet + our fine-tune | 19.74 | 0.942 | +0.12 from augmentation |
| **PrometheusV3 (ours, from scratch)** | **18.06** | **0.914** | **ASPP + dense patches** |
| V3 + ODE side-branch | 17.86 | 0.910 | Physics doesn't help |
| V3 + Freq + DINOv2 | 17.68 | 0.908 | Multi-view doesn't help |
| Sauvola baseline | 15.13 | -- | Classical |

### Comprehensive Ablation (All Approaches Tested)

| Approach | On V3 (18.06) | On DP-LinkNet (19.62) | Verdict |
|----------|:-------------:|:---------------------:|---------|
| Dense patch extraction | **+2.72 dB** | (already used) | **Dominant factor** |
| Fine-tune (unfrozen encoder) | +0.13 | +0.12 | Modest, consistent |
| ODE side-branch (physics) | -0.07 | N/A | No benefit |
| Frequency branch (FFT) | -0.25* | +0.03 | Negligible |
| DINOv2 branch (ViT-B/14) | -0.25* | +0.00 | Zero |
| Multi-view (freq + DINOv2) | -0.25* | -0.04 | Hurts |
| TTA (8-fold soft voting) | -0.21 | -0.42 | Hurts both |
| Ensemble (DP-LinkNet + V3) | -- | +0.00 | No benefit |

*Tested together on V3, not individually.

### SOTA Calibration

Published DIBCO numbers often include TTA. Fair comparison:

| Model | Without TTA | Published (with TTA) |
|-------|:-----------:|:--------------------:|
| DP-LinkNet | 19.62 | 20.83 (+1.21 from TTA) |
| Our V3 | 18.06 | N/A |
| **Real gap** | **1.56 dB** | |

## Key Findings

1. **Dense patch extraction is the single most important technique** -- extracting 4,994 native-resolution patches from 86 images (vs 86 resized whole-page samples) gives +2.72 dB. More impactful than any architectural change.

2. **Physics-informed ODE provides zero benefit** for document binarization when the segmentation backbone is strong. Tested as bottleneck (Phase 4), side-branch (Phase 5), and on DP-LinkNet backbone -- consistently +/-0 dB.

3. **Foundation model features (DINOv2) don't help** -- even 86.6M frozen ViT-B/14 parameters with cross-attention fusion add nothing measurable on DIBCO.

4. **Published SOTA numbers include TTA** -- DP-LinkNet's 20.83 dB drops to 19.62 without TTA. Always compare under the same eval protocol.

5. **Training recipe matters more than architecture** -- our D-LinkNet reimplementation (same architecture) trained from scratch gets 16.74 dB vs their pretrained 19.62. The gap is in normalization, augmentation, LR schedule, and training duration.

6. **Soft-voting TTA hurts both architectures** -- averaging probabilities across augmented views dilutes predictions. The published TTA uses hard voting (sum + threshold at 62.5% agreement).

## Quick Start

### Prerequisites

- Python >= 3.10
- CUDA-capable GPU (12+ GB VRAM recommended)
- PyTorch with CUDA support

### Installation

```bash
git clone https://github.com/AmeyaBorkar/PROMETHEUS.git
cd PROMETHEUS
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu128
pip install torchdiffeq tqdm click pillow numpy scikit-image doxapy
```

### Training

```bash
# PrometheusV3 (ASPP baseline, from scratch)
python training/stage_p_v3.py --epochs 200 --batch-size 8 \
  --save-dir checkpoints/prometheus_v3

# D-LinkNet variant
python training/stage_p_v3.py --decoder-type dlink --epochs 200 --batch-size 8 \
  --save-dir checkpoints/prometheus_v3_dlink

# Two-stage refinement (requires pretrained DP-LinkNet weights)
python training/stage_p_v3_refine.py --epochs 80 --batch-size 8 \
  --save-dir checkpoints/two_stage

# Fine-tune pretrained DP-LinkNet with novel branches
python training/stage_p_v3_finetune_dplinknet.py --use-freq --epochs 100 \
  --save-dir checkpoints/dplinknet_ft_freq
```

### Evaluation

```bash
# PrometheusV3
python eval/eval_fullres.py \
  --ckpt checkpoints/prometheus_v3/best.pt \
  --ckpt-format prometheus_v3 --binarize learned --show-baseline \
  --out-dir eval/results_v3

# Two-stage model
python eval/eval_fullres.py \
  --ckpt checkpoints/two_stage/best.pt \
  --ckpt-format two_stage --binarize learned --show-baseline \
  --out-dir eval/results_two_stage

# Pretrained DP-LinkNet
python eval/eval_fullres.py \
  --ckpt checkpoints/dplinknet_pretrained/best.pt \
  --ckpt-format dplinknet_raw --binarize learned --show-baseline \
  --out-dir eval/results_dplinknet

# Run all fine-tune experiments overnight
python run_finetune_experiments.py
```

## Project Structure

```
PROMETHEUS/
|-- models/prometheus_v3/         # Main model package
|   |-- model.py                  #   PrometheusV3 (supports unet/dlink decoders)
|   |-- encoder.py                #   ResNet34 with OS16 support
|   |-- aspp.py                   #   Atrous Spatial Pyramid Pooling
|   |-- decoder.py                #   U-Net decoder (OS16)
|   |-- dlink_center.py           #   DP-LinkNet HDC + SPP center block
|   |-- linknet_decoder.py        #   LinkNet decoder (additive skips)
|   |-- ode_side_branch.py        #   Neural ODE parallel branch
|   |-- frequency_branch.py       #   FFT spectral feature extraction
|   |-- dino_branch.py            #   Frozen DINOv2 ViT-B/14 features
|   |-- cross_attention.py        #   Multi-head cross-attention fusion
|   |-- boundary_refinement.py    #   BARRN (two-stage refinement)
|
|-- training/
|   |-- stage_p_v3.py             #   Main V3 training (dense patches)
|   |-- stage_p_v3_refine.py      #   Two-stage refinement training
|   |-- stage_p_v3_finetune_dplinknet.py  # Fine-tune pretrained DP-LinkNet
|   |-- trainer.py                #   Generic training loop
|   |-- callbacks/                #   Early stopping, checkpointing
|
|-- data/loaders/
|   |-- patch_dataset.py          #   Dense overlapping patch extraction
|   |-- dataset_loaders.py        #   DIBCO, SD7K, IAM, FUNSD, CORD
|
|-- eval/
|   |-- eval_fullres.py           #   Full-res tiled eval + TTA
|   |-- eval_ensemble.py          #   Multi-model ensemble evaluation
|
|-- models/dps/                   # Neural ODE physics simulator
|-- models/prometheus_unet/       # Phase 4 ODE-bottleneck U-Net
|-- docs/paper/                   # Training log, metrics, journey, paper structure
|-- run_finetune_experiments.py   # Overnight experiment suite
```

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Framework | PyTorch 2.10 (CUDA 12.8) |
| Neural ODEs | torchdiffeq |
| Foundation Model | DINOv2 ViT-B/14 (frozen, 86.6M params) |
| Evaluation | doxapy (official DIBCO metrics) |
| Encoder | torchvision ResNet34 (ImageNet pretrained) |
| Hardware | NVIDIA RTX 5070 Ti Laptop (12 GB VRAM) |

## Citation

```bibtex
@software{prometheus_doc,
  title     = {PROMETHEUS-Doc: Comprehensive Investigation of Physics-Informed
               and Multi-View Approaches for Document Image Binarization},
  author    = {Borkar, Ameya},
  year      = {2026},
  url       = {https://github.com/AmeyaBorkar/PROMETHEUS}
}
```

## License

TBD

---

<p align="center">
  <sub>Built by <a href="https://github.com/AmeyaBorkar">Ameya Borkar</a></sub>
</p>

---

*Generated 2026-05-02 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/PROMETHEUS*
