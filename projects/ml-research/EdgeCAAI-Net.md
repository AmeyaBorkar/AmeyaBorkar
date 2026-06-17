# EdgeCAAI-Net

> A ~1.42M-parameter "Conformer-Lite" music-genre classifier (PyTorch) whose real contribution is methodological: by training the *same* model on a **track-disjoint** vs an **artist-disjoint** split of FMA-small, the repo empirically demonstrates that standard genre benchmarks leak artists across train/test. The track-disjoint split here has 559 of 765 test artists also present in training; the artist-disjoint split has zero artist overlap, and macro-F1 drops 0.5598 → 0.4546 (−0.105). The model also carries early-exit heads (compute-adaptive inference) and a Gradient-Reversal-Layer / GroupDRO adversarial branch intended to push the backbone toward artist-invariant representations.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/EdgeCAAI-Net |
| **Visibility** | Public |
| **Category** | ml-research |
| **Primary language(s)** | Python (100% of source); YAML for experiment configs |
| **Local path** | `C:\Users\ameya\Documents\SoundModel` |
| **Default branch** | `master` (21 commits; HEAD `6646e30`) |
| **Lines of code (computed)** | 2,607 lines of Python across 16 `.py` files; 536 lines of YAML across 16 config files |
| **Source files (computed)** | 16 Python files, 16 YAML configs (excludes cached `.pt` tensors, datasets, `__pycache__`) |
| **Key dependencies** | `torch` (≥2.1), `torchaudio` (mel transform), `librosa` (audio decode only), `soundfile`, `numpy`, `pandas`, `scikit-learn`, `pyyaml`, `tqdm`, `matplotlib`/`seaborn`, `ptflops` (listed, unused), `torchvision` (MobileNet baseline) |
| **License** | None — no `LICENSE` file present. README states "released for academic and research purposes" |
| **Last commit** | `6646e30` — "Plan phase 4 final evaluation" — 2026-05-01 |

---

## What it actually is

EdgeCAAI-Net (Edge **C**ompute-**A**daptive, **A**rtist-**I**nvariant Net) is a self-contained research codebase, not a packaged library. It comprises:

1. **A small audio classifier** (`models/edgecaai_net.py`) — a "Conformer-Lite" backbone (depthwise temporal conv + multi-head attention + FFN) with three early-exit heads and attentive-statistics pooling. Computed parameter count: **1,418,217 (1.42M)** as instantiated by the training script.
2. **An artist-leakage experiment harness** — `scripts/make_splits.py` builds *track-disjoint* and *artist-disjoint* splits of FMA-small with explicit `assert`-based leakage verification; the same model is trained on each and the macro-F1 gap is the headline result.
3. **An adversarial artist-invariance branch** (`models/artist_invariance.py`) — a Gradient Reversal Layer feeding an auxiliary artist classifier, plus a GroupDRO alternative, intended to stop the backbone using artist identity as a genre shortcut.
4. **Four baselines** (TinyCNN, SmallCRNN, MobileNetV3-Small, TinyTransformer) under `models/baselines/`, each run on both split types to show the leakage gap is model-agnostic.
5. **Supporting tooling** — GPU-batched mel-feature caching, latency benchmarking, a `run_all.py` job runner, and an epoch-by-epoch `results/training_log.md`.

The three name components map to: **Edge/Compute-Adaptive** = early exits + budget loss; **Artist-Invariant** = GRL/GroupDRO; the leakage experiment is the reason artist-invariance exists.

---

## Architecture & how it's structured (audio → features → model → heads)

End-to-end data flow:

```
raw audio (.mp3 FMA / .wav GTZAN)
   │  librosa.load → mono @ 22050 Hz                    [scripts/cache_features.py]
   ▼
3 s segments, 1.5 s hop (50% overlap), peak-normalized
   │  torchaudio MelSpectrogram (n_fft=1024, hop=512, n_mels=64), on GPU in batches
   ▼
log-mel:  log1p(10 · mel)  →  cached as data/processed/<set>/<track>_seg<k>.pt
   │  CachedMelDataset loads .pt, transposes to (T, 64)   [train.py]
   │  optional SpecAugment (time+freq mask) + random gain
   ▼
EdgeCAAINet.forward(x: (B, T, 64))                       [models/edgecaai_net.py]
   Stem (DW-sep 2D conv → flatten freq → Linear → LayerNorm) → (B, T, 128)
   6 × ConformerLiteBlock
        ├─ after block 2 → Exit head 1  (logits)
        ├─ after block 4 → Exit head 2  (logits)
        └─ after block 6 → final AttentiveStatsPooling → ClassifierHead (Exit 3)
   final embedding (256-d) → gate Linear (budget) ; → ArtistClassifier+GRL (adversarial, in train.py)
   ▼
deep-supervision loss over 3 exits + budget loss + artist-invariance loss
```

Annotated source tree (datasets, `.pt` caches, `__pycache__` omitted):

```
SoundModel/
├── models/
│   ├── edgecaai_net.py        # 308 LOC — main model: Stem, ConformerLiteBlock,
│   │                          #   AttentiveStatsPooling, ClassifierHead, EdgeCAAINet
│   ├── exits.py               # 145 LOC — EarlyExitHead, ExitManager,
│   │                          #   compute_deep_supervision_loss, compute_budget_loss
│   ├── artist_invariance.py   # 161 LOC — GradientReversalLayer, ArtistClassifier,
│   │                          #   GroupDRO, compute_artist_invariance_loss
│   ├── __init__.py
│   └── baselines/
│       ├── tiny_cnn.py        # depthwise-separable CNN
│       ├── small_crnn.py      # CNN stem + BiGRU
│       ├── mobilenet_baseline.py # torchvision MobileNetV3-Small, 1-ch input
│       └── tiny_transformer.py   # TransformerEncoder, no exits/GRL
├── configs/
│   ├── default.yaml           # all shared hyperparameters
│   ├── fma_track_disjoint.yaml / fma_artist_disjoint.yaml / gtzan_config.yaml
│   ├── baselines/  (8 configs: each baseline × {track,artist}-disjoint)
│   └── ablations/  (4 configs: backbone_only, exits_only, exits_budget, full_groupdro)
├── scripts/
│   ├── make_splits.py         # 255 LOC — split generation + leakage assertions
│   ├── cache_features.py      # 192 LOC — GPU-batched mel caching
│   └── measure_latency.py     # 154 LOC — per-exit + adaptive CPU latency
├── train.py                   # 469 LOC — main training loop (exits+budget+GRL/GroupDRO)
├── train_baseline.py          # 263 LOC — plain CE training for baselines
├── eval.py                    # 238 LOC — early-exit inference + macro-F1/BA/ECE/per-exit
├── run_all.py                 # 187 LOC — runs all baseline+ablation jobs, resume-safe
├── data/splits/*.json         # generated splits (present in tree, with artist_id per track)
├── results/training_log.md    # full epoch-by-epoch logs (source of the headline numbers)
└── requirements.txt
```

---

## Model architecture (in code) — layers, the gradient-reversal/adversarial branch, parameter count

**Backbone** (`EdgeCAAINet`, defaults `d_model=128, n_heads=4, n_blocks=6, ffn_dim=256, n_mels=64`):

- **Stem** — `DepthwiseSeparableConv2d(1→64)` over the spectrogram treated as a 1-channel image, then the frequency axis (`stem_channels × n_mels = 64×64 = 4096`) is flattened and projected by a `Linear(4096 → 128)` + `LayerNorm`. This is the single largest parameter block (~524K params).
- **6 × `ConformerLiteBlock`** — each block is three pre-norm residual sub-modules: (1) `DepthwiseTemporalConv` (Conv1d, kernel 11, groups=d_model) for local temporal patterns, (2) `nn.MultiheadAttention` (4 heads, batch_first), (3) FFN `Linear(128→256)→GELU→Dropout→Linear(256→128)`. No subsampling — temporal resolution is preserved through all blocks.
- **Final head** — `AttentiveStatsPooling` produces a 256-d vector (attention-weighted **mean ⊕ std**), fed to `ClassifierHead` (`Dropout→Linear(256→128)→GELU→Linear(128→num_classes)`).
- **Early exits** — `ExitManager` holds an `EarlyExitHead` at blocks 2 and 4 (block 6 is the final classifier). Each exit head has its own lightweight `AttentivePooling1D`, an MLP classifier, AND a separate sigmoid `confidence_head` (the confidence_head is defined and exposed via `get_confidence`, but `inference()` actually uses softmax-max confidence, not the learned head — the learned confidence estimator is effectively unused at inference).
- **Gating** — `self.gate = Linear(256 → 3)` produces per-exit gate logits consumed only by `compute_budget_loss`.

**Inference (`EdgeCAAINet.inference`)** walks blocks, and at each early-exit position computes softmax-max confidence; if the batch-mean confidence ≥ threshold it returns early, otherwise falls through to the final exit. Default threshold in code = **0.8**.

**Adversarial / artist-invariance branch** (`models/artist_invariance.py`):

- `_GradientReversal(torch.autograd.Function)` — identity on forward, `−alpha · grad` on backward.
- `GradientReversalLayer(alpha=0.2)` wraps it; `ArtistClassifier` = `GRL → Linear(256→128) → ReLU → Linear(128→num_artists)`.
- The adversarial objective: the artist classifier tries to predict the track's artist from the 256-d embedding; because gradients are reversed before reaching the backbone, the backbone is pushed to make the embedding *un*predictive of artist — i.e. artist-invariant — while still classifying genre. This is exactly the intended fix for the leakage the experiment exposes.
- `GroupDRO` is the alternative: treats each artist as a group, keeps per-group weights updated by exponentiated gradient, and optimizes the worst-group loss.

> **Important code detail / dead-code flag:** `EdgeCAAINet.__init__` also defines its *own* `self.artist_head = nn.Linear(d_model*2, num_artists)` gated on `num_artists>0`. But every call site (`train.py`, `eval.py`, `measure_latency.py`) constructs the model **without** passing `num_artists`, so it defaults to 0 and this internal head is never created. The GRL training actually uses the **external** `ArtistClassifier` from `artist_invariance.py`, instantiated separately in `train.py` and added to the optimizer. The model's built-in `artist_head` and its `result['artist_logits']` path are dead code. Notably the built-in head has **no GRL**, so even if enabled it would not produce adversarial (invariant) training — only the external `ArtistClassifier` applies gradient reversal.

**Computed parameter counts** (verified by instantiating the modules):

| Module | Params |
|---|---|
| `EdgeCAAINet` (num_artists=0, as trained) | **1,418,217 (1.42M)** |
| External `ArtistClassifier` (GRL, e.g. ~500 artists) | ~97K (separate optimizer params, not in the 1.42M) |
| TinyCNN baseline | 650,504 (0.65M) |
| SmallCRNN baseline | 981,448 (0.98M) |
| TinyTransformer baseline | 687,112 (0.69M) |
| MobileNetV3-Small baseline | 1,525,768 (1.53M) |

The README's "~1.42M parameters" is **accurate**. The README baseline param labels (0.65M / 0.98M / 1.53M / 0.69M) all match the computed values.

---

## The artist-leakage finding — how the code demonstrates/controls for it

The leakage demonstration lives in `scripts/make_splits.py` and is corroborated by `results/training_log.md`:

- **`make_track_disjoint_split`** — splits *tracks* (genre-stratified `train_test_split`, 70/15/15). Tracks are disjoint across splits, but **artists are not** — the same artist's songs can land in both train and test.
- **`make_artist_disjoint_split`** — groups all of an artist's tracks together, shuffles *artists*, and assigns whole artists to train/val/test, guaranteeing no artist appears in two splits.
- **`verify_no_leakage`** — `assert`s track-disjointness for both modes, and additionally `assert`s artist-disjointness when `split_type == "artist_disjoint"`.

**Empirical proof from the actual generated split files** (`data/splits/*.json`, measured directly):

| Split type | train artists | test artists | **train∩test artist overlap** |
|---|---|---|---|
| track_disjoint | 1929 | 765 | **559** (artists leak) |
| artist_disjoint | 1616 | 347 | **0** (clean) |

So the standard "track-disjoint" protocol leaks ~73% of test artists into training. Same model, same hyperparameters, only the split changes:

| Split | Artist Inv. | Best Val Macro-F1 | Best Val BA | (from training_log.md) |
|---|---|---|---|---|
| FMA track-disjoint | none | 0.5598 (epoch 8) | 0.5620 | overfits: train-F1 0.87, val plateaus |
| FMA artist-disjoint | GRL α=0.2 | 0.4546 (epoch 14*) | 0.4647 | |

**Headline drop: −0.105 macro-F1 (≈18.8% relative)**, and the gap reproduces across every baseline (TinyCNN −0.112, SmallCRNN −0.104, EdgeCAAI-Net −0.105). This consistency is the core evidence that the inflation is an artifact of the splitting protocol, not of any one architecture.

**Control / mitigation:** the artist-disjoint runs add the GRL adversarial loss (`ai_cfg.method == "grl"` in `train.py`) so the backbone is penalized for encoding artist identity. Note the artist-invariance loss is applied only to the **final** embedding (not per-exit), and only on samples with a valid `artist_id >= 0` (GTZAN, which has no artist metadata, is excluded). The README frames GRL as *fixing* leakage; the code shows it as a regularizer applied on the clean (artist-disjoint) split, trading a little raw accuracy for invariance — the ablation table shows exits+budget (no GRL) actually scores slightly higher F1 (0.4662) than the GRL run (0.4546), so GRL's benefit is invariance, not headline F1.

---

## Audio feature pipeline (librosa)

Defined in `scripts/cache_features.py` and parameterized by `configs/default.yaml → audio`/`features`:

- **Loading:** `librosa.load(file_path, sr=22050, mono=True)`. librosa is used **only for decoding** (chosen explicitly to dodge torchcodec/FFmpeg issues on Windows — see the docstring). This is the single librosa touchpoint in the whole pipeline.
- **Segmentation:** fixed 3.0 s windows with a 1.5 s hop (50% overlap); short clips are zero-padded to one segment.
- **Normalization:** peak normalization (`waveform / max|waveform|`).
- **Spectrogram:** computed with **`torchaudio.transforms.MelSpectrogram`** (`n_fft=1024`, `hop_length=512`, `n_mels=64`), batched on GPU when available — **not** `librosa.feature.melspectrogram`.
- **Compression:** `torch.log1p(10 · mel)` i.e. `log(1 + 10·mel)` — a custom log compression, **not** dB (`librosa.power_to_db`) and **not** MFCC. There are no MFCCs, chroma, or hand-crafted features anywhere; the model consumes raw 64-bin log-mel spectrograms only.
- **Caching:** each segment saved as `<track_id>_seg<k>.pt` containing `{log_mel, genre_idx, track_id, segment_idx}`. Crucially the cached tensors **do not store `artist_id`** — `CachedMelDataset` recovers `artist_id` from the split JSON at load time, which is how the GRL/GroupDRO branch gets its labels.

> **README discrepancy:** the README's prose says "Audio loading uses `librosa` throughout" and lists the mel step as if librosa-based, but the actual mel transform is `torchaudio`. librosa is decode-only.

---

## Datasets & evaluation

**Datasets** (gitignored; not in the repo — only metadata-derived split JSONs and a partial `.pt` cache are tracked):

- **FMA-small** (`Datasets/fma/fma_small` + `fma_metadata`) — the `small` subset, genre-top labels, 8 genres: Electronic, Experimental, Folk, Hip-Hop, Instrumental, International, Pop, Rock. `make_splits.py` reads `tracks.csv`/`genres.csv` (multi-index header) and pulls `artist_id` from `(artist, id)`. Generated split sizes (track-level records): track-disjoint ≈ 5599/1200/1201; artist-disjoint ≈ 5523/1203/1274 (train/val/test). Each track expands to ~18 cached 3 s segments.
- **GTZAN** (`Datasets/gtzan_kaggle/.../genres_original`) — 10 genres, discovered from directory structure; **`artist_id = -1`** (no artist metadata), so artist-disjoint requests silently fall back to track-disjoint and GRL is disabled. Used as a "legacy benchmark" reference; reaches the highest F1 (0.7973) precisely because it is small and uncontrolled for artists.

**Evaluation** (`eval.py`):

- Runs `model.inference()` per sample with a confidence threshold (CLI default 0.8), recording the chosen exit index.
- Reports **macro-F1**, **balanced accuracy**, **Expected Calibration Error** (15-bin ECE, hand-implemented), a confusion matrix, and **per-exit statistics** (count, %, F1, accuracy, mean confidence). Training-time validation (`train.py`/`train_baseline.py`) tracks macro-F1 + balanced accuracy and early-stops on val macro-F1 (patience 10).
- `scripts/measure_latency.py` benchmarks full-model vs adaptive (early-exit) CPU latency over 100 runs and reports the exit distribution — the "compute-adaptive" payoff.

The README's results tables (main model, baselines, ablations) match `results/training_log.md`. Note the log shows several cells still **pending** (MobileNet artist-disjoint, both TinyTransformer runs, GroupDRO ablation) — the README presents some of these as blanks/dashes too, so the experiment grid is genuinely incomplete, consistent with the last commit being "Plan phase 4 final evaluation."

---

## Tech stack & dependencies

- **Framework:** PyTorch (≥2.1), `torch.nn` / `torch.autograd.Function` (custom GRL), `torch.utils.data`.
- **Audio:** `torchaudio` (mel transform), `librosa` + `soundfile` (decode), 22.05 kHz mono.
- **Data/metrics:** `pandas` (FMA CSV parsing), `numpy`, `scikit-learn` (`train_test_split`, `f1_score`, `balanced_accuracy_score`, `confusion_matrix`).
- **Baseline C:** `torchvision.models.mobilenet_v3_small` adapted to 1-channel input.
- **Config:** `pyyaml` with a hand-rolled `defaults:`-key deep-merge in `load_config`.
- **Listed but unused in source:** `ptflops` (in requirements, no import found — FLOP counting was presumably planned). `matplotlib`/`seaborn` are listed for plotting; `results/plots` and `results/tables` are empty (`.gitkeep` only), so plotting/table export isn't wired up yet.

No package metadata (`setup.py`/`pyproject.toml`), no tests, no CI — it is a scripts-and-configs research repo.

---

## Build / run / train / eval (actual commands)

From the code's real argparse signatures (the README's example flags are partly wrong — see below):

```bash
# 0. Install
pip install -r requirements.txt              # Python 3.10+

# 1. Generate splits — ACTUAL flags from make_splits.py
python scripts/make_splits.py --dataset fma --split-type track_disjoint
python scripts/make_splits.py --dataset fma --split-type artist_disjoint
python scripts/make_splits.py --dataset gtzan --split-type track_disjoint
#   (FMA paths default to Datasets/fma/...; override with
#    --fma-audio-dir / --fma-metadata-dir / --output-dir)

# 2. Cache log-mel features — ACTUAL flags (note: --output-dir, not --out-dir)
python scripts/cache_features.py \
    --split-file data/splits/track_disjoint_train.json \
    --output-dir data/processed/fma_track \
    --device cuda --batch-size 64
#   repeat for val/test and for artist_disjoint_* / gtzan_track_disjoint_*

# 3. Train
python train.py --config configs/fma_track_disjoint.yaml      # no GRL
python train.py --config configs/fma_artist_disjoint.yaml     # GRL α=0.2
python train.py --config configs/gtzan_config.yaml
python train_baseline.py --config configs/baselines/tiny_cnn_track.yaml
python run_all.py                  # all baselines+ablations, resume-safe
python run_all.py --only baselines # or --only ablations ; --dry-run to list

# 4. Evaluate (early-exit inference)
python eval.py --config configs/fma_track_disjoint.yaml \
    --checkpoint results/fma_track_disjoint/checkpoints/best.pt \
    --threshold 0.8 --split test --output results/.../eval.json

# 5. Latency
python scripts/measure_latency.py \
    --config configs/default.yaml \
    --checkpoint results/fma_artist_disjoint/checkpoints/best.pt
```

Checkpoints are written to `<results>/checkpoints/best.pt` and tracked via Git LFS (`.gitattributes`: `*.pt filter=lfs`).

---

## Status, completeness & notable gaps

- **Working & coherent:** the model, both training loops, feature caching, split generation with leakage asserts, eval with ECE/per-exit stats, and the job runner are all complete and internally consistent. The headline leakage result is reproducible from the tracked split JSONs.
- **Incomplete experiment grid:** the training log marks MobileNet artist-disjoint, both TinyTransformer runs, and the GroupDRO ablation as *pending*; no checkpoints exist for them. The repo is mid-"phase 4."
- **Dead code:** the model's internal `artist_head`/`artist_logits` path is never used (and lacks GRL). The per-exit learned `confidence_head` is trained-adjacent but unused at inference (softmax-max is used instead).
- **Empty output dirs:** `results/plots` and `results/tables` are placeholders; `ptflops`/seaborn are declared but no FLOP-count or plotting code exists.
- **No tests, no license file, no packaging.**
- **Reproducibility caveat:** GroupDRO `update_weights` runs every step on potentially sparse per-batch group losses, which can be noisy; and the GRL `alpha` is fixed (no annealing schedule, despite `set_alpha` existing for it).

---

## README vs. code

| Claim in README | Reality in code | Verdict |
|---|---|---|
| "~1.42M parameters" | Computed `EdgeCAAINet` = 1,418,217 | ✅ Accurate |
| Baseline params 0.65/0.98/1.53/0.69M | Computed 0.65/0.98/1.53/0.69M | ✅ Accurate |
| Track→artist F1 drop −0.105 (18.8%) | 0.5598 → 0.4546 in training_log; matches | ✅ Accurate |
| "Audio loading uses librosa **throughout**" / mel via librosa | librosa is **decode-only**; mel is `torchaudio.transforms.MelSpectrogram` | ⚠️ Misleading — torchaudio computes features |
| Inference confidence threshold "default: 0.9" (Architecture & Step 4) | Code default is **0.8** (`inference()`, `eval.py`, `measure_latency.py`) | ⚠️ Discrepancy |
| Step 1 split cmd uses `--data-dir/--metadata-dir/--out-dir` | `make_splits.py` flags are `--fma-audio-dir/--fma-metadata-dir/--output-dir` + `--split-type` (required) | ⚠️ README commands won't run as written |
| Step 2 cache cmd uses `--out-dir` | Actual flag is `--output-dir` | ⚠️ Wrong flag name |
| "GRL feeds into an artist classifier; main network learns to fool it" | True for the **external** `ArtistClassifier` in train.py; the model's built-in `artist_head` (no GRL) is dead code | ⚠️ Mechanism correct but not via the in-model head implied by the class |
| GRL improves the model | Ablation shows exits+budget (no GRL) F1 0.4662 > GRL 0.4546; GRL trades F1 for invariance | ⚠️ README's "incremental improvement each component" overstates GRL's F1 effect (README itself notes the real benefit is the gap, which is consistent) |
| "Empirical validation that artist leakage inflates scores" | Strongly supported: 559/765 test artists leak in track-disjoint vs 0 in artist-disjoint; gap reproduces across all baselines | ✅ Core thesis well-supported by code+data |

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\SoundModel. Source of truth: https://github.com/AmeyaBorkar/EdgeCAAI-Net*
