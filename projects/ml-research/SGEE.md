# SGEE — Semantically-Grounded Emotion Embeddings

> A PyTorch / HuggingFace research codebase that fine-tunes a RoBERTa-base backbone into a 512-d "SGEE" embedding by fusing the transformer `[CLS]` vector with NRC-VAD lexicon statistics and surface (persuasion) features, then anchoring that embedding in valence-arousal-dominance space via multi-task heads and a prototype-based InfoNCE contrastive loss. Training is multi-stage: **Stage 1** = supervised multi-label emotion classification (+ VAD regression) on GoEmotions; **Stage 2** = VAD grounding with prototype contrastive alignment; **Stage 3** (present in code, beyond the README's stated scope) = domain adaptation to persuasion/clickbait datasets with knowledge-distillation anti-forgetting. A real caveat surfaced by reading the code: the VAD regression targets for GoEmotions are **lexicon-derived** (mean NRC-VAD of in-text tokens), not gold human VAD annotations.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/SGEE-Semantically-Grounded-Emotion-Embeddings |
| **Visibility** | Public |
| **Category** | ml-research |
| **Primary language(s)** | Python (100% of source); plus one HTML demo page |
| **Local path** | `C:\Users\ameya\Documents\sgee_aca_project` |
| **Default branch** | `main` |
| **Lines of code (computed)** | 24,390 lines of Python across 60 `.py` files (excludes `.git`, `__pycache__`, vendored `data/raw/WANDS`); + 594 lines in `src/index.html` |
| **Source files (computed)** | 60 Python files; 1 YAML config (`configs/base_config.yaml`, effectively empty — a single comment line); 1 HTML |
| **Key dependencies** | `torch>=2.0`, `transformers>=4.30` (RoBERTa), `scikit-learn` (KMeans), `numpy`, `pandas`, `tqdm`, `pyyaml`; **also imported but NOT in requirements.txt:** `datasets` (HuggingFace), `scipy`, `matplotlib`, `seaborn`, `flask`, `flask_cors` |
| **License** | None — no `LICENSE` file in repo; README states "for academic and research purposes" only |
| **Last commit** | `13d2752` — *"Initial commit: SGEE Stage 1 & 2 - Semantically Grounded Emotion Embeddings"*, 2026-02-27 (single-commit history) |

---

## What it actually is

SGEE is an academic ("ACA project") research codebase that learns an emotion-aware text representation by **grounding a transformer in psycholinguistic VAD space**. The pipeline:

1. A **RoBERTa-base** backbone (`AutoModel.from_pretrained("roberta-base")`) produces a pooled `[CLS]` embedding (768-d).
2. Two hand-engineered feature vectors are computed per text: a **17-d NRC-VAD lexicon feature vector** (mean/std/max/min/weighted-mean valence-arousal-dominance over matched lexicon tokens, plus token count + coverage ratio) and a **16-d surface feature vector** (caps ratio, punctuation counts, emoji/currency/number counts, and persuasion keyword counts: CTA / scarcity / urgency / social-proof).
3. These are encoded (small MLPs → 32-d and 16-d) and **concatenated** with the 768-d backbone vector (768 + 32 + 16 = 816-d), then projected through a GELU MLP with a residual connection and LayerNorm into the **512-d SGEE embedding**.
4. Multi-task heads read the SGEE embedding: an **EmotionHead** (28-class multi-label, sigmoid/BCE) and a **VADHead** (3-d regression, `tanh`-bounded to [-1, 1]).
5. In Stage 2+, **VAD prototypes** (KMeans cluster centers over the NRC-VAD lexicon) plus **learnable prototype embeddings** drive an InfoNCE contrastive loss that pulls each SGEE embedding toward the prototype matching its VAD target.

The actual repository is substantially larger than the README implies: it contains a full Stage 3 (persuasion / domain adaptation), a DistilBERT teacher for clickbait pseudo-labels, a Flask inference server with a web UI, and ~10 ablation baselines (A-H plus plain/multitask RoBERTa) with logged metrics.

---

## Architecture & how it's structured (annotated tree)

```
sgee_aca_project/
├── src/
│   ├── config.py                      # SGEEConfig + dataclasses (ModelConfig, TrainingConfig, ...). NOTE: file
│   │                                  #   header says "src/utils/config.py" but it lives at src/config.py
│   ├── models/
│   │   ├── backbone.py                # SGEEBackbone: AutoModel(roberta-base) wrapper; cls/mean/max pooling,
│   │   │                              #   gradient checkpointing, attention extraction
│   │   ├── sgee_projection.py         # SGEEProjection: lexicon(17→32) + surface(16→16) encoders,
│   │   │                              #   concat→816→[768,512] GELU MLP + residual → 512-d SGEE embedding
│   │   ├── heads.py                   # EmotionHead (28-class BCE), VADHead (3-d tanh regression),
│   │   │                              #   CombinedEmotionVADHead
│   │   ├── prototypes.py              # VADPrototypeManager (KMeans over NRC-VAD), LearnedPrototypeEmbeddings
│   │   ├── persuasion_heads.py        # Stage 3: CTA/scarcity/urgency/social-proof/clickbait/actionability heads
│   │   │                              #   + 64-d persuasion "fingerprint"
│   │   └── full_model.py              # SGEEModel: backbone → projection → emotion + VAD heads
│   ├── data/
│   │   ├── datasetsProj.py            # GoEmotionsDataset (loads .pt cache or HF), create_dataloaders;
│   │   │                              #   VAD targets = lexicon vad_mean (see Datasets section)
│   │   ├── features.py                # LexiconFeatureExtractor (17-d), SurfaceFeatureExtractor (16-d)
│   │   ├── preprocessing.py           # text cleaning pipeline
│   │   └── stage3_unified_dataset.py  # unified multi-dataset loader for Stage 3
│   ├── training/
│   │   ├── trainStage1.py             # Stage 1 entrypoint (interactive: quick/dev/full modes)
│   │   ├── trainStage2.py             # Stage 2 entrypoint (loads Stage 1 ckpt; prototype contrastive)
│   │   ├── train_stage1_enhanced.py   # larger Stage-1 trainer variant (1,269 LOC)
│   │   ├── trainer.py                 # SGEETrainer: metrics, checkpointing (1,149 LOC)
│   │   ├── losses.py                  # Stage 1: EmotionLoss(BCE), VADLoss(Huber), ContrastiveLoss(InfoNCE),
│   │   │                              #   MultiTaskLoss (per-stage weight schedules)
│   │   ├── stage2_losses.py           # Stage2MultiTaskLoss: AdaptiveEmotionLoss (pos_weight + effective-number),
│   │   │                              #   WeightedVADLoss, ContrastiveVADLoss + equipartition + soft-assign
│   │   ├── stage2_trainer.py          # Stage 2 trainer w/ prototype management
│   │   ├── stage3_losses.py / stage3_trainer.py / Stage3TrainingScript2.py   # Stage 3 (persuasion + distill)
│   │   ├── train_baseline_{E,F,G,H}.py            # ablations
│   │   ├── train_baseline_roberta_{plain,multitask}.py   # RoBERTa-only ablations
│   │   ├── run_multi_seed.py / run_sgee_seeds.py # multi-seed runners (seeds 42/43/44)
│   │   └── evaluation.py              # STUB — single comment line "# Metrics & validation"
│   ├── sgee_stage2_inference.py / sgee_stage3_inference.py   # inference wrappers
│   ├── server.py                      # Flask + flask_cors server (/, /analyze, /batch, /model-info, /health)
│   └── index.html                     # 594-line single-page web UI for the Flask demo
├── models/
│   ├── Baselines/                     # baseline_a..g outputs + AllBaseLineMetrics.txt + logs/weights
│   ├── Stage1RemasteredRun/           # re-run Stage 1 training + optimization scripts
│   └── Stage2Reclustered/             # Stage 2 with re-clustered prototypes
├── data/
│   ├── lexicons/NRC-VAD-Lexicon-v2.1/ # NRC-VAD lexicon (unigrams, MWE, per-dimension files)
│   ├── raw/                           # AFINN, VADER, WANDS, Upworthy archive, Webis clickbait-17/22, etc.
│   ├── processed/, preprocessed_cache/, Stage3UnifiedDataset/   # cached .pt tensors + teacher distilbert
│   ├── teacherModelWebis.py           # DistilBERT teacher for clickbait pseudo-labels
│   └── *_preprocessing.py             # per-dataset preprocessing (upworthy/wands/webis/goemotions/stage3)
├── configs/base_config.yaml           # essentially empty (one comment line)
├── testRuns.py                        # Stage 1 → Stage 2 weight-compatibility checker
├── requirements.txt                   # 7 lines (incomplete — see dependency note)
└── README.md
```

**Forward pipeline (`SGEEModel.forward`, `full_model.py`):**
`input_ids, attention_mask, lexicon_features[17], surface_features[16]`
→ `SGEEBackbone` → pooled `[CLS]` (768)
→ `SGEEProjection` (encode lexicon→32, surface→16; concat→816; MLP [768,512] + residual + LayerNorm) → SGEE embedding (512)
→ `EmotionHead` (→ 28 logits, sigmoid) **and** `VADHead` (→ 3, tanh).

---

## Model & method (in code)

### RoBERTa backbone (`src/models/backbone.py`)
- `SGEEBackbone` loads `roberta-base` via `AutoTokenizer` / `AutoConfig` / `AutoModel`. Hidden size 768 is read from the model config and hard-assumed downstream (`SGEEProjection.backbone_dim = 768`).
- Pooling is configurable (`cls` default, plus attention-masked `mean_pool` / `max_pool`); default uses the first-token (`[CLS]`) hidden state.
- Gradient checkpointing is enabled by default; `freeze_layers` can freeze the first N encoder layers (default 0 → fully fine-tuned).
- Optional attention-weight extraction (last layer, averaged across heads) is wired for attribution but not central to training.

### VAD grounding mechanism
Two complementary mechanisms:
1. **VAD regression head** — `VADHead` regresses a 3-d `tanh`-bounded VAD vector from the SGEE embedding; supervised against per-text VAD targets.
2. **Prototype contrastive grounding** (Stage 2) — `VADPrototypeManager` runs **KMeans (k=8 default, `random_state=42`)** over the NRC-VAD lexicon (filtered to values in ~[0,1]) to obtain VAD cluster centers, each given a human-readable descriptor (e.g. "Very Positive, High Energy, High Control") and example words. `LearnedPrototypeEmbeddings` holds k trainable 512-d vectors (normalized). Each text is hard-assigned to its nearest prototype by `torch.cdist` between its VAD target and the prototype centers; an InfoNCE loss then pulls the SGEE embedding toward that prototype embedding. A `cache/vad_prototypes_k4` directory also exists, and `models/Stage2Reclustered/` re-runs with re-clustered prototypes (k=4 variant).

### Projection heads (`src/models/sgee_projection.py`, `heads.py`)
- `LexiconFeatureEncoder`: 17 → (2×32 → 32) with LayerNorm/ReLU/Dropout.
- `SurfaceFeatureEncoder`: 16 → 16 with a final `Tanh` (bounded).
- `SGEEProjection`: concat 816 → MLP over `projection_hidden_dims=[768,512]` with GELU; residual projection (816→512) added; final LayerNorm. Output 512-d. Xavier init throughout.
- `EmotionHead`: 512 → 256 → 128 → 28, sigmoid; threshold 0.5 for predictions; carries the 28 GoEmotions label names.
- `VADHead`: 512 → 128 → 128 → 3, `tanh`.

---

## Training phases — exactly how the multi-phase training works

The README presents **two** phases (Stage 1 + Stage 2). The code implements **three** (Stage 3 exists and is functional, though the README explicitly scopes this release to Stages 1-2). All stage weighting lives in `losses.py` / `stage2_losses.py` / `stage3_losses.py`.

### Stage 1 — supervised emotion classification (+ light VAD) — `trainStage1.py`
- Multi-label **BCE** emotion loss (`EmotionLoss` = `BCEWithLogitsLoss`, optionally with class `pos_weight` from dataset stats) + **Huber** VAD regression (`VADLoss`, `huber_delta=0.1`).
- Stage weights (`MultiTaskLoss._adjust_weights_for_stage`, stage `'emotion'`): `emotion=1.0, vad=0.5, contrastive=0.0`. **No contrastive loss in Stage 1.**
- Interactive runner offers quick (50 train / 2 ep), development (1000 / 3 ep), full (~40k GoEmotions / 5 ep), or custom; auto flags `--auto_quick/--auto_dev/--auto_full`.
- Saves a `_complete.pt`, a `_weights.pt`, and a JSON metrics report.

### Stage 2 — VAD grounding + prototype contrastive alignment — `trainStage2.py` / `stage2_losses.py`
- **Loads the Stage 1 checkpoint** (`stage1_model_path` set in the script) and continues training. `testRuns.py` is a dedicated Stage1→Stage2 weight-compatibility checker.
- `Stage2MultiTaskLoss` (subclasses `MultiTaskLoss`, stage `'vad_grounding'`) composes:
  - **`AdaptiveEmotionLoss`** — `BCEWithLogitsLoss` with hard-coded GoEmotions `pos_weight` (mostly 10.0, 9.59 for class 0, 2.05 for neutral) **plus** class-balanced "effective-number" per-class scaling (Cui et al.), softened by `√` and clipped to [0.5, 2.0].
  - **`WeightedVADLoss`** — Huber (`delta=0.05`) with per-dimension weights `[1.0, 1.2, 1.2]` (V, A, D) and optional lexicon-confidence weighting.
  - **`ContrastiveVADLoss`** — InfoNCE between L2-normalized SGEE embeddings and prototype embeddings at `temperature=0.10`, **plus** a small `smooth_l1` VAD-to-assigned-center alignment term (`+ 0.05 * vad_align`). Real NRC prototype centers are injected via `set_centers(...)`.
  - Two **drop-in regularizers**: an **equipartition** KL term (encourage using all prototypes evenly, default weight ~0.02) and a **soft-assignment** auxiliary (softmax over VAD-space distances as soft targets, default weight ~0.05).
  - Stage 2 weights: `emotion=1.0, vad=1.0, contrastive_vad=0.2` (persuasion/demographic = 0).
- The total Stage 2 loss is therefore: `L = L_emotion(adaptive BCE) + L_vad(weighted Huber) + 0.2·L_contrastive_vad + 0.02·L_equipartition + 0.05·L_soft_align`.

### Stage 3 — domain adaptation to persuasion + anti-forgetting — `stage3_losses.py` / `Stage3TrainingScript2.py`
(Present in code; beyond the README's stated release scope.)
- Adds **persuasion heads** (`persuasion_heads.py`): six sigmoid regression outputs — `cta_strength, scarcity, urgency, social_proof, clickbait_score, actionability` — plus a 64-d **persuasion fingerprint** and a confidence scalar. All are regression (0-1), not classification.
- Trains on a **unified multi-dataset** mix (`stage3_unified_dataset.py`): Upworthy + WANDS + Webis clickbait, with ~20% GoEmotions retained for anti-forgetting.
- Loss terms (per the Stage 3 loss/trainer): `emotion≈0.5, vad≈0.8, contrastive_vad≈0.2, persuasion=1.0 (MSE), anti_forgetting≈0.3 (KL-distillation from the frozen Stage 2 model, T=2.0, α=0.5), domain_adaptation≈0.1 (MMD-style alignment)`.
- A **DistilBERT teacher** (`data/teacherModelWebis.py`; cached under `data/processed/teacher_clickbait_distilbert/`) generates clickbait pseudo-labels for the Webis data.

---

## Datasets & VAD lexicon

| Resource | Role | Where used |
|---|---|---|
| **GoEmotions** (27 emotions + neutral = 28 classes) | Stage 1/2 supervised emotion labels | `datasetsProj.py` (loads `data/preprocessed_cache/goemotions_{train,val}_preprocessed.pt`, else HF `datasets.load_dataset`) |
| **NRC-VAD Lexicon v2.1** (Saif Mohammad) | VAD feature extraction + KMeans prototype centers + VAD regression targets | `features.py`, `prototypes.py`; files under `data/lexicons/NRC-VAD-Lexicon-v2.1/` |
| **Upworthy Archive** | Stage 3 persuasion/engagement | `data/upworthy_preprocessing.py`, `data/raw/upworthy-archive-datasets/` |
| **WANDS** | Stage 3 social-proof / relevance | `data/wands_preprocessing.py`, vendored at `data/raw/WANDS/` |
| **Webis Clickbait-17 / -22** | Stage 3 clickbait (+ DistilBERT teacher pseudo-labels) | `data/webis_preprocessing.py`, `data/teacherModelWebis.py` |
| AFINN, VADER, inquirer, eMFD, persuasion keywords | present in `data/raw/` | auxiliary lexicons (not all wired into the core model) |

**Important code-derived caveat — where VAD targets come from.** In `datasetsProj.py`, the GoEmotions `vad_targets` are set to the **mean NRC-VAD of the lexicon-matched tokens in each text** (`lexicon_obj.vad_mean.{valence,arousal,dominance}`), not gold human VAD annotations. So the VAD head is trained to predict the *lexicon's aggregate VAD* of a sentence. This makes the reported VAD MAE values (≈0.05-0.06) easy to interpret as "how close the head's prediction is to the lexicon average," and it means the contrastive grounding is anchored to lexicon-derived VAD rather than an independently annotated VAD gold standard.

---

## Evaluation & metrics

Metrics are computed in `src/training/trainer.py` (the dedicated `evaluation.py` is a stub). Computed metrics:
- **Emotion (multi-label):** macro-F1 (`emotion_f1_macro`), micro-F1 (`emotion_f1_micro`), per-class/overall accuracy, **subset (exact-match) accuracy**, balanced accuracy, confidence/entropy, top-k accuracy.
- **VAD regression:** overall **MAE** (`vad_mae_overall`) and **RMSE** (`vad_rmse_overall`); per-dimension stats. (Pearson per-dimension is referenced in baseline reporting.)
- **Persuasion (Stage 3):** MAE / RMSE / correlation per tactic.
- **Training diagnostics:** gradient norm, samples/sec, peak memory.

**Actual reported numbers** (from `models/Baselines/AllBaseLineMetrics.txt` and `baseline_*_output/*/results.json`; GoEmotions val):

| Run | Val macro-F1 | Val accuracy | Val subset-acc | Val VAD MAE |
|---|---|---|---|---|
| Baseline A (NRC-VAD high-weight) | 0.4959 | — | — | 0.0574 |
| Baseline B (random prototypes) | 0.4814 | — | — | 0.0560 |
| Baseline C (reclustered, low weight) | 0.4900 | — | — | 0.0567 |
| Baseline D (k=4 prototypes) | 0.4832 | — | — | 0.0666 |
| Baseline E (Stage 1 only) | 0.4977 | 0.9661 | — | — |
| Baseline F (Stage 2, no contrastive) | 0.5045 | — | — | 0.0506 |
| Baseline G (RoBERTa-only, features zeroed) | 0.5044 | — | — | 0.1126 |
| SGEE Stage 2 (seed 42) | 0.4937 | 0.9448 | 0.2438 | 0.0566 |
| RoBERTa plain (seed 42, 5 ep) | 0.4732 | 0.9696 | 0.4735 | — |
| RoBERTa multitask (seed 42, 5 ep) | 0.4706 | 0.9701 | 0.4749 | 0.0622 |

Honest reading: macro-F1 sits in a tight ~0.47-0.50 band across all variants — the lexicon/surface/prototype machinery does not, on these logs, produce a large emotion-F1 gain over plain or multitask RoBERTa. Baseline G's high VAD MAE (0.11 vs ~0.05) shows the lexicon/surface features matter mostly for the VAD-prediction task (unsurprising, since VAD targets are themselves lexicon-derived).

---

## Tech stack & dependencies

- **Core ML:** PyTorch (`torch>=2.0`), HuggingFace **Transformers** (`>=4.30`, RoBERTa-base), HuggingFace **`datasets`** (GoEmotions loading), **scikit-learn** (`KMeans` for prototypes).
- **Numerics/IO:** numpy, pandas, scipy, pickle, json, yaml, tqdm.
- **Viz / reporting:** matplotlib, seaborn (`Project Data/generate_figures.py`).
- **Serving:** Flask + flask_cors (`src/server.py`), static `index.html` UI.
- **Distillation:** a DistilBERT teacher (Transformers) for clickbait.

**Dependency discrepancy:** `requirements.txt` lists only `torch, transformers, numpy, pandas, scikit-learn, tqdm, pyyaml`. The code additionally imports `datasets`, `scipy`, `matplotlib`, `seaborn`, `flask`, and `flask_cors` — none of which are pinned. A clean environment installed from `requirements.txt` alone would fail to run dataset loading, figure generation, or the server.

---

## Build / run / train (actual commands)

```bash
git clone https://github.com/AmeyaBorkar/SGEE-Semantically-Grounded-Emotion-Embeddings.git
cd sgee_aca_project
pip install -r requirements.txt
# Also needed in practice (not pinned): pip install datasets scipy matplotlib seaborn flask flask-cors

# Stage 1 — emotion classification (interactive mode menu, or auto flags)
python src/training/trainStage1.py            # interactive
python src/training/trainStage1.py --auto_quick   # 50 samples / 2 epochs smoke test
python src/training/trainStage1.py --auto_full    # full GoEmotions / 5 epochs

# Stage 2 — VAD grounding + contrastive (edit stage1_model_path inside the script first)
python src/training/trainStage2.py

# Verify Stage 1 → Stage 2 weight compatibility
python testRuns.py

# Baselines / ablations
python src/training/train_baseline_roberta_plain.py
python src/training/train_baseline_roberta_multitask.py
python src/training/train_baseline_E.py   # ... F, G, H

# Multi-seed (42/43/44)
python src/training/run_multi_seed.py

# Inference server (Flask + web UI at /)
python src/server.py
```

**Run-as-is gotchas found in code:**
- `trainStage1.py` hard-codes a **Colab path** for the lexicon: `lexicon_dir = r"/content/lexicons/lexicons/NRC-VAD-Lexicon-v2.1"`. On any non-Colab machine this must be changed to the repo's `data/lexicons/NRC-VAD-Lexicon-v2.1`.
- Several training scripts have filenames with spaces/parentheses (e.g. `Stage2RetrainingScript-Re-clusteredPrototypes(Single Epoch, Fixed).py`, `data/preprocess_stage3_FIXED (1).py`), which complicates `python <file>` invocation.
- Internal `import` statements assume `src/` is on `sys.path` (scripts insert it at runtime) — running from arbitrary CWDs may need `PYTHONPATH=src`.

---

## Status, completeness & notable gaps

- **Functional and trained:** Stage 1, Stage 2, Stage 3, the baseline suite, and the Flask server are all implemented; logged metrics and saved checkpoints exist under `models/Baselines/`, `models/Stage1RemasteredRun/`, `models/Stage2Reclustered/`.
- **Single commit:** the entire project landed in one "Initial commit" — no incremental history to trace design evolution.
- **`evaluation.py` is a stub** (one comment line); all metric logic actually lives in `trainer.py`.
- **`configs/base_config.yaml` is empty** (one comment); the README's claim that hyperparameters can be overridden via this YAML is technically supported by `SGEEConfig.load_from_yaml`, but the shipped file contains nothing.
- **VAD supervision is lexicon-derived, not gold** (see Datasets) — a meaningful methodological limitation for a project whose headline is "grounding in VAD space."
- **No `LICENSE` file** despite the README's "academic use" statement.
- **Requirements are under-specified** (missing `datasets`, `scipy`, `matplotlib`, `seaborn`, `flask`, `flask_cors`).
- **Hard-coded environment paths** (Colab `/content/...`) and a hard-coded `roberta-base` hidden size of 768 reduce portability to other backbones.
- **Duplicate / variant scripts** (`stage3_training_script.py` vs `Stage3TrainingScript2.py`, `preprocess_stage3.py` vs `preprocess_stage3_FIXED (1).py`, multiple Stage-1/Stage-2 trainer variants) reflect research iteration rather than a cleaned-up release.

---

## README vs. code

| README claim | Code reality |
|---|---|
| "This release covers **Stage 1 and Stage 2**." | Stage 3 (persuasion + domain adaptation + DistilBERT distillation) is fully present in `src/training/stage3_*`, `persuasion_heads.py`, `stage3_unified_dataset.py`. The "two-phase" description undercounts the implemented pipeline. |
| Stage 1 losses: "BCE, VAD regression". | ✅ Matches (`EmotionLoss` BCE + `VADLoss` Huber; contrastive weight 0 in Stage 1). |
| Stage 2: "Prototype-contrastive, weighted VAD, adaptive emotion." | ✅ Matches `ContrastiveVADLoss` + `WeightedVADLoss` + `AdaptiveEmotionLoss`; code adds *undocumented* equipartition and soft-assignment regularizers. |
| VAD targets implied to be ground-truth VAD. | ❌ Not stated, but code derives GoEmotions VAD targets from **lexicon `vad_mean`**, not human annotations — a material caveat the README omits. |
| Dependencies: "PyTorch, HuggingFace Transformers, scikit-learn" + 7-line `requirements.txt`. | ⚠️ Incomplete: code also requires `datasets`, `scipy`, `matplotlib`, `seaborn`, `flask`, `flask_cors`. README even says "A requirements.txt will be generated separately" — it exists but is under-specified. |
| Repo structure lists `src/config.py`, models, data, training, etc. | ✅ Broadly accurate, but omits the Flask `server.py`/`index.html`, the `models/Baselines` and `Stage2Reclustered` trees, and the per-dataset Stage 3 preprocessors. |
| Config table (`backbone_name=roberta-base`, `sgee_embedding_dim=512`, `batch_size=16`, `backbone_lr=2e-5`, `heads_lr=1e-4`, `max_epochs=6`, `emotion_num_classes=28`). | ✅ All match `ModelConfig`/`TrainingConfig` defaults in `src/config.py` (note: `lexicon_feature_dim=32`, `surface_feature_dim=16`, but **raw** lexicon/surface input dims are 17/16). |
| "Configurable via `configs/base_config.yaml`." | ⚠️ Loader supports it, but the shipped YAML is empty. |
| Install via `git clone https://github.com/<your-username>/...`. | ⚠️ Placeholder URL never updated to the real repo. |
| GoEmotions "(27+1 classes)". | ✅ 28 classes (27 + neutral), label list hard-coded in `heads.py` and `datasetsProj.py`. |

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\sgee_aca_project. Source of truth: https://github.com/AmeyaBorkar/SGEE-Semantically-Grounded-Emotion-Embeddings*
