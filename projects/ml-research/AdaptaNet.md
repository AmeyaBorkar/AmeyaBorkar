# AdaptaNet

> A PyTorch research framework for a hybrid CNN→Transformer image model: a "dynamic" CNN backbone (deformable conv, selective kernel, log-polar, coordinate attention) feeds a gated channel projection into an adaptive Transformer encoder (input-dependent head pruning, Mixture-of-Experts FFN, context-conditioned LayerNorm, heterogeneous attention), ending in attentive pooling → classification head + embeddings. It ships a large standalone metric-learning loss library (12 loss criteria + 4 dynamic loss-weighting schemes + 4 training utilities) and a multi-dataset training pipeline (CIFAR-100 / CUB-200 / SOP). Notably, the actual `train.py` uses only plain cross-entropy or a single ArcFace head — the 20+-loss `LossIntegrationSystem` is built and unit-tested but never wired into training.

| Field | Value |
|---|---|
| Repository | https://github.com/AmeyaBorkar/AdaptaNet |
| Visibility | Public |
| Category | ml-research |
| Primary language(s) | Python (100%) |
| Local path | `C:\Users\ameya\Documents\AdaptaNet` |
| Default branch | `main` |
| Lines of code (computed) | 14,276 lines of Python across 33 `.py` files (excluding `data/`, `__pycache__`, `.pytest_cache`) |
| Source files (computed) | 33 Python files: `dynamic_cnn/` (6), `transformer/` (10), `losses/` (7), `scripts/` (6), `tests/` (2), root (`model.py`, `__init__.py`) |
| Key dependencies | `torch>=2.0`, `torchvision>=0.15`, `numpy>=1.24`, `einops>=0.6`, `tqdm`, `requests`, `Pillow`, `gdown` |
| License | None — no `LICENSE` file in repo. README states "for research and educational purposes." |
| Last commit | `23ffc3a` — "Add ablation, visualization, and benchmark scripts" (2026-03-05); working tree has 2 uncommitted items: a 1-line typo fix in `scripts/train.py` (`total_mem`→`total_memory`) and an untracked `plan.md` |

### Computed size breakdown (Python LOC, source dirs)

| Area | LOC | Notable files |
|---|---|---|
| `transformer/` | 8,328 | `integration.py` (2,867), `encoder.py` (1,521), `pyramid.py` (1,211), `layers.py` (703), `attention.py` (563), `hetero_attention.py` (496), `config.py` (440), `moe.py` (253), `norm.py` (179) |
| `scripts/` | 2,051 | `train.py` (646), `visualize.py` (532), `benchmark.py` (291), `preprocess_datasets.py` (261), `ablate.py` (183), `download_datasets.py` (138) |
| `dynamic_cnn/` | 1,849 | `layers.py` (600), `attention.py` (404), `backbone.py` (337), `blocks.py` (323), `norm.py` (108) |
| `losses/` | 1,397 | `integration.py` (365), `classification.py` (221), `representation.py` (214), `consistency.py` / `utils.py` (182 each), `weighting.py` (171) |
| `tests/` | 443 | `test_fixes.py` (245), `test_model.py` (198) |
| root | 208 | `model.py` (198), `__init__.py` (10) |

---

## What it actually is

AdaptaNet is a **single-image classification / metric-learning research model** assembled in `model.py` (the `AdaptaNet` class). The pipeline, verified from the forward pass:

```
[B,3,H,W]
  → DynamicCNNBackbone           (stem + 4 stages of DynamicResidualBlocks; returns dict of stem/stage_1..4 features)
  → CrossStageFeatureHierarchy   (bidirectional FPN over the 5 feature maps; optional, for ablation)
  → AdaptiveChannelProjection    (gated 1x1 conv: value(x) * sigmoid(gate(x)) → embed_dim)
  → flatten H'·W' → [B, L, embed_dim]
  → AdaptiveTransformerEncoder   (depth blocks; MoE/context-norm/hetero-attn on specific layers)
  → AttentivePool                (MLP-scored softmax over tokens → [B, embed_dim])
  → LayerNorm → Linear head
  → {"logits": [B,num_classes], "embeddings": [B,embed_dim]}
```

It is explicitly research-grade: a `ROADMAP.md` lays out a planned paper (baselines vs ResNet-50/ViT/Swin, ablation tables, MoE/head analysis, target venue CVPR/ECCV/WACV/BMVC), and `plan.md` describes an intended training strategy. The "more config knobs than sense" characterization is accurate — `AdaptiveTransformerConfig` alone exposes ~50 constructor parameters across five module groups, many of which (pyramid, integration, token reduction, axial attention) are not exercised by the default `AdaptaNet` assembly.

Target tasks/datasets (from `scripts/`): **CIFAR-100** (100-class classification), **CUB-200-2011** (200-class fine-grained classification), and **Stanford Online Products / SOP** (22,634-class image retrieval, evaluated by Recall@K). Images are preprocessed to 224×224 with ImageNet normalization and cached as `.pt` tensors.

---

## Architecture & how it's structured

Annotated source tree (source dirs only; `data/` is git-ignored and holds downloaded/preprocessed datasets):

```
AdaptaNet/
├── model.py                  # AdaptaNet wrapper + AdaptiveChannelProjection + AttentivePool
├── __init__.py               # package docstring only (no exports)
│
├── dynamic_cnn/              # "Dynamic" CNN backbone (1,849 LOC)
│   ├── norm.py               # AdaptiveNorm (InstanceNorm↔LayerNorm switch), ScaleAdaptiveNorm
│   ├── attention.py          # PositionalEncoding, CoordinateAttention, CoordinateSEBlock,
│   │                         #   AdaptiveCoordinateSEBlock, MemoryEfficientAttention,
│   │                         #   CrossScaleAttentionFusion
│   ├── layers.py             # HyperNetwork, DynamicConv2d, FactorizedConv2d,
│   │                         #   SelectiveKernel, DeformableConv2d, LogPolarConv
│   ├── blocks.py             # StochasticDepth, DynamicResidualBlock, DynamicExpertMixture,
│   │                         #   CrossStageFeatureHierarchy
│   └── backbone.py           # DynamicCNNBackbone, MultiScaleDynamicCNN
│
├── transformer/             # Adaptive Transformer (8,328 LOC)
│   ├── config.py             # AdaptiveTransformerConfig (~50 knobs)
│   ├── attention.py          # EfficientSelfAttention (linear/softmax), RelativePositionalBias, AxialAttention
│   ├── layers.py             # AdaptiveLayerNorm, SwiGLU, Mlp, StochasticDepth,
│   │                         #   TransformerEncoderLayer/Block, SpatialReduction, TokenReduction
│   ├── encoder.py            # AdaptiveMultiHeadAttention (head pruning), AdaptiveTransformerEncoder*
│   ├── moe.py                # MoEFFN (token-routed experts + load-balance aux loss)
│   ├── norm.py               # ContextConditionedLayerNorm
│   ├── hetero_attention.py   # HeterogeneousAttention (standard + linear + adaptive, gated)
│   ├── pyramid.py            # MultiScalePyramidTransformer + supporting modules (NOT used by AdaptaNet)
│   └── integration.py        # UnifiedAdaptiveTransformer + fusion machinery (NOT used by AdaptaNet)
│
├── losses/                  # Metric-learning loss library (1,397 LOC)
│   ├── classification.py     # ArcFace, CosFace, AdaCos, Circle
│   ├── representation.py     # SupCon, Triplet, BarlowTwins, VICReg
│   ├── consistency.py        # Viewpoint, Illumination, Temporal, Structural
│   ├── weighting.py          # DynamicPrioritization, CurriculumIntegration, UncertaintyWeighting, GradientNormalization
│   ├── utils.py              # MarginAnnealing, ClassBalancedWeighting, HardNegativeMining, TemperatureScheduling
│   └── integration.py        # LossIntegrationSystem, LocationRecognitionLossModule
│
├── scripts/
│   ├── download_datasets.py  # CIFAR-100 (torchvision), CUB-200 (HTTP), SOP (gdown)
│   ├── preprocess_datasets.py# resize→224, ImageNet-normalize, save .pt
│   ├── train.py              # training loop (AMP, EMA, CutMix/Mixup, cosine LR)
│   ├── ablate.py             # spawns train.py subprocesses per component-disabled variant
│   ├── benchmark.py          # params / FLOPs(est) / memory / throughput
│   └── visualize.py          # attention maps, MoE utilization, head importance, t-SNE
│
├── tests/                   # pytest: test_model.py, test_fixes.py
├── README.md  ROADMAP.md  plan.md  requirements.txt  .gitignore
```

`*` Note: `transformer/__init__.py` exports `AdaptiveTransformerEncoder`, `MultiScalePyramidTransformer`, and `UnifiedAdaptiveTransformer`, but `model.py` instantiates only `AdaptiveTransformerEncoder`. The pyramid (`pyramid.py`, 1,211 LOC) and unified integration (`integration.py`, 2,867 LOC — the single largest file in the repo) are fully implemented but dead code with respect to the assembled model. `use_pyramid` defaults to `False` in the config.

**Model assembly (`AdaptaNet.__init__`, defaults):** `cnn_layers=[2,4,6,2]`, CNN width progression `base_width × {1,2,4,8,16}`, `target_channels=512`, `embed_dim=256`, `transformer_depth=6`, `num_heads=8`. The transformer config the wrapper builds always turns on the adaptive features and restricts them to specific layers:

- `adaptive_heads=True` (all layers)
- `use_moe=True`, `moe_layers=[2,5]` (only when depth≥6)
- `use_context_norm=True`, `norm_layers=[1,3,5]` (only when depth≥6)
- `use_hetero_attn=True`, `hetero_attn_layers=[0,3]` (only when depth≥4)

---

## Model components (in code)

### Deformable CNN backbone (`dynamic_cnn/`)

`DynamicCNNBackbone` (`backbone.py`): a stem (two 3×3 convs, stride-2 then stride-1, `AdaptiveNorm`+SiLU) → 4 stages of `DynamicResidualBlock` each capped with a `ScaleAdaptiveNorm` → a 1×1 `channel_aligner` to `target_channels` → an optional **log-polar branch** (`use_log_polar=True` default) whose globally-pooled features are concatenated and fused back via 1×1 conv. `forward` returns `{"out": aligned/fused, "features": {stem, stage_1..4}}`; `AdaptaNet` consumes the `features` dict, not `out`.

Key conv primitives in `layers.py`:

- **`DeformableConv2d`** — the deformable backbone op. Predicts per-pixel 2-coordinate **offsets** (zero-initialized) and a per-point **modulation mask** (bias-initialized to 0.5, passed through `hardsigmoid`), samples `kernel_size²` deformed locations via `F.grid_sample` (bilinear, align_corners), modulates and sums them, then runs a regular conv and an `AdaptiveCoordinateSEBlock`. This is a hand-rolled modulated deformable conv (DCNv2-style), not `torchvision.ops.deform_conv2d`.
- **`SelectiveKernel`** — parallel conv paths over `kernel_sizes=[3,5,7]` (large kernels go through `FactorizedConv2d`, i.e. depthwise 1×k + k×1 + pointwise), softmax-routed fusion (temperature 0.5) via global-pooled split attention, then `AdaptiveCoordinateSEBlock`.
- **`DynamicConv2d`** — `HyperNetwork` generates per-sample kernel weights from global context; applied as a grouped conv (groups = batch) and averaged across kernel sizes. (Implemented and unit-tested, but `DynamicResidualBlock` uses `DeformableConv2d`/`SelectiveKernel`, so `DynamicConv2d` is not on the default backbone path.)
- **`LogPolarConv`** — vectorized transform of features into log-radius/angle space (`num_angles=8`, `num_scales=4`) for rotation/scale awareness.

`DynamicResidualBlock` (`blocks.py`) is a bottleneck (1×1 reduce → 3×3 → 1×1 expand). The middle conv is `DeformableConv2d` when `use_deformable and stride==1`, else `SelectiveKernel`. Each block adds `AdaptiveCoordinateSEBlock`, `CoordinateAttention`, and `StochasticDepth` (drop-path). The backbone only deforms stride-1 blocks (first block of each stride-2 stage falls back to selective kernel).

`CrossStageFeatureHierarchy` (`blocks.py`) is a bidirectional FPN (top-down + bottom-up + learned 3×3 fusion per stage) over the five backbone feature maps; `AdaptaNet` takes its last fused output. There is also a `DynamicExpertMixture` (CNN-level MoE of residual blocks with top-k routing) and a `MultiScaleDynamicCNN` (multi-scale ensemble of backbones) — both implemented and tested but **not** used by `AdaptaNet`.

### MoE Transformer (`transformer/`)

The model uses `AdaptiveTransformerEncoder` (`encoder.py`), which builds `depth` × `AdaptiveTransformerEncoderBlock`, each wrapping an `AdaptiveTransformerEncoderLayer` with pre-norm, LayerScale, and stochastic depth. Per-layer feature selection is driven by the config's `is_moe_layer` / `is_context_norm_layer` / `is_hetero_attn_layer` checks.

**Experts / routing (`MoEFFN`, `moe.py`):**
- Default **8 experts**, **top-k = 2**, plus an optional **shared expert** (`use_shared_expert=True`, runs on every token, scaled ×0.5).
- Each expert is `Linear → SwiGLU → Dropout → Linear → Dropout`; hidden dim = `dim×mlp_ratio` (4× by default).
- Router: `Linear(dim, dim//2) → GELU → Linear(dim//2, num_experts)`; softmax with `gate_temp=0.1`; **jitter noise** (`moe_jitter_noise=0.1`) added to logits during training.
- Tokens are dispatched per-expert with a **capacity constraint** (`capacity_factor=1.25`); over-capacity tokens are subsampled via `torch.multinomial` on routing weights.
- **Load-balancing auxiliary loss** (`balance_loss_weight=0.01`) is computed during training, stored on the layer's `_moe_loss` buffer, and aggregated by `AdaptiveTransformerEncoder.get_moe_loss()`. (Note: `train.py` never calls `get_moe_loss()`, so the aux loss is computed but not added to the optimized objective.)

**Routing / gating elsewhere in the encoder:**
- **`AdaptiveMultiHeadAttention`** (`encoder.py`): a small MLP predicts per-input importance logits for each head; combined with a learnable `head_z` bias. Training uses **Gumbel-Softmax** head weighting; inference uses temperature softmax and can **prune to top-`max_heads`**. Tracks `head_usage` EMA stats (surfaced via `get_head_importance()`).
- **`HeterogeneousAttention`** (`hetero_attention.py`): runs up to three attention mechanisms in parallel — **standard** scaled-dot-product (+ relative positional bias), **linear** (ELU+1 kernel feature map, O(n)), and **adaptive** (reuses `AdaptiveMultiHeadAttention`) — and a learned router (Gumbel-Softmax, `gating_temperature=0.5`) weights and sums their outputs. Tracks `mechanism_usage`.
- **`ContextConditionedLayerNorm`** (`norm.py`): standard LayerNorm whose scale/bias are augmented by per-sample predictions from a bottleneck context encoder, each gated by a sigmoid scalar initialized to 0 (so it starts as ordinary LayerNorm).
- **`EfficientSelfAttention`** / **`AxialAttention`** (`attention.py`) and `TokenReduction`/`SpatialReduction` (`layers.py`) support linear attention, 2D axial attention, and progressive token downsampling — available via config but off by default in `AdaptaNet`.

### Heads / pooling (`model.py`)

- **`AdaptiveChannelProjection`** — content-aware gated 1×1 conv: `value(x) * sigmoid(gate(x))`, mapping fused CNN channels → `embed_dim`.
- **`AttentivePool`** — `Linear → Tanh → Linear(→1)` scores each token, softmax over the sequence, weighted sum → one vector per image (unit-tested to differ from mean pooling).
- Final `nn.LayerNorm` + `nn.Linear` classification head; returns both `logits` and the pooled `embeddings`.

---

## Loss functions

The README claims "**20+ purpose-built loss functions**." **Reality from code:** there are **12 actual differentiable loss criteria** (classes that return a training loss / logits), plus **4 dynamic loss-weighting modules** and **4 training-utility classes** (schedulers/miners that are not themselves losses), plus 2 orchestrators. Counting every loss-related class gives 22 classes; counting only things that compute a loss gives 12. So "20+" is reachable only by lumping weighting strategies and utilities in with the criteria — the genuine loss count is **12**.

The default training script (`train.py`) uses **none** of the representation/consistency losses — it optimizes plain cross-entropy (with label smoothing / Mixup soft targets) or a single `ArcFaceLoss` head. The full inventory below is what's implemented in `losses/`:

### Actual loss criteria (12)

| # | Loss | File | What it penalizes |
|---|------|------|-------------------|
| 1 | `ArcFaceLoss` | `losses/classification.py` | Additive **angular** margin on the target class (normalized embeddings/weights, cos(θ+m)); returns scaled logits for CE. |
| 2 | `CosFaceLoss` | `losses/classification.py` | Additive **cosine** margin (cos θ − m) on target class; scaled logits for CE. |
| 3 | `AdaCosLoss` | `losses/classification.py` | Cosine logits with a **dynamically adapted scale** `s` (from median angle / B_avg); no fixed margin. |
| 4 | `CircleLoss` | `losses/classification.py` | Unified pair-similarity optimization with per-pair weighting (`gamma=256`, `margin=0.25`). |
| 5 | `SupervisedContrastiveLoss` | `losses/representation.py` | Pulls same-class embeddings together / pushes others apart over the batch similarity matrix (temperature 0.07). |
| 6 | `TripletLoss` | `losses/representation.py` | Anchor–positive vs anchor–negative margin (0.3) with `batch_hard` / `batch_all` / `batch_semi` mining. |
| 7 | `BarlowTwinsLoss` | `losses/representation.py` | Cross-correlation matrix of two views toward identity (redundancy reduction). |
| 8 | `VICRegLoss` | `losses/representation.py` | Variance + invariance (MSE) + covariance regularization across two views. |
| 9 | `ViewpointConsistencyLoss` | `losses/consistency.py` | Maximizes cosine similarity between two viewpoint embeddings. |
| 10 | `IlluminationConsistencyLoss` | `losses/consistency.py` | Cosine + Euclidean distance between normal/illumination-changed embeddings. |
| 11 | `TemporalCoherenceLoss` | `losses/consistency.py` | Intra-location temporal consistency + inter-location separation (hinge margin) by location group ID. |
| 12 | `StructuralPreservationLoss` | `losses/consistency.py` | Preserves local (kNN, KL-div) and global (Frobenius) pairwise-distance structure input→output. |

### Loss-weighting strategies (4) — combine multiple losses, not losses themselves

| Module | File | Role |
|---|---|---|
| `UncertaintyWeighting` | `losses/weighting.py` | Homoscedastic-uncertainty weighting via learnable log-variances. |
| `GradientNormalization` | `losses/weighting.py` | Weights inversely to EMA gradient magnitudes. |
| `DynamicPrioritization` | `losses/weighting.py` | Weights by performance gap to target metrics (`gap^alpha`). |
| `CurriculumIntegration` | `losses/weighting.py` | Sigmoid ramp-in of each loss at milestone steps. |

### Training utilities (4) — schedulers/miners, not losses

`MarginAnnealing`, `ClassBalancedWeighting`, `HardNegativeMining`, `TemperatureScheduling` (`losses/utils.py`).

### Orchestrators

`LossIntegrationSystem` (`losses/integration.py`) wires **11** of the 12 criteria together — 4 classification + 3 representation (SupCon, Triplet, VICReg; BarlowTwins is imported but replaced by VICReg in the forward) + 4 consistency — under one of the four weighting methods. Its own `self.num_losses = 11` (consistency/representation terms default to zero unless augmented views are supplied). `LocationRecognitionLossModule` adds margin/temperature annealing, class balancing, and hard mining on top. Both are exercised by `tests/test_fixes.py` and `tests/test_model.py` but **not** by `train.py`.

---

## Config system — how the "knobs" are organized

Configuration is concentrated in **`transformer/config.py` → `AdaptiveTransformerConfig`**, a plain class (not a dataclass) with ~50 constructor args grouped into:

- **Core**: `dim`, `depth`, `num_heads`, `mlp_ratio`, `qkv_bias`, drop rates, `kernel_fn`, `use_axial_attention`, `use_adaptive_depth`.
- **Module 1 — adaptive attention**: `adaptive_heads`, `max_heads`, `head_z_init`.
- **Module 2 — MoE**: `use_moe`, `num_experts`, `moe_top_k`, `moe_jitter_noise`, `balance_loss_weight`, `capacity_factor`, `use_shared_expert`, `moe_layers`.
- **Module 3 — context norm**: `use_context_norm`, `norm_condition_scale/bias`, `norm_gated`, `norm_bottleneck_ratio`, `norm_layers`.
- **Module 4 — heterogeneous attention**: `use_hetero_attn`, `attention_types`, `gating_temperature`, `hetero_attn_layers`.
- **Pyramid / integration** (largely unused by `AdaptaNet`): `use_pyramid`, `pyramid_*`, `integration_mode`, `fusion_method`, `preserve_token_correspondence`.

The clever bit: each feature can be **scoped to specific layers**. `_process_layer_configs()` turns `moe_layers` / `norm_layers` / `hetero_attn_layers` into per-layer membership lists, and the encoder queries `is_moe_layer(i)` etc. per block. There is also a `layer_specific_params` override map (`set_layer_param` / `get_param`) and built-in validators (`_validate_config`, `_validate_pyramid_config`) that raise on inconsistent settings (e.g. `moe_top_k > num_experts`). The CNN backbone is configured separately through plain constructor kwargs on `DynamicCNNBackbone` (passed via `AdaptaNet(cnn_kwargs=...)`). There is **no** YAML/JSON config schema; the closest thing is the ad-hoc `variant_config.json` written by `ablate.py` and read by `train.py` via the `ADAPTANET_VARIANT_CONFIG` env var.

---

## Training & data pipeline

**Data setup** (`scripts/`):
1. `download_datasets.py` — CIFAR-100 via torchvision; CUB-200 via Caltech HTTP (~1.1 GB); SOP via gdown (~2.9 GB). Idempotent (skips existing).
2. `preprocess_datasets.py` — Resize 256 → CenterCrop 224 (CIFAR resizes 32→224 directly), ImageNet normalize, save `{"images":[N,3,224,224],"labels":[N]}` as `.pt` (SOP chunked at 10k/file).

**Training** (`scripts/train.py`):
- `TensorDataset` loads the cached `.pt` splits; `DataLoader` with pin-memory.
- Per-dataset `DATASET_CONFIGS`: CIFAR-100 (100 cls, 200 ep, bs 128, lr 1e-3, Mixup 0.8 / CutMix 1.0, CE), CUB-200 (200 cls, 100 ep, bs 32, lr 5e-4, CE, no mix), SOP (22,634 cls, 100 ep, bs 64, lr 5e-4, **ArcFace**). CLI flags override any of these plus `embed_dim`, `base_width`, `transformer_depth`, `num_heads`.
- Loop features: **AdamW** (wd 0.05), **cosine LR with linear warmup**, **AMP** (`GradScaler`/`autocast`), **grad clip** (max_norm 1.0), **gradient accumulation**, **EMA** (decay 0.9999, used for eval), **CutMix/Mixup** with soft-label cross-entropy, gradient-checkpointing toggles (guarded by `hasattr`).
- Objective: `arcface` path → `F.cross_entropy(ArcFaceLoss(emb,labels), labels)`; otherwise label-smoothed / soft-target cross-entropy on `logits`. **The multi-loss `LossIntegrationSystem` and MoE aux loss are not used.**
- Eval: Top-1/Top-5 accuracy (classification) or `compute_recall_at_k` (R@1/10/100 for SOP), both on EMA weights. Saves `best.pt` + periodic `epoch_N.pt`.

**Ablation** (`ablate.py`): defines 8 variants (`full`, `no_deformable`, `no_selective_kernel`, `no_moe`, `no_adaptive_heads`, `no_context_norm`, `no_hetero_attn`, `no_feature_fusion`), each launching `train.py` as a subprocess with the corresponding kwargs serialized to `variant_config.json`. **Benchmark** (`benchmark.py`) reports params/FLOPs(est)/memory/throughput; **visualize** (`visualize.py`) renders attention maps, MoE utilization, head importance, and t-SNE from a checkpoint.

The intended (but not implemented) training strategy in `plan.md` — two-view augmentation, curriculum loss weighting via `LossIntegrationSystem`, margin/temperature annealing, MoE aux loss — diverges substantially from the simpler shipped `train.py`.

---

## Tech stack & dependencies

- **Python**, **PyTorch ≥2.0** (core: `torch.nn`, `F.grid_sample`, `F.gumbel_softmax`, `torch.utils.checkpoint`, AMP).
- **torchvision** (datasets, transforms), **numpy**, **einops** (declared in requirements; not heavily used), **tqdm**, **requests**/**gdown**/**Pillow** (dataset download & preprocessing).
- Tests use **pytest**. No `setup.py`/`pyproject.toml`/packaging — it's a flat importable project (imports like `from model import AdaptaNet`, `from dynamic_cnn.backbone import ...` rely on running from the repo root / `sys.path` insertion in scripts).
- No GPU/CUDA custom kernels — deformable conv and log-polar are pure-PyTorch.

---

## Build / run / train (actual commands)

```bash
# Install
pip install -r requirements.txt

# Quick smoke test of the model in Python
python -c "import torch; from model import AdaptaNet; \
m=AdaptaNet(num_classes=200, embed_dim=256, transformer_depth=6, num_heads=8); \
print(m(torch.randn(2,3,224,224))['logits'].shape)"

# Data
python scripts/download_datasets.py --dataset cifar100 cub200
python scripts/preprocess_datasets.py --dataset cifar100

# Train
python scripts/train.py --dataset cifar100
python scripts/train.py --dataset cub200 --epochs 100 --batch-size 32
python scripts/train.py --dataset sop --loss arcface

# Ablation / benchmark / visualization
python scripts/ablate.py --dataset cifar100 --epochs 50
python scripts/benchmark.py
python scripts/visualize.py --checkpoint checkpoints/cifar100/best.pt --dataset cifar100 --output-dir figures/

# Tests
pytest tests/                 # or: python tests/test_fixes.py
```

---

## Status, completeness & notable gaps

- **Builds and runs.** Tests assert end-to-end forward/backward shapes, gradient flow (>30% of params receive gradients), and that adaptive components (gated projection, attentive pool) are input-dependent. The default config is asserted to exceed 1M params.
- **Loss library is decoupled from training.** The headline 20+/multi-loss system, MoE auxiliary loss, two-view augmentation, and curriculum weighting described in README/plan.md are implemented and unit-tested but **not invoked by `train.py`** — actual training is single-CE or single-ArcFace.
- **Significant dead code.** `transformer/pyramid.py` (1,211 LOC) and `transformer/integration.py` (2,867 LOC) — together ~30% of the entire codebase — implement a multi-scale pyramid transformer and a unified fusion transformer that the assembled `AdaptaNet` never uses (`use_pyramid=False`). Likewise `DynamicConv2d`, `DynamicExpertMixture`, `MultiScaleDynamicCNN`, axial attention, and token reduction are built but off the default path.
- **No published results.** `ROADMAP.md` has baseline/ablation tables with `?`/`TBD` placeholders; no `checkpoints/`, logs, or result JSONs are committed (and would be git-ignored).
- **Custom deformable conv.** Hand-written modulated deformable conv via `grid_sample` rather than `torchvision.ops.deform_conv2d` — functionally plausible but unoptimized and per-point looped in `_sample_features`.
- **No license file**, no packaging metadata, no CI config in the working tree.
- **Hygiene:** one uncommitted 1-line typo fix in `train.py`; `plan.md` untracked; `.gitignore` excludes `data/`, checkpoints, and the original `*.txt` source drafts the modules were extracted from.

---

## README vs. code

| README / docs claim | Code reality |
|---|---|
| "20+ purpose-built loss functions" | **12** actual loss criteria. Reaching "20+" requires counting the 4 weighting strategies + 4 utilities + 2 orchestrators as "losses." The integration system itself wires 11 together. |
| "general-purpose … classification, retrieval, metric learning" | Architecturally yes (emits logits + embeddings), but the shipped `train.py` only does classification CE / single-ArcFace; the metric-learning loss stack is never trained. |
| Architecture diagram shows CNN → CrossStageFeatureHierarchy → projection → Adaptive Transformer → pool → head | **Matches** `model.py` exactly (including the optional fusion bypass for ablation). |
| README structure lists pyramid (`pyramid.py`) and unified integration (`integration.py`) as transformer components | Present and large, but **not used** by `AdaptaNet` (pyramid off by default). |
| "Adaptive Transformer with head pruning, MoE FFN, multi-scale pyramid" | Head pruning ✅ (`AdaptiveMultiHeadAttention`), MoE ✅ (`MoEFFN`, layers [2,5]), pyramid ❌ in the assembled model. |
| `plan.md`: training uses `LossIntegrationSystem` (curriculum), two-view aug, margin/temp annealing, MoE aux loss | **Not implemented** in `train.py`; it's a forward-looking plan, not the current pipeline. |
| Default-config param estimate "~30M params" (plan.md) | Not contradicted by code, but unverified here; `test_default_config_builds` only asserts >1M. The benchmark script would compute the real number. |
| `requirements.txt` lists `einops` | Declared but not materially used in the modules read. |

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\AdaptaNet. Source of truth: https://github.com/AmeyaBorkar/AdaptaNet*
