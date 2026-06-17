# LocateVision

> A campus location image classifier: a hybrid **ResNet-50 + Swin-Base Transformer** feature-fusion model with a gated-attention head (PyTorch / `timm`), exposed through a single-endpoint **Flask** `/predict` API and a static multi-page HTML/CSS/JS frontend that lets a user upload a photo and see which campus spot it is. The repo's GitHub-reported primary language is HTML because the frontend (~3,900 lines across 10 self-contained `.html` files with inline CSS/JS) dwarfs the ~550 lines of Python. The model's claimed "99.76% accuracy" appears **only in prose (README + about.html)** and is never computed or logged anywhere in the committed code.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/LocateVision |
| **Visibility** | Public |
| **Category** | ml-research |
| **Primary language(s)** | HTML (~3,900 LOC, frontend) dominant; Python (~550 LOC, model + server) is the actual ML logic |
| **Local path** | `C:\Users\ameya\.repo-cache\LocateVision` |
| **Default branch** | `main` |
| **Lines of code (computed)** | 4,566 total across tracked source files: HTML **3,897**, Python **551**, README 99, requirements 19 |
| **Source files (computed)** | 15 tracked source/text files: 10 HTML, 3 Python, 1 `requirements.txt`, 1 `README.md` (+ `LICENSE`) |
| **Key dependencies** | `torch`, `torchvision` (ResNet-50), `timm` (Swin Transformer), `albumentations`, `flask`, `flask-cors`, `Pillow`, `numpy`, `tqdm` |
| **License** | MIT — `Copyright (c) 2026 Ameya Borkar` (README footer says "© 2025 LocateVision All Rights Reserved" — inconsistent with the actual MIT LICENSE file) |
| **Last commit** | `d86bb00` — "Done" — 2026-02-11 |

## What it actually is

LocateVision is a small end-to-end demo / student project (the dataset is **VIT campus food spots**) that does fine-grained scene classification: given a photo, predict which campus location it shows. It has three parts:

1. **`model/`** — a PyTorch training script (`train.py`) defining a custom `HybridClassifierWithAttention` that concatenates ResNet-50 and Swin-Base features, gates them, and classifies; plus a `deploy.py` smoke-test script.
2. **`server/`** — a Flask app (`image_classifier_server.py`) that loads a saved model and serves one `POST /predict` endpoint returning a class label + softmax confidence.
3. **`frontend/`** — 10 standalone HTML pages (landing, explore/upload, about, contact, coverage, places, and four per-location detail pages) with all CSS and JavaScript inlined. The JS in `explore.html` posts the uploaded image to the Flask `/predict` endpoint and renders a themed description per predicted location.

The number of output classes is **dynamic** in training (`num_classes=len(dataset.classes)`, i.e. one per subfolder under the data directory), but the deployed pieces hard-code label lists that **do not agree with each other** (see [README vs. code](#readme-vs-code)).

## Architecture & how it's structured (model + training + Flask serving; annotated tree)

```
LocateVision/
├── README.md                         # Project overview; source of the "99.76%" claim
├── LICENSE                           # MIT, © 2026 Ameya Borkar
├── requirements.txt                  # torch/torchvision/timm/flask/albumentations/...
├── model/
│   ├── train.py                      # Preprocess -> CustomTensorDataset -> Hybrid model -> train loop (30 epochs)
│   └── deploy.py                     # Loads a full saved model, predicts on a test dir (smoke test)
│                                     #   (README references model/weights/ — NOT present in repo, no .gitignore)
├── server/
│   └── image_classifier_server.py    # Flask app: loads model, exposes POST /predict (+CORS)
└── frontend/
    ├── index.html                    # Landing page (custom inline CSS, animated gradient)
    ├── explore.html                  # 1,615 lines: upload UI + ALL client JS (auth, history, plans, predict)
    ├── about.html                    # About page (loads Bootstrap 5.3 via CDN); repeats 99.76% claim
    ├── contact.html                  # Contact form -> POST /contact (endpoint NOT implemented server-side)
    ├── coverage.html                 # Coverage info (Bootstrap CDN)
    ├── places.html                   # Campus places overview grid
    └── locations/
        ├── fruit-canteen.html        # Per-location detail pages
        ├── vit-canteen.html
        ├── staff-canteen.html
        └── kiosk.html
```

No `.gitignore` exists in the repo despite README claiming `weights/` is "gitignored," and there is **no committed model checkpoint, no dataset, no training log, and no notebook**. The HTML files are fully static and self-contained (no build step, no bundler, no `package.json`).

## Model architecture (in code) — ResNet-50 + Swin fusion, the gated attention

Defined identically in `model/train.py`, `model/deploy.py`, and `server/image_classifier_server.py` (the class is copy-pasted into all three rather than shared as a module). Class: **`HybridClassifierWithAttention(num_classes)`** (`train.py:130`).

**Two parallel backbones, both pretrained on ImageNet:**
- **CNN branch** — `torchvision.models.resnet50(weights='IMAGENET1K_V1')`, truncated to its conv stem + `layer1..layer4`, then `AdaptiveAvgPool2d(1)` → a 2048-d vector (`train.py:134-145`).
- **Transformer branch** — `timm.create_model("swin_base_patch4_window7_224", pretrained=True)` with the final head dropped (`*list(swin.children())[:-1]`) plus `AdaptiveAvgPool2d(1)` → a 1024-d vector (`train.py:147-151`).

The two flattened vectors are **concatenated** (`torch.cat(..., dim=1)`), giving a combined dimension the model computes dynamically at init via a dummy `(1,3,224,224)` forward pass (`train.py:153-160`) — in practice **2048 + 1024 = 3072**.

**Gated attention** — the fusion head is `ShapeDebugGatedAttention(input_dim=3072, output_dim=512)` (`train.py:106-128`). Its mechanism:
```python
features_flat      = features.view(features.size(0), -1)
processed_features = self.input_proj(features_flat)          # Linear 3072->3072
projected_features = self.feature_proj(processed_features)   # Linear 3072->512
gates              = self.gate_layer(features_flat)          # Linear 3072->512 -> Sigmoid
gated_features     = gates * projected_features              # element-wise gating
```
So a sigmoid "gate" (0–1 per channel) modulates a learned 512-d projection of the fused features — adaptive feature selection over the concatenated CNN+Transformer representation. A second, simpler `GatedAttention` class is also defined but **unused** (the model wires up `ShapeDebugGatedAttention`). A `Swish` activation class (`x * sigmoid(x)`) is defined in all three files.

**Classifier head** (`train.py:167-172`): `Linear(512→256) → ReLU → Dropout(0.4) → Linear(256→num_classes)`.

> Note: a subtle inconsistency — `train.py`'s classifier uses **`nn.ReLU()`** while the copies in `deploy.py` and `server` use **`Swish()`** in the same slot. The forward passes also keep all the `print(...)` debug statements (CNN/Swin/combined/gated shapes) — these fire on **every inference call** in production, including inside the Flask server.

## Training & data pipeline — dataset, augmentation, the reported accuracy

`model/train.py` (entry point — runs at import, no `if __name__` guard):

- **Preprocessing** (`preprocess_and_save`, `train.py:26-46`): walks `RAW_DATA_DIR` (default `data/RawImages`, one subfolder per class), applies an **Albumentations** pipeline — `HorizontalFlip(0.5)`, `RandomRotate90(0.5)`, `ColorJitter(0.3/0.3/0.3/0.1, p=0.5)`, `Resize(224×224)`, ImageNet `Normalize`, `ToTensorV2` — and saves each image as a pre-transformed `.pt` tensor in `PROCESSED_DATA_DIR` (default `data/PreprocessedImages`). Augmentation is baked in **once at preprocessing time** (not re-randomized per epoch), which weakens its regularization value.
- **Dataset** (`CustomTensorDataset`, `train.py:55-70`): loads the `.pt` tensors, derives classes from directory names, `class_to_idx` mapping.
- **Split**: 80/20 train/val via `random_split` (`train.py:74-76`).
- **Hyperparameters** (`train.py:18-23`): `BATCH_SIZE=32`, `EPOCHS=30`, `LEARNING_RATE=1e-4`, `torch.manual_seed(42)`.
- **Optimization** (`train.py:197-199`): `CrossEntropyLoss`, `AdamW(lr=1e-4, weight_decay=1e-4)`, `CosineAnnealingWarmRestarts(T_0=10, T_mult=2)`.
- **Train loop** (`train.py:202-233`): standard loop; per-epoch it prints `Validation Accuracy: {accuracy:.4f}` and saves `state_dict()` to `MODEL_SAVE_PATH` (default `sota_hybrid_model_with_attention.pth`) whenever val accuracy improves; prints final "Best accuracy".

**Where does 99.76% come from?** Nowhere in code. The training loop computes accuracy as `correct/total` and prints it at runtime, but **no log, checkpoint, or output is committed**. The figure "99.76%" exists only as a hard-coded string in `README.md:11` and `frontend/about.html:145`. It is an unverifiable claim relative to the repository contents — treat it as a reported (not reproduced) number, and note that with augmentation frozen at preprocessing and an 80/20 random split on what is likely a small same-session campus dataset, a number that high is plausibly optimistic / leakage-prone.

## Serving — the Flask app, endpoints, inference flow

`server/image_classifier_server.py`:

- Loads the model from `MODEL_PATH` (default `../model/weights/FullTrainedSOTAmodel.pth`) via `torch.load(...)` — loading a **pickled full model object**, not a `state_dict`, so the class definitions must match (hence the copy-paste).
- Transform: `Resize(224×224) → ToTensor → Normalize(ImageNet)`.
- **Only one route**: `POST /predict` (`:138-152`). Reads `request.files["file"]`, runs the model, softmaxes, and returns:
  - `{"class": "<label>", "confidence": <float>}` when confidence ≥ 0.90, else `{"class": "Unknown", "confidence": ...}` (a fixed 0.90 reject threshold, `:149`).
- `CORS(app)` is enabled; runs on `0.0.0.0:5000` with `debug=True`.
- **Hard-coded labels**: `CLASS_NAMES = ["Fruit Canteen", "VIT Canteen"]` (`:133`) — only **two** classes, in fixed order, with no persisted class-index mapping from training, so correctness depends entirely on alphabetical/training order matching this list.

**Inference flow (as wired):** `frontend/explore.html` → `fetch("http://127.0.0.1:5000/predict")` with `FormData{file, username}` → Flask runs the hybrid model → returns class+confidence → JS renders the class name, `confidence*100`, and a hand-written `getFeatureDescription()` blurb per location (Kiosk / Fruit Canteen / VIT Canteen / Staff Canteen).

**Major serving gap:** the frontend calls many endpoints the committed server **does not implement** — `explore.html` hits `/init`, `/login`, `/get_user_data`, `/change_password`, `/upgrade_plan`, `/delete_account`, `/get_history`, and `contact.html` hits `/contact`. The committed `image_classifier_server.py` defines only `/predict`. The frontend also reads `data.remaining_day_limit` from the predict response, but the server never returns that field. This means a richer backend (user auth, plans, usage limits, history) existed or was intended but **is not in this repo** — only the classifier endpoint was committed.

## Tech stack & dependencies

| Layer | Tech (from code/`requirements.txt`) |
|---|---|
| Model | PyTorch (`torch`, `torchvision` ResNet-50), `timm` (Swin-Base) |
| Data/aug | `albumentations`, `Pillow`, `numpy`, `tqdm` |
| Listed but unused in code | `pandas`, `scikit-learn` (in `requirements.txt`; no import found) |
| Server | `flask`, `flask-cors`, `werkzeug` |
| Frontend | Plain HTML5 + inline CSS + vanilla JS; **Bootstrap 5.3.0 via CDN** on about/coverage/location pages only; Google Fonts; `localStorage` for client-side session/theme state. `index.html` and `explore.html` use fully custom inline CSS (no Bootstrap). |

There is no `package.json`, no Dockerfile, no CI config, and no test suite.

## Build / run / train / serve (actual commands)

```bash
# Install Python deps
pip install -r requirements.txt

# Train (note: train.py runs immediately on import — no __main__ guard)
cd model
export RAW_DATA_DIR="path/to/raw/images"          # one subfolder per class
export PROCESSED_DATA_DIR="path/to/processed"
python train.py                                   # writes sota_hybrid_model_with_attention.pth

# Smoke-test a saved FULL model (not a state_dict)
cd model
export MODEL_PATH="weights/FullTrainedSOTAmodel.pth"
export TEST_DIR="data/test_images"
python deploy.py                                  # NOTE: CLASS_NAMES is ["Purse","Pouch"] here (wrong labels)

# Serve
cd server
export MODEL_PATH="../model/weights/FullTrainedSOTAmodel.pth"
python image_classifier_server.py                 # http://0.0.0.0:5000, POST /predict

# Frontend: open frontend/explore.html in a browser (static file; expects server at 127.0.0.1:5000)
```

To actually train/serve you must supply your own dataset and produce the `.pth` weights — none are committed.

## Status, completeness & notable gaps

- **Working core:** the hybrid model, training loop, and `/predict` serving path are coherent and would run given a dataset and weights.
- **No artifacts committed:** no weights, no dataset, no training logs, no notebook → the headline 99.76% is unverifiable from the repo.
- **Backend incompleteness:** the frontend depends on ~8 endpoints (auth, history, plans, limits, contact) that the committed Flask server does not implement. As shipped, only image prediction works; login/history/plan UI will fail against this server.
- **Label inconsistencies:**
  - `server` → `["Fruit Canteen", "VIT Canteen"]` (2 classes)
  - `deploy.py` → `["Purse", "Pouch"]` (leftover from a different project; clearly stale)
  - `train.py` → dynamic `len(dataset.classes)` (could be 4+, matching the four `frontend/locations/*` pages: kiosk, fruit-canteen, staff-canteen, vit-canteen)
  These three disagree on both the number and the names of classes.
- **Potential bug:** `server/image_classifier_server.py` references `models.resnet50(...)` (`:62`) but only imports `torchvision.transforms`, **not** `torchvision.models` — instantiating `HybridClassifierWithAttention` there would raise `NameError: models`. The server "works" only because it loads a fully-pickled model object via `torch.load`, so the class is rebuilt from the pickle's own references rather than being constructed locally — but the class body is still broken if ever instantiated directly.
- **Debug noise in prod:** `print(...)` shape-debug statements execute on every forward pass, including server inference.
- **Hardcoded local paths:** `explore.html`'s `getFeatureDescription()` embeds `file:///C:/Users/ameya/OneDrive/Desktop/...` paths for map images — non-portable, machine-specific.
- **No tests, no CI, no containerization.**

## README vs. code

| README / prose claim | Reality in code |
|---|---|
| "99.76% Accuracy" | Not computed, logged, or stored anywhere. Only a string in README + `about.html`. Training computes `correct/total` at runtime but nothing is committed. |
| Hybrid ResNet-50 + Swin + Gated Attention + Swish | **True** — matches `HybridClassifierWithAttention`. (Caveat: `train.py` head uses `ReLU`, not `Swish`, unlike the other two copies.) |
| "`weights/` — gitignored" | No `.gitignore` exists in the repo; `model/weights/` is simply absent (no weights committed). |
| Frontend has "user authentication, dark mode, history tracking, and plan management" | The **UI** for these exists in `explore.html`, but the committed Flask server implements **none** of the supporting endpoints (`/login`, `/get_history`, `/upgrade_plan`, etc.) — only `/predict`. |
| Tech stack lists "Bootstrap 5" | Partly true: Bootstrap 5.3.0 CDN is loaded on about/coverage/location pages; `index.html` and `explore.html` use custom inline CSS instead. |
| License "© 2025 LocateVision All Rights Reserved" (README footer) | Actual `LICENSE` is **MIT, © 2026 Ameya Borkar** — README footer contradicts the permissive MIT license. |
| `requirements.txt` lists `pandas`, `scikit-learn` | No `import pandas` / `sklearn` anywhere in the Python — unused deps. |
| Project root shown as `ASEPojectRoot/` | Confirmed by leftover absolute paths in `explore.html` (`.../Desktop/ASEPojectRoot/...`) — original local project name. |

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\.repo-cache\LocateVision. Source of truth: https://github.com/AmeyaBorkar/LocateVision*
