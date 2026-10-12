# harness-variance

> A statistics-first research codebase asking whether **agent-benchmark harness effects are additive** (they shift every model's score together, so rankings survive) **or interactive** (they reorder models). It runs no agents. It re-analyses public evaluation logs: the Harbor Hub Terminal-Bench 2.1 run database (351,644 trials, up to 98 rollouts per model × harness × task) and the SWE-bench/experiments leaderboard. The work combines a selection-bias gate, Bayesian beta-binomial GLMMs (brms/Stan) with a custom attenuation-corrected variance estimator, a generalizability-theory G-study/D-study (lme4), and assumption-free nonparametric rank-reversal counts. The headline, measured directly: **harness changes the gap between two models by 119% of the gap, so 11 of 26 observed model orderings (42.3%) reverse across harnesses**. Only 0–1 of those reversals are "demonstrable", and the variance share of the interaction is small (ψ_rank 4.5%). The ranking was never clear to begin with. Group 19 course research project (SY CS-AIML, Semester V, VIT Pune), targeting TMLR.

| | |
| --- | --- |
| **Repository** | https://github.com/AmeyaBorkar/harness-variance |
| **Visibility** | Private (research) |
| **Category** | ml-research |
| **Course context** | Group 19 · SY CS-AIML Div-A · Semester V, A.Y. 2026-27 · VIT Pune |
| **Primary language(s)** | Python (data acquisition, the selection gate, nonparametric analyses) + R 4.6 (Bayesian GLMMs, G-study). Markdown is the research log. |
| **Local path** | `C:\Users\ameya\Documents\EDIsem5` (checked out on branch `ameya`, identical to `master` at `80465ab`) |
| **Default branch** | `master`. Per-person working branches `ameya` and `gupta`. `origin/gupta` is one unmerged commit ahead. |
| **Lines of code (computed)** | **4,043 Python** (3,413 non-blank) + **2,688 R** (2,448 non-blank) = **6,731 LOC**, plus **4,778 lines of Markdown** (README, `CLAUDE.md`, 3 docs, 23 notes) |
| **Source files (computed)** | 16 non-empty `.py` modules (+4 empty `__init__.py`) + 18 `.R` scripts = 34 code files. 28 `.md`, 2 PDF memos, 7 archived aider-leaderboard HTML pages in `reference/`. Excludes the gitignored `data/` (~758 MB locally: 484 MB Harbor JSONL, 22 MB dates sidecar, 154 MB SWE-bench clone, ~101 MB derived CSVs, logs and cached brms fits). |
| **Data corpora** | Harbor Hub `trial` table (Terminal-Bench 2.1 slice), pulled via Supabase PostgREST. SWE-bench/experiments (Verified split, 104 submissions × 499 instances). τ²-bench is planned and not pulled. HAL traces were cut (D5). |
| **Key dependencies** | Python: `numpy`, `scipy` (sparse, stats), `statsmodels` (binomial GLM), `requests`, `pyyaml`. `pandas` is listed but never imported. R: `brms` on the **`cmdstanr`** backend (rstan does not compile under R 4.6 + Rtools45), `posterior`, `lme4`, `data.table`, `jsonlite`. `install.R` also installs `gtheory`, `mirt`, `difR`, `csranks`, `challengeR`, `glmmTMB`, `performance`, `effectsize` and `DHARMa`, none of which any analysis script references. |
| **License** | None (no `LICENSE` file). Corpus data is deliberately **never committed**. Harbor's ToS gives ownership of results to the submitters, so the repo ships reconstruction code only. |
| **Last commit** | `master`: `80465ab` "Merge ameya: step 5 respecified and computed" (2026-09-22). First commit: `1cdafca` "Scaffold: harness variance project" (2026-09-21). **73 commits on master** (7 merges), all within ~26 hours (2026-09-21 02:56 to 2026-09-22 05:14 IST). **Ameya Borkar 56 · Arnav Gupta (NotArnav03) 17.** `origin/gupta` adds `1bdeb6d` "Propose six robustness tests, none run" (Arnav Gupta, 2026-09-24), which makes 74 commits across all branches. |

---

## What it actually is (be concrete)

**The research question.** The 2026 agent-evaluation literature disagrees on whether the scaffold or "harness" (Claude Code, Codex, fast-agent, Terminus-2 and so on) interacts with the model. The project's claim is that the disagreement comes from **single-rollout evaluation**. At K = 1 rollout per (model, harness, task), the model × harness interaction and the Bernoulli rollout noise fall into the same residual. An additive fit therefore "fits well" by construction. Harbor's Terminal-Bench 2.1 slice has up to 98 rollouts per (cell, item), which makes it the first public corpus where the two can be separated.

**It is a re-analysis, not a benchmark runner.** Nothing in the repo executes an agent. The handover's "Spent" line reads **$0**. The pipeline is:

1. **Acquire** the two corpora, and normalise model and harness names. For example, `claude-4-sonnet-20250514` and `claude-sonnet-4-20250514` are folded together, nested provider prefixes are stripped, and the placeholder "model" `Undisclosed` is dropped.
2. **Gate on selection.** If a corpus is a curated leaderboard, outcome-based submission can *manufacture* interaction. `truncation.py` decides this empirically for each corpus.
3. **Find the estimable design.** Contrasts are only identified within a connected component of the bipartite model–harness graph. On Harbor that leaves **25 models × 26 harnesses with 63 of 650 cells observed (9.7% dense)**. That gives **13 interaction degrees of freedom**, 159 discriminating items and 70,099 trials. SWE-bench gives 24 × 35, 63 cells and 5 df.
4. **Estimate.** The methods are a Python fixed-effects preview, a brms beta-binomial GLMM with a custom main-effect estimator, a crossed random-effects cross-check, and an lme4 G-study/D-study. Every estimator was validated by **parameter-recovery simulation on the real design skeleton** before its numbers were quoted.
5. **Measure rankings directly, without a model.** This step counts pairwise rank reversals, builds simultaneous rank confidence sets, and splits item-level harness shifts into uniform and non-uniform parts.

**Locked decisions D1–D11 (`docs/DECISIONS.md`).** The project overturned its own starting thesis twice and kept the wreckage visible:

- D1 "large in level, small in rank" was revised by D10.
- The ψ_rank 5–11% hybrid estimate was retracted.
- D7: main effects must be **projection coefficients, not marginal means**. On a 9.7%-dense design the marginal mean showed +40% bias on known truth.
- D8: bias is corrected **analytically**, by `c = tr(PV)/(J−1)`.
- D9: convergence is gated on the **reported quantity**, not the parameter vector.
- D11: a discriminating simulation decided between two estimators that disagreed.

## Architecture & how it's structured

```
EDIsem5/  (repo: harness-variance)
├─ README.md                 headline results + reproduce steps (484 lines; doubles as the results summary)
├─ CLAUDE.md                 machine-facing conventions: data rules, estimator rules, gates, claims discipline
├─ requirements.txt          requests, numpy, pandas, pyyaml, scipy, statsmodels
├─ docs/
│  ├─ DECISIONS.md           D1–D11, superseded text kept in boxed "retracted" blocks
│  ├─ HANDOVER.md            per-step status index + "Gotchas" (each one once produced a wrong number)
│  ├─ WORKFLOW.md            branch-per-person, file ownership, "notes are findings, shared docs are indexes"
│  └─ Gate-the-Spend.pdf · Two-Designs-One-Paper.pdf   decision memo + the selection objection it answers
├─ notes/                    23 one-finding-per-file notes: results, failed predictions, retractions, licence drafts
├─ reference/aider-pages/    7 archived aider.chat leaderboard pages (background)
├─ src/                                Python
│  ├─ acquire/harbor_snapshot.py       keyset-paged PostgREST pull of Harbor `trial` (+ --dates-only sidecar, --resume)
│  ├─ acquire/swebench_pull.py         shallow-clones SWE-bench/experiments → one row per (submission, instance)
│  ├─ common/normalize.py              conservative model/harness canonicalisation + placeholder filter
│  ├─ diagnostics/truncation.py        THE GATE: within-cell failure rate, per-period minimum, pooled left tail
│  ├─ analysis/build_dataset.py        Harbor → rollouts_tb21.csv; union-find bipartite components; task@hash items
│  ├─ analysis/preliminary.py          first additive-vs-saturated fit (superseded by compare_corpora)
│  ├─ analysis/compare_corpora.py      Python preview on both corpora; sparse IRLS; item bootstrap by reweighting
│  ├─ analysis/export_design.py        estimable skeleton → design_tb21.csv (model, harness, item, n[, y]) for R
│  ├─ analysis/gstudy_audit.py         replication audit per G-study facet before fitting
│  ├─ analysis/rank_stability.py       STEP 4 (Ameya): reversals, credible flips, rollout null, cluster bootstrap
│  ├─ analysis/rank_reversal.py        STEP 4 replication (Arnav): 2×2 blocks, item-resample null, gap sweep
│  ├─ analysis/rank_sets.py            simultaneous rank confidence sets per harness
│  ├─ analysis/item_shift.py           STEP 5 respecified: uniform vs non-uniform item shift, wild cluster bootstrap
│  ├─ sim/selection_sim.py             interaction manufactured by outcome selection from exactly-additive truth
│  ├─ sim/recovery.py                  synthetic recovery generator (Arnav's route B)
│  └─ sim/harness_identifiability.py   many-replicate IRLS check of the harness main effect (unregularised; flagged unusable)
└─ analysis/                           R
   ├─ install.R · verify.R             12-package install with Windows/Rtools45 traps; verify compiles a real Stan model
   ├─ _maineffects.R                   projector(), decompose(), attenuation(), corrected_sd()  ← the estimator (D7/D8)
   ├─ _convergence.R                   check_convergence(), check_derived()                     ← the gate (D9)
   ├─ _gstudy_lib.R                    G-study formula, erho(), phi_coef(), D-study tables
   ├─ 01_recovery.R · 01b_recovery_re.R · recovery.R    step-1 parameter recovery, routes A (Ameya) and B (Arnav)
   ├─ 01c_recovery_calibrated.R        D11 discriminating test at three candidate truths
   ├─ 01d_recalib_fixed.R · 01e_recalib_random.R · 01e_recalib_isolate.R   Arnav's recalibration runs
   ├─ 01f_noise_floor_check.R          cheap test of Arnav's registered prediction against a cached fit (it failed)
   ├─ 02a_glmm_fixed.R                 STEP 2, source of reported numbers: fixed cells + random item, beta-binomial
   ├─ 02b_glmm_random.R                crossed random-effects cross-check (~1 h at 2 cores)
   ├─ 02c_combine.R                    three-route agreement; no MCMC
   ├─ 03_gstudy.R                      STEP 3: G-study/D-study, 7 crossed random effects (lme4, nAGQ = 0)
   └─ 03b_gstudy_recovery.R            recovery test of the G-study estimator
```

**Data flow:**

1. `harbor_snapshot.py` → `data/raw/harbor_trial.jsonl` (351,644 trials).
2. `build_dataset.py` → `rollouts_tb21.csv` (86,822 TB 2.1 rollouts).
3. `export_design.py --with-outcomes` → `design_tb21.csv`, the estimable component aggregated to **5,307 (cell, item) rows, 70,099 trials**.
4. That file feeds every R script and the three nonparametric Python scripts.

Results land in `data/derived/` as `.log`, `.json` and cached `.rds` posteriors. All of it is gitignored. Each script has one job, and each analysis step gets a new file rather than an edit to an old one (WORKFLOW rule).

## Code walkthrough (the important parts)

**Acquisition (`harbor_snapshot.py`, `swebench_pull.py`).**
- Harbor is read from its public Supabase PostgREST endpoint. The publishable key is hard-coded with a comment explaining that it is the one in Harbor's own Apache-2.0 client.
- Paging is **keyset** (`id=gt.<last>`), because offset paging hits a `57014 statement timeout` past ~300k rows. The puller checkpoints for `--resume`, sleeps 0.15 s between pages, and pulls only 8 columns.
- SWE-bench is a `git clone --depth 1`. Each `results/results.json` is expanded into pass/fail over the union of instances. The submission date is parsed from the `YYYYMMDD_` directory prefix, because the selection test needs period strata.

**The selection gate (`diagnostics/truncation.py`).** Three tests, in this order of authority:
1. **Within-cell failures.** "A cherry-picked corpus cannot contain its own failures." If any cell with 200+ rollouts keeps ≥10% zero-reward runs, the corpus is not outcome-selected.
2. **Per-period minimum.** If the minimum climbs over time, the submission bar moved.
3. **Pooled left-tail mass** (weakest).

The script picks year or month granularity from the date span. Result: SWE-bench is selected, Harbor is not. That split case is what makes the Harbor arm interpretable.

**The estimable design (`build_dataset.components`, `compare_corpora.estimable` / `discriminating_items`).**
- Union-find runs over `('m', model)` / `('h', harness)` nodes.
- Cells below 100 trials or 3 rollouts per item are dropped (Harbor), as are all-0 or all-1 cells. The largest component is kept.
- Items that nobody passes or nobody fails are removed, because they cause perfect separation in IRLS. That is 19 of 178 on Harbor and 41 of 499 on SWE-bench Verified.
- Items are keyed `task@content_hash`, since every TB task has exactly two versions. Dropping the minority version would delete four harnesses and 18 cells.

**Python preview (`compare_corpora.py`).** It fits additive (`item + model + harness`) and saturated (`item + cell`) binomial GLMs and g-computes model and harness SDs onto the probability scale. The interaction SD is the SD of per-cell mean residuals, and the script also reports ψ_rank = σ²_mh/(σ²_m+σ²_mh) and ψ_total. Three details matter:
- **Overdispersion φ is estimated from replicated groups only.** A singleton group has E[Pearson²] = 1 regardless of φ, and SWE-bench is 92% singletons.
- The **item cluster bootstrap reweights instead of rebuilding.** Drawing an item w times equals multiplying its (successes, trials) by w. A hand-written **sparse IRLS** (`_irls`, four non-zeros per row, warm-started from the full fit) replaced a dense design matrix that ran the machine out of memory (119 MB dense against 0.92 MB sparse for SWE-bench).
- Under D8 these SDs are **uncorrected**: inflated denominators, so ψ_rank here is a lower bound.

**The reported estimator: `02a_glmm_fixed.R` + `_maineffects.R`.**
- The model is `brm(y | trials(n) ~ 0 + cellf + (1 | itemf), family = beta_binomial())`, with `normal(0,3)` on cells and `student_t(3,0,2.5)` on the SD, run at 4 chains × 2600 (1000 warmup) on cmdstanr.
- For every posterior draw, `decompose()` projects the 63 cell effects onto the additive model + harness space by weighted least squares (`qr(X·√w)`, w = trials per cell). The model and harness **coefficients** become the main effects, with the reference level's zero restored so each factor is a full finite population. The **projection residual** is the interaction.
- `attenuation()` computes `c = (tr V − ΣV/J)/(J−1)` from the posterior covariance of the coefficient vectors. `corrected_sd()` subtracts `2c` per draw and reports the truncation fraction and noise floor √(2c).
- ψ_rank and ψ_total come from **one posterior**. The interaction is converted to percentage points by the delta method at each draw's average cell, `σ·p̄(1−p̄)`.
- The file header records why: the old `tapply(fitted, model, mean)` carried each model's ~2.5 harnesses' effects and inflated the harness SD by +40% in recovery.

**The convergence gate (`_convergence.R`).** `02a` has no global intercept, so cell and item effects sit on a ridge (cor −0.756) that ruins parameter-level Rhat/ESS. Every *reported* quantity is an SD of centred effects and is invariant to that ridge. So:
- `check_derived()` gates the derived per-draw matrix with chain structure preserved (Rhat ≤ 1.01, ESS ≥ 400).
- Divergences always block.
- Nothing is called gate-passed on fewer than 4 chains.

**Cross-checks (`02b`, `02c`).** `02b` fits `1 + (1|model) + (1|harness) + (1|cell) + (1|item)` as crossed random effects. `02c` prints three routes side by side: `02a`, `02b`, and the model-free step-4 counts read from `rank_stability.json`. It also prints an explicit "RETRACTED and not to be reinstated" block.

**Recovery tests (`01*.R`, `recovery.R`, `sim/recovery.py`, `03b`).** These simulate outcomes on the real 63-cell skeleton from known components and refit with the same estimator.
- `01c` is the D11 discriminating test. The fixed and random fits disagreed on the harness SD (0.869 vs 0.460), so it simulates at *each* claimed truth and asks which one could have produced the observed estimate.
- It also fixes a generator bug that two earlier runs shared. They drew the interaction at the target SD across all 63 cells, but the estimand is the residual after projection, which keeps only √(13/62) ≈ 46% of it.

**G-study (`_gstudy_lib.R`, `03_gstudy.R`).**
- The model is `glmer(cbind(y, fail) ~ 1 + (1|model) + (1|harness) + (1|model:harness) + (1|item) + (1|model:item) + (1|harness:item) + (1|row), binomial, nAGQ = 0)`. The row-level effect is the three-way term, which is identifiable because rows carry multiple rollouts.
- Rollout noise enters as the delta-method `1/(p̄(1−p̄))`.
- `erho()` and `phi_coef()` implement the two universes. In **A** (harness fixed) σ²_mh counts as universe score. In **B** (harness random) it is error divided by n_h.
- The script records that the pre-registered linear-probability fallback was tried first and failed: heteroscedastic residual, then a singular fit. That is why it uses a binomial GLMM.

**Step 4, no model (`rank_stability.py`).**
- For every harness pair sharing ≥2 models, and every pair of those models, accuracies are restricted to items present in **all four cells** (always 89). Then:
  - `d1 = acc(m1,h1) − acc(m2,h1)`
  - `d2 = acc(m1,h2) − acc(m2,h2)`
  - a **flip** means `sign(d1) ≠ sign(d2)`
  - a **credible flip** additionally needs both gaps' 95% item-bootstrap intervals to exclude zero.
- **The control** (`rollout_null`) assumes harness has no effect and computes each gap's beta-binomial rollout variance, `Σ n p(1−p)[1+(n−1)ρ]/(Σn)²` with ρ = 1/(φ+1). It then gives the analytic disagreement probability around the precision-weighted pooled gap.
- A **moment-matched interaction SD** with no GLMM is `√(var(DiD) − mean bootstrap var)/2`.
- A cluster bootstrap resamples harness pairs.
- Arnav Gupta's `rank_reversal.py` re-implements the step independently. It uses ≥81 shared items, a double item-resample null, a gap-threshold sweep and a same-family "sibling" test.

**Rank sets (`rank_sets.py`).** This is Mogstad-Romano-Shaikh-Wilhelm in outline. The script bootstraps items and takes **one** critical value as the 95th percentile of `max_ij |d*−d|/se` across all pairs. Then i beats j when `d > c·se`, and the rank set is `[1 + #beats-i, k − #i-beats]`. It also checks whether another harness's observed ranks fall inside these sets.

**Step 5 respecified (`item_shift.py`).** For each (harness pair, item), the uniform shift is `Δ_i = p(i|h1) − p(i|h2)`, pooled over the shared models. The non-uniform shift is a per-item model-pair DiD. Noise is subtracted with a beta-binomial variance applied **within** each cell. An earlier version pooled the `(n−1)ρ` inflation across models and "found" a corrected SD of exactly 0. The script enumerates all Rademacher sign patterns for a wild cluster bootstrap.

## Findings so far

All numbers below were checked against the saved artifacts in `data/derived/`. The 02a/02b posteriors and the D11 scenario summaries were re-read from their cached `.rds` files.

| Finding | Number | Source |
| --- | --- | --- |
| **Observed model orderings that reverse across harnesses** | **11 of 26 = 42.3%**, Wilson [25.5, 61.1]; cluster bootstrap over harness pairs [23.5, 58.3] | `rank_stability.log` / `.json` (`rank_stability.py`) |
| Gap vs change in gap | mean \|gap\| **7.0 pp**, mean \|Δgap\| **8.3 pp = 119%** of the gap | `rank_stability.json` (0.0696 / 0.0826) |
| Credible reversals (both orderings clear) | 1 of 26 = 3.8% (item bootstrap); 0 of 26 with φ = 3.70 per D10 (that run's output is not saved) | `rank_stability.log`; `docs/DECISIONS.md` D10 |
| Rollout-noise null (harness has no effect, items fixed) | **12.8%** expected disagreement (3.3 of 26) vs 42.3% observed, an excess of 29.5 pts | `rank_stability.log` |
| Reversal rate depends on the harness pair | codex \| fast-agent 5/10; monet \| codex 3/6; monet \| fast-agent 1/6 | `rank_stability.log` |
| Kendall τ is uninformative here | 0.000 / 0.667 / 0.000 on the only 3 harness pairs sharing ≥3 models | `rank_stability.json` |
| Same model, two harnesses (level shift) | 63 pairs, median 4.2 pp, mean 6.3 pp, max 32.9 pp; 21/63 exclude zero | `rank_stability.log` |
| Independent replication (Arnav Gupta) | 13/30 = 43.3% vs 12.5% null; at a ≥10 pp gap only 1/8 = 12.5% vs 3.8% | `notes/step4-two-implementations-agree.md` (`rank_reversal.py`; output not saved, not re-run) |
| **Variance decomposition (02a, gate-passed)** | model SD 0.981 [0.838, 1.139] logit ≈ 13.4 pp; harness 0.869 [0.740, 1.002]; interaction **0.211 [0.162, 0.261] = 3.96 [3.05, 4.92] pp**; item 1.518 | `02a_v3.log`, `glmm/fixed_draws_v3.rds` |
| ψ_rank / ψ_total (02a) | **4.5% [2.5, 7.2]** / 45.4% [38.3, 52.7]; P(model SD > harness SD) = 0.941, ratio 1.13 | `02a_v3.log` |
| The random-effects route disagrees ninefold | ψ_rank **41.2% [10.3, 91.6]**; cell SD 0.531 vs 02a's 0.211, despite 0 divergences, max Rhat 1.0083 and min ESS 501 | computed from `glmm/random_draws_v3.rds`; `02b_v3b.log` |
| Model-free interaction SD | 4.52 pp (1.14× the GLMM's 3.96) | `rank_stability.json` |
| D11 discriminating test (fixed estimator, harness truth → estimate) | 0.460 → 0.474 [0.302, 0.654]; 0.869 → 0.727 [0.621, 0.846]; 1.10 → 1.173 [1.036, 1.332] (scenario C's own gate = FALSE) | `recovery/recov_{A,B,C}_fixed_summary.rds` |
| **G-study headline** (89 items, 5 rollouts, 1 harness) | Eρ² **0.974** harness-fixed vs **0.618** harness-random; Φ 0.932 / 0.475 | `03_gstudy.log` |
| D-study | Eρ² ≥ 0.80 needs **3 harnesses**, ≥ 0.90 needs 7, ≥ 0.95 needs 17. One rollout reaches 96.0% of the infinite-rollout value. Infinite items on one harness top out at 0.634. | `03_gstudy.log` |
| G-study recovery (6 reps) | model × harness +2.2%; harness main effect −16.7%; Eρ² gap +11.1% | `03b_recovery.log` |
| Variance shares (logit, excluding rollout) | item 47.3%, model × item 13.9%, harness × item 11.5%, model 9.1%, m × h 5.2%, harness 3.8% | `03_gstudy.log` |
| Rank sets on `fast-agent` (11 models, 89 items) | **10 of 11 sets wider than one place**; the top model (83.7%) has set 1–6; only last place (54.0%) is pinned. 2 of 13 cross-harness placements fall outside the sets. | `rank_sets.json` |
| Item-level mechanism (step 5) | uniform shift 14.4 pp vs non-uniform 14.1 pp, ratio **0.984** against a registered prediction of 1.273 (failed); 267 / 1,958 comparisons on 3 harness pairs | `item_shift.json` |
| Selected vs unselected corpus | Harbor ψ_rank 3.6% [2.0, 5.5] vs SWE-bench 0.2% [0.1, 0.2]; difference −3.4% [−5.4, −1.9]. The prediction that selection inflates the interaction **failed** in direction. | `comparison.txt` (`compare_corpora.py`) |
| Selection gate | SWE-bench per-period minimum 0.4% → 2.8% → 23.2% → 71.9% (selected). Harbor 70.2% → 9.5% → 0.0% → 74.4%. Deepest Harbor cell: 7,190 rollouts with 19.5% zeros (not selected). | `cell_accuracy_{swebench,harbor_tb21}.csv` |
| Dead items | 41 of 499 SWE-bench Verified instances never solved by any submission (8.2%); 19 of 178 Harbor items nobody fails | `comparison.txt` |

## How to reproduce

The actual dependency order (this differs from the README; see below). Windows, Python 3, R 4.6 with Rtools45:

```bash
pip install -r requirements.txt
python src/acquire/harbor_snapshot.py               # ~8 min, 484 MB
python src/acquire/harbor_snapshot.py --dates-only  # ~3 min, 22 MB sidecar
python src/acquire/swebench_pull.py --split verified
python src/analysis/build_dataset.py                # -> rollouts_tb21.csv
python src/diagnostics/truncation.py                # the gate (pass --input/--source '' for SWE-bench)
python src/analysis/compare_corpora.py --boot 200   # Python preview, ~19 min
python src/analysis/export_design.py --with-outcomes   # -> design_tb21.csv (needed by everything below)

python src/analysis/rank_stability.py --boot 20000  # step 4, ~1 min
python src/analysis/rank_reversal.py                # step 4 replication
python src/analysis/rank_sets.py                    # rank confidence sets
python src/analysis/item_shift.py                   # step 5

Rscript analysis/install.R && Rscript analysis/verify.R   # -> "STAN COMPILE OK"
Rscript analysis/02a_glmm_fixed.R     # ~9 min, the reported numbers
Rscript analysis/02b_glmm_random.R    # ~1 h, cross-check
Rscript analysis/02c_combine.R        # three routes, no MCMC
Rscript analysis/03_gstudy.R          # ~35 s
Rscript analysis/03b_gstudy_recovery.R 6
```

The brms `file =` cache keys on data and formula, not on chains or iterations. Fit filenames are versioned (`fit_fixed_v3`) so a configuration change does not silently reload old draws. Run each brms fit as a separate process: a combined script was killed for memory twice on a 31 GB machine.

## Tests

**There are no unit tests and no CI.** There is no pytest/testthat suite and no `.github/` directory. Validation is statistical and built into the workflow:

- **Parameter recovery before real data.** `01_recovery.R`, `01b`, Arnav Gupta's `recovery.R` + `sim/recovery.py`, `01c` (three truths), `03b` (G-study, 6 reps). Two independent step-1 implementations both found the model SD inflated by about +13.5% (+13.6% / +13.3%), which led to D7/D8.
- **Convergence gates** (`_convergence.R`) that `stop()` rather than warn.
- **Duplicate implementations by design.** Step 4 was built twice (`rank_stability.py` and `rank_reversal.py`) and agrees within one point. `02c` puts three estimator routes side by side.
- **Registered predictions with falsifiers**, written into script docstrings and notes before runs. Several failed and are kept: noise-floor tripling, selection inflating interaction, the step-5 ratio of 1.273, and clustering widening intervals.
- `verify.R` compiles a real Stan model to prove the toolchain works.

## Status, completeness & notable gaps

- **Done (per `docs/HANDOVER.md`, confirmed by artifacts):**
  - the selection gate and the Python preview on both corpora
  - step 0 (R stack)
  - step 1 (recovery, plus D11 resolved)
  - step 2 (02a/02b/02c, both fits gate-passed)
  - step 3 (G-study/D-study, recovery-tested)
  - step 4 (rank stability ×2, rollout null, cluster bootstrap, rank sets)
  - step 5 (respecified as item-level shift; multi-group IRT/DIF was shown not to run on a within-subjects design)
- **Not started:**
  - **step 6**: τ²-bench replication (MIT-licensed; not pulled; "one benchmark is the biggest weakness")
  - **step 7**: the experiment, repointed by D3 to explaining the ~12 pp harness *main* effect over ~8 named design dimensions (~$325 on a preemptible H100, or free on a departmental 80 GB node)
  - no paper draft is in the repo (TMLR is the target)
- **Open items:**
  - OSF preregistration is unchecked.
  - Licence emails exist only as drafts (`notes/licence-emails.md`).
  - The SWE-bench figures still lack the D8 correction, so they must not be compared with Harbor's corrected numbers.
  - `harness_identifiability.py` is unregularised. Its saved scenario-A output has `c_harness ≈ 4.8×10⁵`, and the script flags its harness rows as unusable.
  - Planned but not built: a wild cluster bootstrap for step 4, and rank sets with harness as a facet.
  - Six proposed robustness tests (posterior predictive check, prior sensitivity on 02b, harness jackknife and others) sit on the **unmerged `gupta` branch**, none run.
- **Activity:** 73 master commits in ~26 hours (2026-09-21/22). The last activity anywhere is 2026-09-24, about three weeks before this doc.
- **Inherent limits the project states itself:** n = 26 comparisons from 7 harness pairs; one benchmark; observational data, where harnesses are identities rather than design toggles; 15 of 26 harnesses carry a single model; 13 interaction df.

## README vs. code

The headline numbers are **accurate**. 42.3% (11/26), 7.0 / 8.3 pp / 119%, the 12.8% null, cluster bootstrap [23.5, 58.3], ψ_rank 4.5% [2.5, 7.2], 02b's 41.2% [10.3, 91.6], the D11 table, every G-study/D-study figure, the rank-set claims, step 5's 14.4 / 14.1 pp and the gate figures all match the saved outputs. Discrepancies worth knowing:

1. **"8 harness pairs" vs 7.** The README headline, D10 and `rank_stability.py`'s docstring and hard-coded print text say the 26 comparisons come from 8 harness pairs. The script's own computed output says "only 7 harness pairs", and the JSON contains 7 distinct pairs. The README contradicts itself, using "7" two paragraphs later.
2. **"3 harnesses × 25 items beats 1 harness × 300 items at an eighth of the rollouts"** (README, HANDOVER, `notes/step3-gstudy.md`). The Eρ² values (0.779 vs 0.629) are right. The arithmetic is not: at 5 rollouts per item it is 375 vs 1,500 rollouts, which is **a quarter**.
3. **Scenario C is quoted despite failing its gate.** The README's D11 table gives 1.173 [1.036, 1.332] with no caveat. The saved summary has `gate = FALSE` (sd_harness Rhat 1.0108), and DECISIONS.md says "Do not quote 1.173." The conclusion does not depend on it.
4. **Python-preview diagnostics have no saved artifact.** The README table gives φ = 3.70 / 2.19 and interaction-test p = 3.1e-06 / 0.0035. The only saved output, `comparison.txt`, predates the "φ from replicated groups only" fix and shows φ = 3.86 / 1.60 and p = 6.84e-06 / 2.19e-04. The ψ figures match; the φ and p values come from an unsaved re-run.
5. **The reproduce order is broken.** The README's bash block runs `rank_stability.py` before `export_design.py --with-outcomes`, which appears later in the PowerShell block and produces its input. `rank_sets.py`, `item_shift.py`, `rank_reversal.py` and the `03` G-study scripts are not listed at all.
6. **Stale sections.** The README "Open" list still includes the D11 third scenario (done) and "G-study, D-study, simultaneous rank sets" (all done and reported above it). "Decisions locked" item 11 still calls D11 "the open blocker", although the README's own box says it is resolved. HANDOVER says `scipy`/`statsmodels` are missing from `requirements.txt`; they are present now, while `pandas` is listed and unused.
7. **"Gate passed on the parameters"** for 02a is true only of the two gated parameters (`sd_itemf`, `phi`). Parameter-wide, `02a_v3.log` shows max Rhat 1.0212 with 62 of 225 parameters above 1.01. This is by design under D9, because the cell/item ridge leaves reported quantities unaffected, but the README wording overstates it.
8. **The ground rule "pre-register before fitting anything to real data" was not followed in practice.** All real-data fits exist and OSF preregistration is still unchecked. `03_gstudy.R` openly documents departing from the pre-registered linear-probability G-study.
9. **Stale code text:**
   - `rank_stability.py` prints "step 2's GLMM gave 2.89 pp" (superseded by 3.96 pp).
   - `item_shift.py`'s docstring describes 5 harness pairs and 32 sign patterns; the run had 3 pairs and 8 patterns.
   - `harbor_snapshot.py` estimates "~70 MB"; the actual pull was 484 MB.

---

*Doc generated 2026-10-12 by reading the source at C:\Users\ameya\Documents\EDIsem5. Source of truth: https://github.com/AmeyaBorkar/harness-variance (private)*
