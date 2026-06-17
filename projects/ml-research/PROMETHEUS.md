# PROMETHEUS

> A PyTorch document-image binarization study that ablates "fancy ideas" — physics-informed Neural ODEs, an FFT spectral branch, frozen DINOv2 features, ASPP, and a D-LinkNet decoder — against a plain segmentation baseline on the DIBCO benchmark. The headline, code-and-results-confirmed finding is that the physics-informed Neural ODE side-branch changes DIBCO-2017 PSNR by **−0.07 dB** (17.93 → 17.86) versus the identical no-ODE model: it does nothing. The single technique that actually mattered was dense native-resolution patch extraction (+2.72 dB). The repo also ships a separate, much larger "Document Genome" subsystem (DGI/NRE/AOSS/CAFR/WPIA/XDMN) and a FastAPI + Next.js web demo that are orthogonal to the binarization study.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/PROMETHEUS |
| **Visibility** | Public |
| **Category** | ml-research |
| **Primary language(s)** | Python (PyTorch) — 191 files, ~29.6k LOC; plus a TypeScript/Next.js frontend (~33 .ts/.tsx, ~3.2k LOC) and a FastAPI backend |
| **Local path** | `C:\Users\ameya\Documents\PROMETHEUS` |
| **Default branch** | `main` |
| **Lines of code (computed)** | Python: **29,635** lines / **191** files (excludes `.git`, `__pycache__`, `data/`, `checkpoints/`). Frontend src: ~3,261 lines TS/TSX + ~6.9k CSS. Docs/markdown: ~6.9k lines. (The 464k JS figure from a naïve scan is `frontend/node_modules` and is excluded.) |
| **Source files (computed)** | 191 Python files (core study: models 84, training 24, tests 27, data 18, eval 9, backend 19, inference 5, genome 6, scripts 12, root 3) |
| **Key dependencies** | `torch>=2.1`, `torchvision`, **`torchdiffeq>=0.2.3`** (Neural ODEs), DINOv2 via `torch.hub`, `scikit-image`, `opencv-python`, `albumentations`, `click`, `diffusers`/`transformers` (genome subsystem), `fastapi`/`uvicorn` (backend), `wandb`/`tensorboard` |
| **License** | **None** — no `LICENSE` file; README says "TBD" |
| **Last commit** | `8cdc732` — *docs(paper): IEEE conference draft, figures, and cover letter* — 2026-05-20 (94 commits total; first commit 2026-02-28). Working tree has uncommitted additions: `CONTRIBUTIONS.md`, several `docs/*.pptx`, `docs/build_deck.py`, `docs/paper/PRESENTATION_SCRIPT_AMEYA.md` |

---

## What it actually is

PROMETHEUS-Doc is an **ablation study disguised as a model**. The research question driving it: *does physics-informed degradation modeling (a Neural ODE that "reverses" image degradation) help document binarization on DIBCO?* The codebase systematically builds the fancy components, wires them as toggleable branches around a conventional segmentation backbone, trains with and without each, and evaluates on held-out DIBCO 2017. The repo's own paper draft answers the question bluntly: **"Answer: No."** (`docs/paper/PROJECT_JOURNEY.md`).

The genuinely binarization-focused, real (not stub) code is the **`PrometheusV3`** model: a ResNet34 encoder (output-stride 16) → DeepLabV3+-style **ASPP** bottleneck → optional **ODE / FFT / DINOv2** auxiliary branches fused by cross-attention → **U-Net decoder** → sigmoid binary head. An alternate path swaps in a reimplemented **DP-LinkNet** center+decoder. A predecessor `PrometheusUNet` put the ODE in the bottleneck (Phase 4, abandoned). The Neural-ODE engine itself is a real `torchdiffeq`-backed module (`models/dps/`, the "Differentiable Physics Simulator").

Surrounding this are two large bodies of code that are **not part of the binarization study**:
- A speculative **"Document Genome"** stack — `DGI` (ViT document inferrer), `NRE` (DiT diffusion re-renderer), `AOSS`, `CAFR`, `WPIA`, `XDMN` — trained in lettered stages A–G. Much of it is scaffolding for a different (document-understanding/re-rendering) pipeline.
- A **web demo**: FastAPI backend (`backend/`) + Next.js 16 / React 19 frontend (`frontend/`) for interactive restore/OCR/genome viewing.

> Naming caution: the README's title backronym is "Physics-Reversible Omni-Modal Encoding via Temporal Hierarchical Emergent Understanding of Symbolic Documents." In this repo **`DPS` = "Differentiable Physics Simulator"** (an invertible Neural ODE), *not* diffusion posterior sampling. The only true diffusion code is in the unrelated `nre/` genome subsystem.

## Architecture & how it's structured (data → model → train/eval)

**Data flow.** Real DIBCO degraded↔GT page pairs → `DensePatchDataset` extracts overlapping **256×256 patches at stride 128** (~5k native-res patches/epoch instead of 86 resized pages) → ImageNet-normalized input tensors, binary GT `(gray > 128)` → `PrometheusV3.forward` → BCE+Dice loss → checkpoint. Split is **leave-one-year-out**: train on DIBCO 2009–2016, validate/test on held-out **2017**.

**Model flow** (V3, unet mode): `image(B,3,256,256)` → ResNet34-OS16 (e1..e4) → ASPP(e4) → [optional ODE/FFT/DINO branches → cross-attention fusion] → U-Net decoder with concat skips → binary head → `{logits, binary}`.

**Train/eval.** Multi-script, CLI-flag-driven. Binarization training scripts live in `training/stage_p_*`; evaluation is full-resolution overlapping-tile inference (`eval/eval_fullres.py`) computing PSNR / SSIM / F-measure.

Annotated tree (study-relevant; `data/raw`, `data/cache`, `checkpoints`, `wandb`, `node_modules` excluded):

```
PROMETHEUS/
├── models/
│   ├── prometheus_v3/            # ★ MAIN binarization model
│   │   ├── model.py              #   PrometheusV3 (decoder_type: "unet" | "dlink")
│   │   ├── encoder.py            #   ResNet34Encoder (OS16: layer4 stride removed)
│   │   ├── aspp.py               #   ASPP — DeepLabV3+, dilations (6,12,18)
│   │   ├── decoder.py            #   UNetDecoderOS16 (concat skips)
│   │   ├── dlink_center.py       #   DPLinkNetCenter (HDC 1/2/4 + SPP 2/3/5)
│   │   ├── linknet_decoder.py    #   LinkNetDecoder (additive skips)
│   │   ├── ode_side_branch.py    #   ★ ODESideBranch — physics ODE, PARALLEL to ASPP
│   │   ├── frequency_branch.py   #   FrequencyBranch — FFT log-mag + phase CNN
│   │   ├── dino_branch.py        #   DINOBranch — frozen DINOv2 ViT-B/14 (torch.hub)
│   │   ├── cross_attention.py    #   CrossAttentionFusion (8-head MHA)
│   │   └── boundary_refinement.py#   BARRN + Sobel + boundary_weighted_bce (Stage-2)
│   ├── prometheus_unet/          # Phase-4 predecessor: ODE-in-bottleneck U-Net
│   │   └── ode_bottleneck.py     #   ODEBottleneck (512→64→ODE→512, critical path)
│   ├── dps/                      # ★ Neural-ODE physics engine ("Diff. Physics Simulator")
│   │   ├── model.py              #   DPS: param-encoder + invertible ODE (restore/degrade)
│   │   ├── modules/ode_func.py   #   ★ DegradationODEFunc: dx/dt=f(x,t,params), FiLM ResBlocks, mass-action
│   │   ├── modules/param_encoder.py
│   │   └── flows/invertible_degrade.py  # torchdiffeq integrator (Euler direct / adjoint), DDE adaptive
│   ├── dplinknet/dplinknet.py    # Vendored DPLinkNet34 (loads published pretrained .th)
│   ├── binarization/             # BinarizationHead (turns DPS RGB output → prob map)
│   ├── common/losses/            # dice, soft_fmeasure, BinarizationLoss, restoration losses
│   └── dgi|nre|aoss|cafr|wpia|xdmn/  # "Document Genome" subsystem (orthogonal to binarization)
├── training/
│   ├── trainer.py                # Generic Trainer (AMP-capable but unused, grad-clip, callbacks)
│   ├── stage_p_v3.py             # ★ main V3 training (dense patches, BCE+Dice)
│   ├── stage_p_v3_monitored.py   # self-contained V3 loop w/ heavy per-epoch metrics
│   ├── stage_p_v3_finetune_dplinknet.py # fine-tune pretrained DP-LinkNet + Freq/DINO
│   ├── stage_p_v3_refine.py      # two-stage DP-LinkNet + BARRN
│   ├── stage_p_unet.py           # Phase-4 ODE-bottleneck training
│   └── stage_a..g_*.py           # genome-subsystem curriculum (B/C/D/E/F/G)
├── eval/
│   ├── eval_fullres.py           # ★ main DIBCO evaluator (tiled, TTA, 10 ckpt loaders)
│   ├── eval_ensemble.py          # DP-LinkNet + V3 probability ensemble
│   ├── run_ablations.py          # checkpoint/stage ablation matrix
│   ├── metrics.py                # PSNR, SSIM, F-measure, DRD (simplified)
│   └── results_*/fullres_results.json  # ★ stored ablation numbers
├── data/
│   ├── loaders/patch_dataset.py  # ★ DensePatchDataset (256/stride-128)
│   ├── loaders/dataset_loaders.py# DIBCO/FUNSD/CORD/IAM/SD7K loaders + registry
│   ├── loaders/combined_patch_dataset.py # DIBCO+SD7K combined
│   └── degradation/physics_models/degradation_engine.py # classical 9-type degradation
├── configs/{training,evaluation,inference}/*.yaml  # genome-system configs (NOT V3 hyperparams)
├── backend/ (FastAPI)  frontend/ (Next.js 16 + React 19)  # web demo
├── run_overnight.py / run_finetune_experiments.py        # ★ experiment runners
├── docs/paper/{PROJECT_JOURNEY,TRAINING_LOG,METRICS,...}.md, abstract.tex  # findings
├── README.md  ROADMAP.md  CONTRIBUTIONS.md
└── requirements.txt  pyproject.toml  setup_project.py (1.6k-line scaffolder)
```

## Model architecture (in code) — the network(s), blocks, with shapes/flow

**`PrometheusV3`** (`models/prometheus_v3/model.py`). Two structural variants via `decoder_type`:

- **`"unet"` (default):** ResNet34-OS16 → ASPP(256ch) → optional aux branches → U-Net decoder → head.
- **`"dlink"`:** ResNet34-OS32 → `DPLinkNetCenter` (HDC+SPP) → `LinkNetDecoder` → head. *In dlink mode the aux branches are force-disabled* (`self.use_ode = use_ode and decoder_type != "dlink"`, same for freq/dino).

Default forward (unet mode), with channel/shape annotations taken from the code/README:

```
image (B,3,H,W)  ImageNet-normalized
 → ResNet34Encoder (OS16):  e1(64,H/4)  e2(128,H/8)  e3(256,H/16)  e4(512,H/16)
 → ASPP(e4)                 → (B,256,H/16,W/16)
 [opt] ODESideBranch(e4)    → (B,256,H/16);  bottleneck = ode_fusion(cat→512→256)
 [opt] FrequencyBranch(img) → (B,256,H/16),  DINOBranch(img) → (B,256,H/16)
       bottleneck = CrossAttentionFusion(spatial, aux_list)   # (B,256,H/16)
 → UNetDecoderOS16: cat(bottleneck,e3)@H/16→256 ; up+cat(e2)@H/8→128 ; up+cat(e1)@H/4→64 ; up→(B,64,H,W)
 → binary_head: Conv(64→32,3)+BN+ReLU+Conv(32→1,1) → sigmoid
 ⇒ {"logits":(B,1,H,W), "binary":(B,1,H,W)}
```

### ODE block (physics-informed) — REAL
Two wrappers, both reusing the `models/dps/` engine:

- **V3 (current): `ODESideBranch`** (`models/prometheus_v3/ode_side_branch.py`). Runs **parallel to ASPP, not in the critical path** — by deliberate design so the 1×1 fusion conv "can learn to ignore the ODE contribution" if it's useless (per the module docstring). Flow: `e4(512)` → `param_extractor` (GAP→Linear→GELU→Linear → 128-d params) and `proj_in`(512→128ch) → `InvertibleDegradation.restore(h, params)` (4 Euler steps) → `proj_out`→256ch → concatenated with ASPP, fused by 1×1 conv.
- **v1/v2 (predecessor): `ODEBottleneck`** (`models/prometheus_unet/ode_bottleneck.py`) — sat **in** the critical path (512→64→ODE→512 + residual). Abandoned in Phase 4 ("512→64ch compression kills capacity").

The ODE **dynamics** live in `DegradationODEFunc` (`models/dps/modules/ode_func.py`), modeling `dx/dt = f(x, t, params)` with FiLM-conditioned `ResBlock`s (the "enhanced" architecture V3 uses) plus a time embedding. The **physics-informed / mass-action** gating (the actual "physics" claim) gates velocity by degradation severity:

```python
# Mass-action gating: velocity proportional to degradation severity.
if self._params is not None and self.mass_action:
    param_norm = torch.norm(self._params, dim=1, keepdim=True)   # (B,1)
    gate = param_norm / (param_norm + self._mass_action_c)       # (B,1)
    velocity = velocity * gate.unsqueeze(-1).unsqueeze(-1)        # (B,C,H,W)
```

So clean inputs (‖params‖≈0 → gate≈0) yield ~identity; degraded inputs (gate≈1) get full restoration. This is **mass-action velocity gating, not a literal reaction-diffusion PDE.** Integration is in `InvertibleDegradation` (`models/dps/flows/invertible_degrade.py`), which imports `from torchdiffeq import odeint_adjoint` and provides: a default **fixed-step Euler "direct" integrator with gradient checkpointing** (`x = x + dt*dx`), an **adjoint** path (off by default; documented as having ~34% gradient error at 4 steps), and an adaptive **"DDE" (degradation-depth)** path with per-image step size `dt = -depth/num_steps`. In V3/UNet the ODE operates in **feature space**, not on RGB.

### ASPP — REAL
`ASPP` (`models/prometheus_v3/aspp.py`). DeepLabV3+ style, 5 branches: 1×1 conv; three dilated 3×3 at **rates (6, 12, 18)** (configurable via `aspp_rates`); and global-average-pool branch. `in=512, out=256`; project conv reduces `256*5→256` with Dropout(0.5). There is **no explicit "no-ASPP" ablation flag — ASPP is the V3 baseline backbone.**

### DINOv2 backbone — REAL, frozen, torch.hub
`DINOBranch` (`models/prometheus_v3/dino_branch.py`). Loaded via **`torch.hub.load("facebookresearch/dinov2", "dinov2_vitb14", pretrained=True)`** (not timm). **Always frozen** (`eval()`, `requires_grad=False`, `train()` overridden to keep DINO in eval). `embed_dim=768`, `patch_size=14`. Features pulled with `get_intermediate_layers(x, n=1, reshape=True)` under `no_grad`, input reflect-padded to a multiple of 14, output interpolated to H/16 and projected 768→256 by a trainable 1×1 conv. Only the projection trains (README's "86.6M frozen params add nothing measurable").

### Decoder / segmentation head
- **U-Net path:** `UNetDecoderOS16` — concat skips; because bottleneck and e3 are both at H/16, level-3 concatenates without upsampling. ASPP+U-Net combo ≈ DeepLabV3+.
- **D-LinkNet path:** `DPLinkNetCenter` (HDC dilations 1/2/4, SPP pools 2/3/5) + `LinkNetDecoder` (additive skips, bottleneck decoder blocks).
- Shared head: `Conv(64→32,3)→BN→ReLU→Conv(32→1,1)→sigmoid`.
- `DPLinkNet34` (`models/dplinknet/`) is a separate **vendored** reimplementation whose layer names match a published pretrained checkpoint (`dibco_dplinknet34.th`) so it loads cleanly.

### Other "fancy ideas" — all REAL (not stubs)
- **FrequencyBranch** — `torch.fft.fft2` + `fftshift` → 2-channel `[log-magnitude, phase]` grayscale spectrum → 4-layer stride-2 CNN → 256ch@H/16.
- **Multi-view fusion** — `CrossAttentionFusion`: spatial features = queries; aux branches concatenated along sequence dim = keys/values; `nn.MultiheadAttention` (8 heads) + residual + FFN(256→1024→256). Each branch independently toggleable; "no aux" ⇒ pure ASPP baseline.
- **BARRN** (`boundary_refinement.py`) — ~481K-param Stage-2 residual refiner: takes RGB + Stage-1 prob map (4ch), fixed Sobel edge detection + boundary attention, predicts a **logit delta** added to the Stage-1 logit. Paired loss `boundary_weighted_bce` weights boundary pixels 5×.
- **No cellular automata.** "Genome" refers to the separate DGI/NRE/… document-understanding subsystem, not CA.

## Ablations & experiments — what variants the code defines and what each tests

Two mechanisms.

**(A) Architecture-branch ablation (the main DIBCO ablations)** — CLI flags on `PrometheusV3`/training scripts:
- `--use-ode` → ODE side-branch on/off (own `--ode-lr` group). **Tests: does physics help?**
- `--use-freq` → FFT spectral branch. **Tests: does frequency-domain analysis help?**
- `--use-dino` → frozen DINOv2 branch. **Tests: do foundation-model features help?**
- `--decoder-type {unet, dlink}` → ASPP+U-Net vs D-LinkNet (forces OS32 for dlink).
- `--output-stride {8,16,32}`, `--freeze-encoder N`, `--unfreeze-all` → recipe ablations.

The clean comparison (same backbone, same data) is produced by `run_overnight.py`: train V3 *without* ODE (runs 3–4) vs V3 *with* ODE (runs 5–8). `run_finetune_experiments.py` does the equivalent on a pretrained DP-LinkNet: base fine-tune vs `+freq`, `+dino`, `+freq+dino`, and an ensemble.

Inside `eval/eval_fullres.py` two more ablations are baked into checkpoint loaders: `stage_d_raw` = `BinarizationHead` **without DPS** (tests whether the ODE restoration contributes vs the head alone) and `two_stage` = DP-LinkNet + BARRN. TTA folds ablated via `--tta {1,4,8}`; ensemble blend weights ablated in `eval_ensemble.py` (50/50…80/20).

**(B) Checkpoint/stage ablation matrix** — `eval/run_ablations.py` `ABLATIONS` list across genome-subsystem stages (`stage_b_dps`, `stage_c_dps`, `stage_e_dps`, `stage_e_dps_nre`, `stage_f_cafr`) × datasets (sd7k / dibco / dibco2017). This isolates the DPS / NRE / CAFR contributions and ODE integration granularity (`--ode-steps`).

**What the ablation actually measures:** all comparisons are on **held-out DIBCO 2017 (20 images)** in **PSNR / SSIM / F-measure**. There is no "with/without ASPP" axis (ASPP is the baseline), and DRD / pseudo-F are never computed in any ablation (see Datasets & Results).

## Loss functions & training loop

**Losses** (`models/common/losses/` + inline in training scripts):
- **Binarization:** `BinarizationLoss = λ_bce·BCE + λ_dice·Dice + λ_fm·(1−softF1)` (defaults 1.0 / 1.0 / 0.5). The plain V3 wrapper uses `1.0·BCE + 1.0·Dice`. `soft_fmeasure_loss` follows DIBCO convention (foreground = dark, value 0): computes soft TP/FP/FN on the inverted maps and returns `1 − F1`.
- **Boundary-weighted BCE** (`boundary_weighted_bce`) — dilates the FG mask, weights boundary pixels 5× (BARRN Stage-2).
- **Restoration (genome/DPS stages):** `CombinedRestorationLoss = λ_mse·MSE + λ_perceptual·VGG + λ_ssim·(1−MS-SSIM)` (defaults 1.0 / 0.1 / 0.1).
- **Genome-system losses** (`genome_losses.py`: fidelity, physics-cycle-consistency, coherence, entropy-belief, anti-hallucination, identity) are defined and used in stages A–G but **not wired into the DIBCO binarization stages**.

**Training loop.** Core is `training/trainer.py` (`Trainer.fit → _train_epoch → _train_step`), though the monitored V3 script replaces it with a self-contained loop. Confirmed defaults:
- **Optimizer:** `AdamW` everywhere (`weight_decay` 1e-4 for Stage-P binarization, 0.01 for restoration stages), with **differential per-group LRs** (separate main / ODE / encoder groups).
- **Scheduler:** linear warmup → cosine (`SequentialLR([LinearLR, CosineAnnealingLR])`), stepped per epoch (~5-epoch warmup) for Stage-P.
- **Epochs/batch (V3):** 200 epochs / batch 8 (batch 4 when DINO is on).
- **AMP:** `Trainer` supports it but **defaults off and no stage enables it** — effectively never used.
- **Gradient accumulation:** none. **EMA:** none. **Grad clip:** norm 1.0 (0.5 when the ODE flow is unfrozen in Stage C).
- **Checkpoints:** `ModelCheckpoint` saves best-only (`best.pt`) with model/optimizer/scheduler state + a config JSON alongside for eval-time reconstruction.

The genome subsystem adds an explicit **lettered curriculum** (Stage A data-gen → B per-module pretrain → C physics calibration → D joint → E self-supervised AOSS → F CAFR routing → G XDMN memory), each loading the previous stage's checkpoint. This is what the `checkpoints/stage_a..g`, `dps`, `cafr` directories correspond to.

## Datasets & evaluation (DIBCO handling, metrics)

**DIBCO handling.** Two loaders: `DIBCODataset` (whole-page, resized — minor) and the workhorse **`DensePatchDataset`** (`data/loaders/patch_dataset.py`). The latter loads full-res pages into RAM and extracts **overlapping 256×256 patches at stride 128 (50% overlap)** — not 288. GT binarized as `(gray > 128)` → 1.0 background / 0.0 text; input ImageNet-normalized; sub-patch images reflect-padded. **Split is by DIBCO year (leave-one-out): train `exclude_years=[2017]`, validate `only_years=[2017]`** — deterministic, no random split, seed 42. On disk: DIBCO **2009, 2010, 2011, 2012, 2013, 2014, 2016, 2017** (no 2015/2018/2019). Other datasets have loaders (`SD7K` actively combined with DIBCO via `CombinedPatchDataset`, using Otsu on the clean image as binary GT; FUNSD/CORD/IAM exist but return `clean=None` and aren't used for the supervised binarization metric; CVL referenced in config only).

**Degradation engine** (`data/degradation/physics_models/degradation_engine.py`): a **classical, non-differentiable** 9-type simulator (ink diffusion, photochemical fading, cellulose oxidation, water damage, deformation, bleed-through, JPEG artifacts, biological degradation, scanning artifacts). It is *forward synthesis* for synthetic data / online augmentation — **the real DIBCO binarization training does not use it** (it trains on real DIBCO pairs with simple flip/rot90/photometric augmentation). This is distinct from the learned ODE inverse model.

**Metrics** (`eval/metrics.py`, all self-implemented in NumPy — no external DIBCO tool actually called):
- **F-measure** — TP/FP/FN with foreground=0, background=255; `F = 2PR/(P+R)`.
- **PSNR** — `10·log10(255² / MSE)`.
- **SSIM** — wraps `skimage.metrics.structural_similarity` (reported alongside; not a DIBCO metric).
- **DRD** — `compute_drd` exists but is a **simplified/approximate** version (5×5 reciprocal-distance weighting, the code itself flags it "Simplified"), and crucially **is not called by any of the DIBCO-2017 eval scripts** (`eval_fullres.py`, `run_ablations.py`, `eval_ensemble.py` compute only PSNR/SSIM/F-measure).
- **pseudo-F-measure (pFM/Fps)** — **listed in config but never implemented** anywhere in `eval/`.

So despite the README/config naming the full DIBCO metric suite, **only PSNR, SSIM, and F-measure are actually computed in the study.**

## Results — what's stored in the repo

Stored as `eval/results_*/fullres_results.json` (held-out DIBCO 2017, 20 images, tile 256/overlap 64, learned binarization, TTA=1) and tabulated in `docs/paper/*.md`. Key numbers:

**The headline ODE ablation (same backbone, same data):**

| Variant | F-measure | PSNR (dB) | SSIM | Source |
|---|---|---|---|---|
| **V3 baseline (no ODE)** | **0.91156** | **17.934** | 0.94025 | `eval/results_prometheus_v3/` |
| **V3 + ODE side-branch** | **0.91019** | **17.855** | 0.94023 | `eval/results_v3_ode/` |
| V3 fine-tuned (no ODE, best) | 0.91443 | 18.062 | 0.94247 | `eval/results_v3_finetune/` |
| V3 + ODE fine-tuned | 0.90949 | 17.846 | 0.93989 | `eval/results_v3_ode_finetune/` |

→ Adding the ODE side-branch = **−0.079 dB PSNR / −0.00138 F / SSIM unchanged** (the docs round to "−0.07 dB"). The ODE variant is marginally **worse on every metric.**

**DP-LinkNet branch ablation:**

| Variant | F-measure | PSNR (dB) | Source |
|---|---|---|---|
| DP-LinkNet pretrained | 0.93856 | 19.618 | `results_dplinknet_pretrained/` |
| **DP-LinkNet ft base (best)** | **0.94206** | **19.740** | `results_dplinknet_ft_base/` |
| + DINOv2 | 0.94088 | 19.617 | `results_dplinknet_ft_dino/` |
| + Frequency (FFT) | 0.93963 | 19.645 | `results_dplinknet_ft_freq/` |
| + Freq + DINOv2 | 0.94026 | 19.574 | `results_dplinknet_ft_multiview/` |

→ Every added branch slightly **hurt** the DP-LinkNet base fine-tune.

**Stored conclusions (verbatim):**
- `docs/paper/PROJECT_JOURNEY.md`: *"Does physics-informed degradation modeling help document binarization? Answer: No. When the segmentation backbone has sufficient capacity and proper training data, physics features are redundant."*
- `docs/paper/TRAINING_LOG.md`: *"ODE side-branch provides negligible benefit. V3+ODE (17.86) vs V3 (17.93): −0.07 dB."*
- `docs/paper/METRICS.md`: *"no novel feature extraction approach provides meaningful improvement over the base segmentation recipe. The dominant factors are (1) dense native-resolution patches and (2) architecture + training recipe maturity."* (Dense patches = **+2.72 dB**, the single biggest factor.)
- `abstract.tex` frames it as "honest ablation … including negative results where auxiliary modules do not improve pixel-level restoration." The one place physics helped large was **Stage-C physics calibration of the standalone DPS (+6.6 dB)** — but that gain becomes redundant once a proper segmentation backbone + dense patches are used.

**Best overall:** fine-tuned DP-LinkNet **19.74 dB**; from-scratch V3 **18.06 dB**; SOTA gap to published DP-LinkNet (20.83 dB, which includes TTA) ≈ 1.56 dB without TTA.

**Log-file notes:** `finetune_results.txt` / `overnight_results.txt` are run-status logs (step + OK/duration), no metrics. `overnight_log.txt` is a *failed* UTF-16 run (all steps exit 127, bad Python path); the real numbers came from a later successful re-run.

## Tech stack & dependencies

- **Core:** Python ≥3.10, **PyTorch ≥2.1 (CUDA)**, torchvision (ResNet34 ImageNet-pretrained encoder).
- **Neural ODEs:** **`torchdiffeq>=0.2.3`** (Euler direct integration + adjoint option).
- **Foundation model:** DINOv2 ViT-B/14 via `torch.hub` (frozen).
- **Imaging/metrics:** `scikit-image` (SSIM, Otsu), `opencv-python`, `albumentations`, `pillow`, NumPy.
- **Genome subsystem (orthogonal):** `diffusers`, `transformers`, `torch-geometric`, OCR (`pytesseract`, `paddleocr`), `lpips`, `sentence-transformers`, `jiwer`.
- **Config/logging:** `click` (CLI), `omegaconf`/`hydra-core` (genome configs), `wandb`, `tensorboard`.
- **Web demo:** `fastapi` + `uvicorn` backend; **Next.js 16 / React 19** frontend with shadcn/ui, Tailwind v4, `lucide-react`, `sonner`.
- **Package:** `prometheus-doc` v0.1.0 (`pyproject.toml`); pytest testpaths `tests`; ruff line-length 100.

## Build / run / train / eval (actual commands)

```bash
# Install (per README)
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu128
pip install torchdiffeq tqdm click pillow numpy scikit-image doxapy

# Train PrometheusV3 (ASPP baseline, from scratch)
python training/stage_p_v3.py --epochs 200 --batch-size 8 --save-dir checkpoints/prometheus_v3

# Train with the ODE side-branch (the physics ablation)
python training/stage_p_v3.py --use-ode --ode-lr 1e-4 --epochs 200 --save-dir checkpoints/prometheus_v3_ode

# D-LinkNet decoder variant
python training/stage_p_v3.py --decoder-type dlink --epochs 200 --batch-size 8 --save-dir checkpoints/prometheus_v3_dlink

# Fine-tune pretrained DP-LinkNet + novel branches
python training/stage_p_v3_finetune_dplinknet.py --use-freq --use-dino --epochs 100 --save-dir checkpoints/dplinknet_ft_multiview

# Two-stage boundary refinement (needs pretrained DP-LinkNet weights)
python training/stage_p_v3_refine.py --epochs 80 --batch-size 8 --save-dir checkpoints/two_stage

# Evaluate (full-res tiled) on held-out DIBCO 2017
python eval/eval_fullres.py --ckpt checkpoints/prometheus_v3/best.pt \
  --ckpt-format prometheus_v3 --binarize learned --show-baseline --out-dir eval/results_v3

# Reproduce the headline suites
python run_overnight.py             # V3 vs V3+ODE (the physics ablation)
python run_finetune_experiments.py  # DP-LinkNet base vs +freq/+dino/+multiview/ensemble

# Web demo
uvicorn backend.main:app --reload   # FastAPI backend
cd frontend && npm run dev          # Next.js frontend
```

## Status, completeness & notable gaps

- **Study is complete and the central question is answered** with real stored numbers; the codebase honestly documents its own negative results.
- **Phase 7 (two-stage BARRN refinement) is "TBD"/in progress** — the README's results table leaves it blank.
- **DRD is a simplified approximation and never invoked** by the headline eval scripts; **pseudo-F-measure is configured but unimplemented.** Any claim of full DIBCO-protocol metrics would be overstated — only PSNR/SSIM/F-measure are produced.
- **AMP and EMA are advertised by the architecture but absent in practice** (AMP exists in `Trainer`, never enabled; EMA nowhere).
- **No license** — README says "TBD"; absence of a LICENSE file means default all-rights-reserved despite the public, "open" framing.
- A large fraction of the 191 Python files is the **genome subsystem** and **tests (27 files, ~5k LOC)** — much of the genome stack reads as experimental scaffolding orthogonal to the binarization study.
- Minor sloppiness: a stray typo alias `cosine = CosineAnnualingLR = CosineAnnealingLR(...)` in `stage_p_v3_combined.py` (harmless).
- The frontend ships `node_modules` in the working tree (inflates naïve LOC counts).

## README vs. code

The README is **unusually honest** — it documents the negative findings rather than hiding them, so most discrepancies are factual details, not spin:

| Claim in README | Reality in code/results |
|---|---|
| "ODE provides no benefit (−0.07 dB)" | **Confirmed.** Stored JSONs give −0.079 dB / −0.00138 F. ✅ |
| "Dense patches +2.72 dB, the dominant factor" | **Confirmed** by `METRICS.md`/results. ✅ |
| Tech Stack: "Evaluation: doxapy (official DIBCO metrics)" | **Misleading.** Metrics are **self-implemented in NumPy** (`eval/metrics.py`); no doxapy/official-tool call in the eval scripts. DRD is a simplified approximation and isn't called; pFM is unimplemented. |
| Quick Start installs `doxapy` | Not imported anywhere in the eval path. |
| "PrometheusV3 … 27.9M params"; "D-LinkNet 28.78M"; "BARRN 481K / 86.6M frozen DINO" | Param counts are plausible but **not asserted in code** (no count printed in the model files read); treat as README-reported. |
| Results table shows "F-measure 0.928 / 0.939 / 0.942 …" with DRD implied by the DIBCO framing | **DRD and pFM never appear in any stored result.** All numbers are PSNR/SSIM/F-measure only. |
| Tech Stack: "PyTorch 2.10 (CUDA 12.8)", "RTX 5070 Ti Laptop" | Hardware/version are environment notes; `requirements.txt` only pins `torch>=2.1`. |
| "DPS … physics simulator" / project title "Physics-Reversible" | Accurate, but easy to confuse with diffusion-posterior-sampling — here **DPS = Differentiable Physics Simulator** (an invertible Neural ODE). |
| License section | README "TBD"; **no LICENSE file** exists. |
| Patch size implied generic | Code is specifically **256×256 / stride 128** (cache dirs also show 288 variants, but the active loader uses 256). |
| README omits the genome subsystem and web demo | Both are large, real parts of the repo (`models/dgi|nre|aoss|cafr|wpia|xdmn`, `backend/`, `frontend/`) — orthogonal to the binarization study but a big share of the codebase. |

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\PROMETHEUS. Source of truth: https://github.com/AmeyaBorkar/PROMETHEUS*
