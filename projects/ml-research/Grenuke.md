# Grenuke

> Team Grenuke's solution to the **Amazon ML Challenge 2026** business-entity-resolution task, which placed **3rd at the Grand Finale** (7 Oct 2026) after being named a Top-10 finalist out of **32,000+ teams**. For every Source-1 business record, the pipeline finds every Source-2/3 record that describes the same business. It uses only names and addresses and covers the US, India and **France**, a country that appears only in the test set and has no labels. The system is a cascade. Per-country multi-view token blocking (numba) feeds XGBoost stages 0–3. Multilingual cross-encoders (e5, bge, and Qwen2.5-1.5B/7B with LoRA) read only the ~2.6% of pairs the trees are unsure about. An exact **expected-F0.5 set decision** then picks matches for each entity, and generator-aware French rules plus a **Qwen2.5-7B re-check** of confident predictions finish the job. The final submission ("Composite B") scored **0.990879 macro F0.5 on the public leaderboard**. The repository is an archived competition record: a shared `ber` package, the final model chain as experiment scripts, each member's research code, and a ~20k-line curated knowledge base.

| | |
| --- | --- |
| **Repository** | https://github.com/AmeyaBorkar/Grenuke |
| **Visibility** | Public (made public on 8 Oct 2026, Phase 7 of `docs/ROADMAP.md`) |
| **Category** | ml-research |
| **Primary language(s)** | Python (246 files). Bash (93 scripts, mostly drivers for rented GPU boxes). One PowerShell setup script. |
| **Local path** | `C:\Users\ameya\Documents\GrenukeAmazon` |
| **Default branch** | `main`. History is rebase-only, with 0 merge commits. |
| **Lines of code (computed)** | **27,497 lines of Python**, broken down as: shared `ber` package 4,185 (35 modules) + its tests 963 (12 files); `experiments/ameya` 14,312 (132 files; the final chain under `model-v1/` top level + `stack/` is 8,403); `experiments/bakshi` 4,757 (41 files); `experiments/sachi` 2,477 (22 files); `scripts/` 803 (4 files). Also **3,160 lines of shell** (93 `.sh`) and **30,680 lines of Markdown** (164 files, 20,490 of them in `knowledge/`). Raw `wc -l` over `git ls-files`. Excludes the git-ignored `data_cache/`, `work/`, `output/`, `student_resource/`, `finale/` and the one uncommitted folder (`.tmp.driveupload/`). The tree has no notebooks and no lockfiles. |
| **Source files (computed)** | 246 `.py` (35 package modules, 12 package test files, 199 experiment/script files) + 93 `.sh` + 1 `.ps1`. Tests: 84 pytest functions (70 in the package, 14 in `experiments/bakshi/v7/`). Also 3 PNG figures and 3 PDFs (methodology, plans). |
| **Package / host platform** | Installable Python package **`ber` 1.0.0** (setuptools, src layout, `requires-python >= 3.11`; the pinned environment is Python 3.13) with one CLI, `python -m ber.pipeline`. It is not published to PyPI and has no publish workflow. Full-scale runs used one integration workstation (RTX 5070 Ti 12 GB, 24 cores, 31 GB RAM). Cross-encoders and the 7B model ran on rented H100 80 GB boxes. |
| **Key dependencies** | Core (`pyproject.toml`): `numpy>=2.0`, `pandas>=2.2`, `pyarrow>=15`, `numba>=0.61`. Pinned (`requirements.txt`): `xgboost==3.2.0`, `scikit-learn==1.8.0` (isotonic calibration), `rapidfuzz==3.14.6`, `scipy==1.17.0`, `lightgbm==4.7.0` (pinned but **never imported**). GPU box: `torch 2.11.0+cu128`, `transformers 5.17.0`, `peft 0.21.0`. Models (downloaded, not redistributed): multilingual-e5-small/base/large (MIT), bge-reranker-v2-m3, Qwen2.5-1.5B, Qwen2.5-7B, Qwen3-4B-Base (Apache-2.0). |
| **License** | Apache-2.0, © 2026 Ameya Borkar, Aarush Bakshi and Sachi Dhoka (`NOTICE`). `CITATION.cff` v1.0.0, released 2026-10-08. |
| **Last commit** | `491ca28`, "docs(infra): prepare the repository for public release" (2026-10-08). The first commit is `09955f4` (2026-09-25 13:01 IST). **261 commits** in total: 60 / 77 / 80 on the three competition days (25–27 Sep), 19 on 29 Sep (packaging), 23 on 3–4 Oct (finale prep) and 2 on 8 Oct (public release). |
| **Contributors (`git shortlog`)** | **Ameya Borkar** (`ameyaborkar`) **193 commits (74%)**, Aarush Bakshi (`trustdemons05`) 55 (21%), Sachi Dhoka (`ssdhoka06`) 13 (5%). By area: all 105 commits in `experiments/ameya/` are Ameya's; Ameya has 20 of 26 in `code/` and 67 of 88 in `docs/`; Bakshi has 45 of 53 in `experiments/bakshi/`. |

---

## Competition, placement & field (pinned facts, with sources)

| fact | value | evidence in the repo |
| --- | --- | --- |
| Competition | **Amazon ML Challenge 2026**, Business Entity Resolution task | `README.md` (title, line 7), `CITATION.cff`, `CHANGELOG.md` [1.0.0], `NOTICE`. The methodology document's header reads "ML Challenge 2026". |
| Dates | Leaderboard phase Fri 25 Sep to Sun 27 Sep 2026 (IST). The ZIP was final on 29 Sep at 04:09. The **Grand Finale was Wed 7 Oct 2026**: virtual, a 10-minute talk plus 5 minutes of Q&A with senior Amazon scientists. | `docs/ROADMAP.md`, `knowledge/timeline.md` (summary §8 and the 3 Oct 22:45 row) |
| Field size | **"more than 32,000 teams"** | `README.md` lines 8 and 63. `knowledge/numbers.md` ("★ Final standing", sourced to the organisers' e-mail as relayed by Ameya on 3 Oct). `knowledge/story.md` P6. |
| "90,000+ registrations"? | **Not mentioned anywhere.** A grep of the committed tree (and of the git-ignored private `finale/` notes) finds no registration count. 32,000+ teams is the only field-size figure. | n/a |
| Finalist standing | Top 10 finalist, **"2nd on the list"**. Finalists were ranked by private leaderboard first, then blocking strategy and ML novelty, then candidate efficiency (fewer pairs per record ranks higher). | `knowledge/numbers.md` (Final standing; "How finalists were ranked"), `docs/ROADMAP.md` |
| **Final placement** | **3rd place at the Grand Finale (7 Oct 2026)** | `README.md` lines 8 and 63, `CHANGELOG.md` [1.0.0] "Result", `AGENTS.md` line 25, `docs/ROADMAP.md` Phase 6, `CONTRIBUTING.md`, `knowledge/README.md` |
| Prize | **Not recorded** anywhere in the repo | n/a |
| Metric & final score | Macro **F0.5** per Source-1 entity (an S1 with no true match scores 1 only if the prediction is empty). Composite B scored **0.990879 on the public leaderboard** (public rank 16 at upload). The private leaderboard was published as **rankings only**, so no private score exists. | `README.md` Results, `CHANGELOG.md` Submissions row "2026-09-27 #04", `knowledge/numbers.md` |

The 3rd-place claim appears only in files written on 8 Oct. The organisers' announcement is not in the tree (`finale/` is private and git-ignored), and `knowledge/numbers.md`, last curated on 4 Oct, stops at "2nd of the Top 10". The two are different rankings: the pre-finale list ordered mainly by private leaderboard, and the post-jury placement. They don't contradict each other.

## What it actually is (be concrete)

The task: for each **Source-1 (S1)** business, list all **Source-2/3** records of the same business. The only evidence is names and addresses. An S1 has 0–11 matches (3.46 on average) and about 5.6% have none. The test set hides **look-alike decoys** (a nudged house number, a swapped business word) and adds **France**, which never appears in train. The metric is **macro F0.5**, where one false merge costs as much as four missed copies, so the whole design is precision-first.

The repo has **two code layers**, and keeping them apart is the main thing to understand:

1. **`code/business_entity_resolution/` (the `ber` package):** the shared, tested foundation, with a 9-stage CLI (`records → normalize → block → features → train → predict → decide → write → evaluate`), exact TSV writers, integer entity ids, the official metric, the hash-based holdout and the paired-bootstrap gates. Its **blocking stage (Ameya's) is what the final submission uses**. Its `normalize`, string-`features` and `model` stages (Bakshi's and Sachi's, from day 1) are implemented and tested but **are not part of the final chain**.
2. **`experiments/ameya/model-v1/`:** the **final model chain**, run as standalone scripts against `ber` artifacts: features, XGBoost stages 0–3, cross-encoders, French self-training, the expected-F0.5 decision and the France rules. `RECIPE.md` holds its exact commands. Bakshi's `experiments/bakshi/box/` adds the Qwen2.5-7B training, the 7B re-check and the per-country composition that produced Composite B.

The rest of the repository is process record: plans, 16 gate-decision records, stage contracts, a changelog with every leaderboard upload, and `knowledge/`. That knowledge base has a story, a timeline, 247 decisions, a 329-entry experiment ledger, a sourced fact sheet, 82 recorded conflicts, failures, lessons, a 100-question jury Q&A bank, 11 theory pages with a glossary, and 18 foundations pages. All of it was written to prepare for the Grand Finale.

## Architecture & how it's structured

Annotated tree of the committed files (git-ignored data/work folders omitted):

```
GrenukeAmazon/
├─ README.md · CHANGELOG.md · FEATURES.md · AGENTS.md   (+ CLAUDE.md, GEMINI.md, .cursor/, .github/copilot-instructions.md → AGENTS.md)
├─ CITATION.cff · LICENSE (Apache-2.0) · NOTICE · SECURITY.md · CODE_OF_CONDUCT.md · CONTRIBUTING.md
├─ code/business_entity_resolution/          the `ber` package (pyproject, requirements, README)
│  ├─ src/ber/
│  │  ├─ pipeline.py     CLI: STAGES dict, --stage/--split/--tag/--in kind=tag/--set k=v/--folds
│  │  ├─ config.py       RunConfig (seed 2026; artifact kinds norm/candidates/features/models/scores/matches)
│  │  ├─ records.py · io.py · ids.py · paths.py · artifacts.py · outputs.py
│  │  │                  cached records, safe TSV I/O, eid = source·1e9 + number, Parquet with provenance, exact writers
│  │  ├─ block/          [Ameya] __init__ (views, union/trim, run) · index.py (int64 token + compound keys, per-country IDF)
│  │  │                  · search.py (numba exact top-k) · indic.py (9-script transliteration) · repair.py (domain/OCR)
│  │  │                  · text.py (tokenizer, legal/street lexicons, French department→region)
│  │  ├─ normalize/      [Bakshi] C3 name/address normalisation + lexicons (not in the final chain)
│  │  ├─ features/       [Bakshi] string.py/tokens.py/numbers.py · [Ameya] context.py (retrieval/rivalry) · rule_edits.py (v7)
│  │  ├─ model/          [Sachi] stage1.py (XGBoost OOF) · calibrate.py (isotonic) · decision.py (argmax + expected-F DP, G6)
│  │  └─ eval/           metric.py (macro F0.5, pair recall, oracle) · splits.py (splitmix64 folds) · gates.py · evaluate.py
│  └─ tests/             12 files, 70 test functions
├─ experiments/
│  ├─ ameya/model-v1/    THE FINAL MODEL CHAIN
│  │  ├─ feats.py feats_nx.py feats_lo.py feats_lo_proxy.py feats_legal.py lo_mix.py legal.py   pair-feature groups
│  │  ├─ s1.py (stages 0+1) · s2.py + cluster.py (stage 2 + isotonic + --pseudo) · stage3.py
│  │  ├─ ce.py · ce_box.py · ce_llm_st.py · ce_import.py · zmean_ce.py     cross-encoders on the uncertain band
│  │  ├─ pseudo_labels.py · decide.py · cands_final.py · post_ops.py · acr_join.py · rules.py
│  │  ├─ stack/          stacked decision layer (-dpc), dp_france.py, apply_swapsim.py, compose.py
│  │  ├─ pipeline/asrun/ + pipeline/box_runs/   as-run drivers of the final submissions; rented-box scripts
│  │  └─ RECIPE.md · ANALYSIS_v2–v4.md · RESEARCH_v5–v6.md · FEATURES.md
│  ├─ ameya/final-zip/doc/   submitted methodology document (MD + PDF), make_figures.py, build_pdf.py, 3 figures
│  ├─ ameya/{baseline,block-v0,plan-checks}/   day-1 baseline, blocking miss analysis, plan data checks
│  ├─ bakshi/box/        llm_group.py (7B LoRA per GPU), score_pairs.py, compose_tsv.py, compositeB.sh, analysis/, ops/
│  ├─ bakshi/final-package/   reproduce.sh (VARIANT=compositeB default), audits (*.json), make_package.py, methodology notes
│  ├─ bakshi/v7/         France rule probes + 3 test files
│  └─ sachi/             ce_llm.py (Qwen2.5 LoRA classifier), synth_fr.py, loco_*.py, model/ prototypes
├─ knowledge/            story, timeline, decisions/ (14 areas), experiments, numbers, conflicts, failures, lessons,
│                        qa.md (100 Q), components/ (14 pages), theory/ (11 pages + glossary) + foundations/ (F01–F18)
├─ plans/                FINAL_PLAN.md (+PDF), DECISION.md, ameya/PLAN.md (Plan A), sachi/PLAN.md (Plan B), bakshi/*
├─ docs/                 DEVELOPMENT.md · CONTRACTS.md (C0–C10) · ROADMAP.md · TEAM.md · decisions/ (16 gate records)
│                        · package/ (reproduce_v5all.sh, reproduce_v6all.sh, stale README/Documentation drafts)
├─ submissions/README.md upload protocol (per-upload records were removed for privacy)
├─ scripts/              setup.sh/.ps1 · new_doc.py · md_to_pdf.py · build_book.py (KB → study-book PDF) · kb/digest_transcripts.py
└─ .githooks/ (pre-commit, commit-msg, pre-push) · .github/workflows/checks.yml
```

**Entry points:** `python -m ber.pipeline` (`code/.../src/ber/pipeline.py`). Final chain: the scripts in `experiments/ameya/model-v1/` in `RECIPE.md` order. Submission driver: `experiments/bakshi/final-package/reproduce.sh`, which calls `experiments/bakshi/box/compositeB.sh`.

## Code walkthrough (the important parts)

**`ber/eval/splits.py`: a holdout that can't drift.** `fold(eid) = splitmix64(eid) mod 20`, in pure uint64 arithmetic, so it is identical on every machine. Folds 0–4 are the **25% holdout** (549,699 US/India S1, about 1.90M true pairs). Folds 5–19 form three out-of-fold groups (5–9, 10–14, 15–19). A dev sample (folds 0/5/10/15, 1 in 4) let teammates work on 8–16 GB laptops. `stage1.train_oof_and_full` raises an error if handed holdout folds, and `feature_columns` rejects any column whose name contains "country".

**`ber/eval/metric.py` + `gates.py`.** The official metric is `F0.5 = 1.25·c / (0.25·|T| + |P|)` per S1, with the singleton rule (empty truth scores 1 only for an empty prediction), averaged over every S1. `paired_bootstrap` resamples per-S1 F deltas 1,000 times and keeps a change only if its 95% interval clears zero (the original gate bar was Δ ≥ +0.002).

**`ber/block/` (Ameya): per-country multi-view retrieval.** Records are split by **exact country label** (an empty label joins every partition), so IDF and neighbours never mix countries and France gets its own statistics. `index.py` turns each record into namespaced int64 keys: name tokens, joined name, consonant skeletons, address words and numbers, plus **compound keys** (house number × street word, number × name token, pairs among the first 4 name tokens, and optional name-word × address-word). `search.py` is a numba **exact top-k cosine** over binary sqrt(IDF)-weighted token vectors. Candidates are seeded only through the query's rare tokens (`seed_cap`), then the best `verify` candidates are re-scored exactly, common tokens included. The search runs **both directions** (each S1's top `k_s1=40`, each record's top `k_r=8`) and keeps the union of the S1's top 15 and the record's top 4 (the final recipe's `trim_s1=15, trim_r=4`). A second **`name_short` view** matches names only, with character 4-grams, for records whose address has ≤ 3 tokens or whose name is a single ≥ 8-letter token (domains and handles). `repair.py` segments domains/handles into the country's S1 vocabulary with a DP ("bluegrill.com" → blue, grill) and maps OCR digits to letters ("capita1"). `indic.py` transliterates nine Indic scripts with one offset table. The final chain also uses a learned **Indic→Latin dictionary** (693 entries) fitted on train folds 5–19 only. Output on test: **58,437,794 pairs (33.73 per S1)**, holdout retrieval recall **0.99135** (v0 0.9752 → v3 0.99135).

**`model-v1/feats*.py`: pair features.** Name fuzzy ratios (rapidfuzz), IDF cosine, typo-tolerant unexplained-token mass, skeleton overlaps, legal-form agreement (`lg`), signed house-number relations (`nx`: nudges, digit swaps, suffixes), street/city/state overlap, and retrieval ranks and rivalry log-counts (`ber/features/context.py`). Rivalry uses log-counts rather than rates so that France's smaller S1 pool doesn't look anomalous. `feats_lo.py` learns look-alike **word odds** from labels. `feats_lo_proxy.py` replaces them with a **label-free** proxy for unlabelled countries: the share of moved house numbers among close pairs where a token is the extra word, isotonically mapped onto the label-based odds.

**`s1.py`: stages 0 and 1.** Stage 0 is a small XGBoost (depth 6) trained on a 10% S1 sample, with `tau0` set to keep 99.95% of positives. It removes about 85% of pairs (33.73 → 4.77 per test S1). Stage 1 (depth 9, eta 0.06, GPU `hist`) is trained out-of-fold over the three groups. With `--all`, the holdout becomes a fourth group: four models, each trained on 75% of train.

**`cands_final.py`: the candidate file.** The file holds the pairs stage 2 actually sees, cut to **p1 ≥ 0.02 and among the record's top-2 S1**, plus acronym-join pairs. That gives **6,410,308 pairs = 3.70 per test S1** (versus 4.68 uncut), because the organisers ranked finalists partly on candidate efficiency.

**`ce.py` / `ce_box.py` / `ce_llm_st.py`: cross-encoders on the band only.** Only pairs with **0.02 ≤ p1 ≤ 0.99** are read: **1.49M test pairs**, about 2.6% of retrieval. The input text is `"<name> ; <address>"` for each side, S1 first, truncated to `MAX_LEN = 96`. Three OOF models are trained per encoder. `zmean_ce.py` averages several encoders' **z-scored logits into one stage-2 feature**, because separate features made stage 2 extrapolate where the encoders disagree (four times as often in France). `ce_llm_st.py` wraps Sachi's Qwen2.5 LoRA sequence classifier (`experiments/sachi/ce_llm.py`: LoRA on all attention and MLP projections, `lora_alpha = 2r`). Band AUCs on the labelled holdout band: stage-1 p1 alone 0.930, e5-large 0.939, Qwen2.5-1.5B 0.938, bge-reranker 0.942, Qwen2.5-7B 0.944.

**`s2.py` + `cluster.py` + `stage3.py`: the collective model.** Stage 2 adds rivalry features over the **full candidate graph**: best and second-best p1 per record, margin to the best rival, per-S1 sums and confident counts, and source balance. It also adds **cluster support**, i.e. whether the record resembles the S1's other confident records (numba IDF-Jaccard), plus the cross-encoder mean. Its output is isotonic-calibrated on training-fold OOF scores only. With `--pseudo`, French test rows join training **cross-fitted by S1 thirds**, so no French pair is scored by a model trained on its own pseudo-label. `stage3.py` re-scores contested records (≥ 2 S1 with pc ≥ 0.01) using each rival S1's confident-copy count per source, and rescales the probability mass of empty-address records (true 97.7% of the time).

**`pseudo_labels.py`: guarded self-training.** Positives are final matches with pc ≥ **0.9** (or rule/acronym adds). Negatives are rejected pairs with pc ≤ **0.05** (or rule drops). Everything else is unlabelled. Where rules decided a population, the rules win. Round 1 added **+0.000458** on the public leaderboard. In leave-one-country-out tests, later rounds degraded (0.882 → 0.851 → 0.831 → 0.823), and a round 3 scored −46e-6.

**`decide.py` + `stack/`: decide the way the metric scores.** Ownership comes first: each record stays only under its highest-probability S1. Then, per S1, candidates are sorted by `q = sigmoid(logit(p) + shift)`. The best set is a prefix, and its expected F0.5 is computed exactly under a Poisson-binomial count of the true matches among the other candidates. The tuned stacked layer uses shift +0.2, a further −0.3 when ≥ 4 S1 compete for the record, and a 0.01 "phantom" expected match outside the candidate set. The combined layer gained **+48.1e-6 [+7.1, +91.2]** on the holdout. The DP alone was +33.2e-6 and not significant. (`ber/model/decision.py` holds Sachi's earlier `_expected_f_best_k` formulation and the G6 threshold-vs-DP gate.)

**`post_ops.py` + `acr_join.py`: generator-aware France rules.** Holdout labels showed the data generator edits S1s in a few fixed ways. A word swapped in place at the same address (**op-B**) is a look-alike, only **0.6% true**, and the model accepted about 17k of these in France. Dropping one word and appending a list word (A), appending only (APP), acronyms (ACR), number edits other than look-alike nudges (NUM) and typo'd codes (CODE) are **98–99.9% true copies**. For countries without labels the script drops op-B predictions and adds missed true-copy edits that are already candidates. `acr_join.py` recovers acronym copies ("PU" for "Passion Union") at the S1's address through a join on (country, house number, street word), because blocking can miss them on crowded streets. All lexicons are hand-written; the code uses no external data.

**`experiments/bakshi/box/`: the 7B and Composite B (Bakshi).** `llm_group.py` trains one OOF group of the Qwen2.5-7B LoRA classifier per GPU. It replays the sequential script's RNG draws so the three-GPU run matches the one-GPU recipe, and it checkpoints atomically on interruptible boxes. The **7B re-check** re-reads the predictions with p1 > 0.99 (94.5% of final predictions, which no cross-encoder ever saw) and drops those scored **below logit −6**, a cut-off fixed on labelled data first. On labelled data only 8.3% of such rejects are real matches. `compose_tsv.py` composes per country: **US/India from `g1w`** (the 7B counted twice in the encoder mean; holdout 0.991323), **France from `mixmdp`**, minus the 7B drops (about **840 French + 310 US/India**).

## The pipeline, numbers & leaderboard history

**Funnel (from `knowledge/numbers.md`, measured):**

| stage | pairs per test S1 | holdout pair recall | holdout oracle F0.5 |
| --- | --- | --- | --- |
| retrieval (blocking v3) | 33.73 (58.4M pairs) | 0.99135 | 0.99743 |
| after stage 0 | 4.77 | 0.99079 | 0.99725 |
| stage-2 input | 4.65 | 0.99078 | 0.99724 |
| candidate cut (the submitted file) | **3.70** (6,410,308 pairs) | **0.98354** | 0.99499 |
| final matches | 3.378 | (holdout precision 99.9%, recall 97.5%) | |

**Public leaderboard progression** (`CHANGELOG.md` Submissions, `knowledge/story.md` §4):

| upload | public LB | local holdout | rank at upload | what changed |
| --- | --- | --- | --- | --- |
| v2 (25 Sep) | 0.97608 | 0.98436 | – | blocking v1, look-alike odds, cluster support |
| v3 (25 Sep) | 0.97961 | 0.98882 | – | legal-form features. The gap from holdout to LB pointed at France (~0.93). |
| v5all | 0.98781 | 0.990156 | 15 | proxy odds, e5-small CE, France rules v2, 3.70 cut |
| v6all | 0.988609 | 0.990842 | 15 | blocking repairs, signed numbers, stage 3, rules v3 |
| v7n | 0.989721 | 0.991211 | 8 | mean of two e5-large cross-encoders (+0.001112) |
| v7nst | 0.990179 | 0.991194 | 7 | French self-training round 1 (+0.000458) |
| v7sq-dpc | 0.990545 | 0.991246 | 7 | Qwen2.5-1.5B + self-trained e5-large, stacked decision |
| mixmdp | 0.990699 | 0.991307 | 12 | guarded round-2 labels, France decision layer |
| **Composite B** | **0.990879** | g1w 0.991323 | 16 | g1w US/India + mixmdp France − 7B rejects (**team best; the submitted ZIP**) |
| B7 (last upload) | 0.990875 | – | – | B + deeper French 7B drops: a tie |

The project made 13 scored uploads in total. Public rank at upload fell from 7 to 16 even as the score rose, because other teams kept improving. The final ranking used private data the team never saw.

## Tech stack & tooling

- **Language/data:** Python 3.11+ (pinned 3.13), numpy/pandas/pyarrow (Parquet artifacts under `work/<kind>/<tag>/<split>.parquet`, each with command and git-commit provenance), **numba** `@njit`/`prange` kernels for search, CSR building, cluster support and the DP. The rule in `AGENTS.md` is "never write Python loops over pairs".
- **Models:** XGBoost 3.2 (`hist`, CUDA when available), scikit-learn isotonic regression, rapidfuzz. On the GPU box: PyTorch 2.11 + transformers 5.17 + PEFT 0.21. Every model is MIT or Apache-2.0 with at most 8B parameters, as the challenge required.
- **Compute:** the integration workstation ran all XGBoost stages (stages 1 and 2 peak at about 18–19 GB RAM). Rented H100 80 GB boxes ran the cross-encoders (e5-large about 61 min) and the 7B (three GPUs, about 2 h).
- **Process tooling:** `.githooks/` blocks commits on `main`, data paths, files over 5 MB, secrets and attribution lines, and enforces Conventional Commits. CI (`.github/workflows/checks.yml`) runs a policy job (attribution lines, data/binary files, a repo-wide IP/e-mail/SSH/personal-path scan) and a unit-test job on Python 3.12 that installs only the core dependencies, byte-compiles the sources and runs `pytest`. `scripts/new_doc.py` scaffolds decision, handover and status docs. `scripts/build_book.py` compiles the knowledge base into a study-book PDF.
- **Agents:** `AGENTS.md` is the single rule file for Claude Code, Codex, Cursor, Copilot and Gemini. The knowledge base records that much of the build ran through AI coding-agent sessions directed by each member.

## How to reproduce / run (actual commands)

The data isn't included. The challenge dataset and validator go in `student_resource/dataset/{train,test}/`, or set `BER_DATA_DIR`.

```bash
# setup + unit tests (README)
bash scripts/setup.sh --venv
pip install -r code/business_entity_resolution/requirements.txt
pip install -e code/business_entity_resolution
python -m pytest code/business_entity_resolution/tests -q

# shared-package stages (examples from pipeline.py / RECIPE.md)
python -m ber.pipeline --stage records --split train
BP="--set trim_s1=15 --set trim_r=4 --set indic_dict=ameya-fx1 --set ns_ngrams=4 --set ns_domain_len=8 --set nw_words=6"
python -m ber.pipeline --stage block --split test --tag ameya-block-v3 $BP

# final XGBoost chain v6all (RECIPE.md; ~3 h on the workstation), from experiments/ameya/model-v1:
python feats.py --cands ameya-block-v3 --split train --tag ameya-fx5        # + feats_nx / feats_lo / feats_legal / feats_lo_proxy / lo_mix
python s1.py --feats ameya-fx5 --tag ameya-s1-v6all --groups str,cx,lo0,lg,nx --drop leg__r_only_bits,leg__s1_only_bits --all
python ce.py --feats ameya-fx5 --s1 ameya-s1-v6all --tag ameya-ce-v6 --model intfloat/multilingual-e5-small
python s2.py --feats ameya-fx5 --s1 ameya-s1-v6all --tag ameya-s2-v6all --groups str,cx,lo0,lg,ce,nx --cluster --extra ... --all
python decide.py --scores ameya-s2-v6all --col pc --tag ameya-model-v6all-c2 --p-cand 0.02 --top-r 2
python cands_final.py --s1 ameya-s1-v6all --tag ameya-cands-v6all-c2 --p-cand 0.02 --top-r 2
python post_ops.py --matches ameya-model-v6all-c2 --scores ameya-s2-v6all --cands ameya-cands-v6all-c2 --feats ameya-fx5 --tag ameya-model-v6all-c2-ops
python -m ber.pipeline --stage write --split test --tag ameya-model-v6all-c2-ops --in candidates=ameya-cands-v6all-c2 --in matches=ameya-model-v6all-c2-ops

# the submitted Composite B (ZIP layout; repo source is experiments/bakshi/final-package/reproduce.sh → box/compositeB.sh)
MODEL_DIR="$PWD/code/business_entity_resolution/src/model_v1" bash code/business_entity_resolution/src/reproduce.sh   # VARIANT=compositeB default
```

Composite B needs the e5/bge/Qwen cross-encoders and the 7B trained on GPUs (three 80 GB GPUs, about 2 h for the 7B). Reruns are not bit-identical: GPU nondeterminism, and stage 3 differs on about 232k test pc values (about 500 decisions flip) across machines. The methodology says to expect results within about 0.0001.

## Tests & evaluation

- **Unit tests:** 12 files and 70 test functions in `code/business_entity_resolution/tests/` (`test_block` 19, `test_metric` 7, `test_folds_gates` 7, `test_rule_edits` 7, `test_pipeline` 6, `test_io` 5, and others). They lock the fold hash values, the metric's singleton rule, the exact TSV formats, the block union/trim logic and the expected-F DP. `test_features_v0` uses `importorskip` for rapidfuzz/sklearn, so it skips in CI's core-only install. `experiments/bakshi/v7/` adds 3 files and 14 tests for the France rule probes. The final model chain in `experiments/ameya/model-v1/` has **no unit tests**. The tests were not run for this doc (read-only review).
- **Evaluation methodology (the real test harness):** a fixed hash holdout that nothing trains on, plus a paired bootstrap for every component (16 written gate decisions in `docs/decisions/`; late rules also had to gain on both halves of the holdout). Leave-one-country-out runs stood in for an unseen country, alongside label-free French diagnostics. Uploads that changed France alone made the **public leaderboard the only French measurement**. `experiments/bakshi/final-package/audit_matching.py` and the `audit_*.json` files are strict output audits of the candidate shipments.

## Status, completeness & notable gaps

- **Archived and complete.** `Development Status :: 7 - Inactive`. The competition ended on 27 Sep and the ZIP went in on 29 Sep. The repo went public on 8 Oct with Apache-2.0, a NOTICE, a citation file and a security policy.
- **The final system lives in experiment scripts, not in the package.** `python -m ber.pipeline --stage all` runs Sachi's day-1 `ber.model` (stage-1 XGBoost + threshold/DP), not the submitted chain. The submission comes from `RECIPE.md` plus Bakshi's `compositeB.sh`.
- **Unused team code.** Bakshi's normalisation and string features and Sachi's `ber.model` were delivered and tested but not used by the final chain. `knowledge/story.md` §6 records that the reasons aren't documented.
- **Reproducibility limits.** The data isn't included, rented GPUs are required for the cross-encoders and the 7B, and results are not bit-identical. `reproduce.sh` lists two variants (`v7sb`, `v7ensall2`) as "recognised but NOT implemented".
- **Privacy pruning left dangling links.** The 8 Oct cleanup (`5696cd6`, `491ca28`) removed `finale/`, `knowledge/people/`, `docs/status/`, `docs/handover/2026-*`, `submissions/records/*` and the organisers' bundle. Many `knowledge/` pages and `CHANGELOG.md` still link to those paths. That includes the source cited for "Top 10 of 32,000+ teams" (`finale/README.md`).
- **Open items the team itself records:** whether the private ranking used Composite B or B7 (a tie on the public LB), the exact close time and upload quota, and total rented-compute spend. France's F0.5 was never measured directly (estimated at about 0.93 early and about 0.98 at the end).

## Who built what (git log + knowledge base)

- **Ameya Borkar** (captain, coordinator, integrator; 193 commits) set up the repo, hooks, CI, contracts and the `ber` foundation (io, ids, metric, splits, gates, pipeline CLI). Ameya wrote Plan A and chose the final plan, and built **blocking v0–v3** and the context features. Ameya also built the **entire final model chain** in `experiments/ameya/model-v1/`: features, XGBoost stages 0–3, calibration, the expected-F0.5 decision, the France work (proxy odds, generator rules, acronym join, France decision layer), the cross-encoders on rented GPUs, French self-training and the stacked rules. As captain, Ameya made the uploads, assembled the ZIP, wrote the methodology document and led the knowledge base. The knowledge base says plainly that this work ran through Claude Code agent sessions ("agent for Ameya", 22 sub-agents) directed by Ameya.
- **Aarush Bakshi** (55 commits) did normalisation and string features, then on the last day rebuilt the pipeline on rented GPUs. Bakshi trained the **Qwen2.5-7B** model, built `g1w`, proposed and validated the **7B re-check**, and built **Composite B**, B7, the hash-gated ZIP builder and the runnable `compositeB.sh` driver.
- **Sachi Dhoka** (13 commits) wrote Plan B, which supplied the evidence tags, the gate system, the "ties go to the simpler option" rule and the leave-one-country-out idea. Sachi built `ber.model` with the first gate records, plus the **Qwen LoRA classifier** (`ce_llm.py`) that is the training core of both Qwen models in the submission, and the synthetic French pairs.

## README vs. code

Spot-checks that **matched**: 33.7 pairs per test S1 (58,437,794 pairs); stage-0 output of 4.77 per S1; 3.70 per S1 (6,410,308 pairs); 3.38 matches per S1; cross-encoders on "~2.6%" of pairs (1.49M / 58.4M); band limits 0.02–0.99 (`ce.py` `LO, HI`); pseudo-label cut-offs 0.9/0.05 (`pseudo_labels.py`); LoRA alpha = 2r; the 7B logit −6 cut (`compositeB.sh`); the expected-F0.5 set decision (`decide.py`, `stack/core.py`); one owner per record (argmax); the 549,699-S1 holdout; Apache-2.0 and the ≤ 8B MIT/Apache model rule; and the leaderboard progression (identical in README, CHANGELOG and the KB). Concrete discrepancies:

1. **Record count.** README (and `AGENTS.md`, `docs/DEVELOPMENT.md`, the `ids.py` docstring) says **23.7M records**. `knowledge/numbers.md` measures the files at **24,229,173 (~24.2M)** and logs "23.7M" as a team typo (CF-41).
2. **Candidate recall in the submitted methodology.** `experiments/ameya/final-zip/doc/Documentation_template.md` §1/§3 says the 3.70-per-S1 candidate set keeps "99.1% of the true holdout pairs". That figure is **retrieval** recall; the candidate file keeps **98.35%** (31,304 of 1,901,267 true pairs fall outside it). The README labels 0.99135 correctly as retrieval recall, and the KB corrects the document, which stays as submitted.
3. **Stage 0.** The methodology says stage 0 "removes about 80% of pairs". The measured figure is about 85% (33.73 → 4.77 is −86%) (CF-22).
4. **The 7B re-check isn't France-only.** README's pipeline diagram and the methodology describe 7B drops of French predictions (840). `compositeB.sh` also drops US/India rejects (`usin_scored_*`), about **310 pairs**, as `CHANGELOG.md` records.
5. **README's reproduce scripts don't rebuild the final submission.** `docs/package/reproduce_v6all.sh` / `reproduce_v5all.sh` regenerate the 0.988609 / 0.98781 uploads. Composite B (0.990879) is rebuilt by `experiments/bakshi/final-package/reproduce.sh` → `box/compositeB.sh`, which README never names.
6. **The package README is stale.** `code/business_entity_resolution/README.md`'s package map still labels `ber.normalize`, `ber.block`, `ber.features` and `ber.model` as "a stub"/"stubs". All four are implemented (no `NotImplementedError` remains; `block/` alone is about 1,185 lines). The same README's "Reproduce" section implies `--stage all` regenerates the submission outputs, which isn't the case (see Status).
7. **Stale drafts ship in `docs/package/`.** `Documentation_template.md` is Sachi's "DRAFT v0.1" with TODOs (including "third member's full name"), and `README.md` has "sha256 … TODO". The real submitted document is the one under `experiments/ameya/final-zip/doc/`.
8. **Unused pinned dependency:** `lightgbm==4.7.0` is in `requirements.txt` but nothing imports it.
9. **"Every component shipped only after a paired-bootstrap gate"** is true in spirit, but the plan's +0.002 gate bar was never met after v3. It was relaxed to "interval above 0, both halves" with no decision record (`knowledge/story.md` §6), and France components were chosen on leaderboard evidence because no French labels exist.
10. **Evidence for the headline result isn't in the tree.** "3rd place at the Grand Finale" and "32,000+ teams" are stated in README/CHANGELOG/KB, but the organisers' e-mails and the finale material were moved to a private, git-ignored `finale/`. Links to it from `knowledge/` now 404. No document mentions a prize or a registration count (no "90,000+ registrations").

---

*Doc generated 2026-10-12 by reading the source at C:\Users\ameya\Documents\GrenukeAmazon. Source of truth: https://github.com/AmeyaBorkar/Grenuke*
