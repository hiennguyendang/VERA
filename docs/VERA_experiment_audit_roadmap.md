# VERA — Experiment Audit & Improvement Roadmap

*Updated: 2026-07-08. Source artifacts: `RUN/`, `LOGS/`, phase READMEs, and methodology docs.*

This note records the current experimental readout and the next planned improvements. It is intentionally operational: each item has a status and priority so the project does not drift.

## 2026-07-08 Implementation Pass

Implemented now:

- **M3 diagnostics JSON is expanded:** `phase_3/scripts/5-eval.py --diagnostics-json` now includes
  reliability bins, ECE, best-threshold sweep, per-class diagnostics, and per-concept diagnostics.
- **M3 reliability rendering:** `phase_3/scripts/plot_diagnostics.py` writes CSV summaries and simple
  SVG reliability diagrams from the diagnostics JSON.
- **M3 prediction dump + bootstrap:** `phase_3/scripts/5-eval.py --pred-dump` now writes NPZ
  probabilities/targets; `phase_3/scripts/bootstrap_metrics.py` writes bootstrap CIs from that dump.
- **M3 threshold/gate export:** `phase_3/scripts/export_thresholds.py` exports per-disease
  best-threshold JSON and a concept allow/deny gate for explanation use.
- **M3 faithfulness JSON:** `phase_3/scripts/6-faithfulness.py --diagnostics-json` now writes
  concept go/no-go, intervention summary, and per-concept intervention deltas.
- **Full Phase-3 runner:** repo-root `phase_3.sh` now runs the crosswalk patch, full training grid,
  eval diagnostics, plots, faithfulness, thresholds, concept gates, bootstrap, M5 inference JSONL,
  and M4 region-cache precompute under a fresh tag.
- **M4 diagnostics JSON is expanded:** `phase_4/scripts/3-eval.py --diagnostics-json` now writes
  confusion matrix, per-disease F1, per-region F1, and same-view/cross-view pair slicing.
- **M4 noisy-label hook:** `phase_4/scripts/2-train.py --label-smoothing <float>` is implemented
  for class-weighted CE; this is the lightweight part of the noisy silver-label plan.
- **M4 TempFuse+M3-delta hybrid:** `phase_4/scripts/2-train.py --tempfuse-input-mode feat_logits`
  now appends M3 current/prior/delta disease logits to the TempFuse region feature. This is the
  planned v4 test for "patch temporal fusion + calibrated disease evidence".
- **M4 flip-consistency KL:** `phase_4/scripts/2-train.py --flip-consistency-weight <w>` now adds
  symmetric KL between `(current, prior)` predictions and `(prior, current)` predictions after
  swapping improved/worsened. Implemented; not trained yet.
- **MS-CXR-T audit bridge:** `phase_4/scripts/5-mscxrt_audit.py` maps the local MS-CXR-T CSV to VERA
  image IDs, evaluates M4 on five human temporal findings, and reports mean/max/logsumexp
  region-to-image aggregations.
- **MS-CXR-T adapter hook:** `phase_4/scripts/6-mscxrt_adapter.py` can fine-tune head/pool/all on a
  deterministic subject-hash MS-CXR-T train split and select on hash-val. This is implemented for
  server experiments, not used as the final external audit.
- **M4 retrain matrix:** `phase_4/run_m4_retrain_matrix.sh` runs the server sweep for v3 controls,
  v4 TempFuse+M3-delta variants, regiondiff controls, optional MS-CXR-T adapters, silver eval, and
  MS-CXR-T audit after each checkpoint.
- **Full Phase-4 runner:** repo-root `phase_4.sh` now wraps all Phase-4 work: prerequisite M3
  faithfulness check, optional label rebuild, dataset stats, M3 region-cache bridge, retrain matrix,
  val/test/gold diagnostics, MS-CXR-T audit, plots, M4 inference JSONL, and optional M5 assembly.
- **M4 visualization:** `phase_4/scripts/plot_diagnostics.py` renders confusion matrices,
  per-class/per-disease SVGs, MS-CXR-T comparison bars, and train-log loss/val-F1 curves. A current
  local snapshot was generated under `artifacts/phase4_current/`.
- **M5 global grounding:** global/relational findings are marked with `grounding_type="global"` in
  findings and provenance, instead of pretending they are purely region-grounded.
- **M5 stats JSON:** `phase_5/run.py --stats-json` writes machine-readable verify stats.
- **Sequential audit runner:** root `audit.sh` runs the implemented audit steps with `local4060` or
  `h100mini` profiles.

Ran locally on gold split:

- M3 eval diagnostics: **ran**, `artifacts/diagnostics/m3_B_faithful.gold.diagnostics.json`.
- M3 reliability tables/SVG: **ran**, `artifacts/diagnostics/m3_B_faithful.gold.plots/`.
- M3 faithfulness diagnostics: **ran**, `artifacts/diagnostics/m3_B_faithful.gold.faithfulness.json`;
  concept F1 0.8615, intervention correct-direction 96%, why-faithful allowed.
- M4 diagnostics: **ran**, `artifacts/diagnostics/m4v3_tf.gold.diagnostics.json`;
  macro-F1 0.5665, change-only F1 0.6103.
- MS-CXR-T external audit: **ran**, `artifacts/diagnostics/m4v3_tf.mscxrt.diagnostics.json`.
  Local CSV rows 1,045; usable pairs 964; class counts stable 494 / improved 225 / worsened 510.
  Best aggregation was `lse`: macro-F1 0.5695, change-only F1 0.6463.
- M5 smoke demo: **ran**, verify stats clean on synthetic examples.

Not run yet:

- Full `test` diagnostics after these new JSON/plot additions.
- M4 training with `--label-smoothing`; implemented but not trained.
- M4 v4 hybrid training with `--tempfuse-input-mode feat_logits`; implemented but not trained.
- M4 flip-consistency KL training with `--flip-consistency-weight`; implemented but not trained.
- MS-CXR-T adapter training; implemented but not run.
- Full Phase-4 rerun with `bash phase_4.sh --profile h100mini --tag xwalk_v2`; implemented but not run.
- M5 real report audit from `m3_pred.jsonl`/`m4_pred.jsonl`; `phase_5/run.py --stats-json` is ready,
  but real prediction JSONL files were not present at the default paths.
- Full post-crosswalk Phase-3 rerun with `bash phase_3.sh --profile h100mini --tag xwalk_v2`;
  implemented but not run.

Still blocked / not implementable by code alone:

- Reader study.
- CheXplus/NIH ablation, bootstrap confidence intervals, deletion/insertion, and ordinal/pairwise
  M4 objectives: these need either full reruns or a larger design pass.

## Priority Legend

- **P0:** required for the main scientific claim.
- **P1:** high-value improvement or reviewer-risk reducer.
- **P2:** useful ablation/engineering polish.
- **P3:** optional extension or future-work branch.

## Current Verdict

M3 is close to shippable. The best default is **`m3_B_faithful`**: detector boxes, bbox-masked attention pooling, global head on, faithful masked non-negative concept-to-disease head. It keeps almost the same image performance as free MLP while giving a structural 100% concept-intervention pass.

M4 is the main unresolved scientific risk. Current silver-test performance is only moderate, and the evaluation labels are still NLP-derived `comparison_cues`, not a human temporal test set. Any temporal-faithful claim remains provisional until B2 is resolved.

## M3 Audit Done From Existing Artifacts

| Run | Test F1 | Test AUC | Region F1 | Concept F1 | Faithfulness read |
|---|---:|---:|---:|---:|---|
| `m3_A` | 0.8783 | 0.8300 | 0.8620 | — | where-faithful only |
| `m3_B` | 0.8842 | 0.8308 | 0.8637 | 0.8986 | intervention 85%; not structurally guaranteed |
| **`m3_B_faithful`** | **0.8831** | **0.8293** | **0.8632** | **0.8903** | **intervention 100%; ship candidate** |
| `m3_Bf_gtbox` | 0.8829 | 0.8336 | 0.8632 | 0.8898 | detector-box loss is small |
| `m3_Bf_noglobal` | 0.8759 | 0.8108 | 0.8594 | 0.8885 | global head matters |
| `m3_B_nonneg` | 0.8791 | 0.8274 | 0.7893 | 0.6972 | nonneg without mask weakens concepts |

Per-class weak points for `m3_B_faithful`: **Pneumothorax** F1 0.5715, **Pneumonia** F1 0.6948, **Enlarged Cardiomediastinum** F1 0.8025. Some classes have high F1 but mediocre AUC, e.g. Atelectasis AUC 0.7564 and Fracture AUC 0.7048, so threshold/calibration must be checked before writing strong per-class claims.

### M3 Decisions

- **Done:** choose `B-faithful` for the "why" channel.
- **Done:** keep global head; `noglobal` drops image AUC by about 0.0185.
- **Done:** detector boxes are acceptable for M3; GT boxes improve AUC only about +0.0043.
- **Implemented + ran on gold:** `phase_3/scripts/5-eval.py --diagnostics-json ...` now writes
  per-class/per-concept F1/AUC/ECE and threshold-sweep diagnostics.
- **Ran result:** `artifacts/diagnostics/m3_B_faithful.gold.diagnostics.json`.
  Gold split: image F1 0.8482, image AUC 0.8103, region F1 0.8306, concept F1 0.8615.
- **Still not done:** full test diagnostics have not been rerun after implementation; gold is a quick local check.

## M4 Diagnostic Done From Existing Artifacts

| Run | Test macro-F1 | Change-only F1 | Stable | Improved | Worsened | Read |
|---|---:|---:|---:|---:|---:|---|
| `m4v2_base` | 0.5684 | 0.5662 | 0.5727 | 0.5080 | 0.6244 | old baseline |
| `m4v3_tf` | 0.5638 | **0.5843** | 0.5227 | **0.5272** | **0.6414** | best change-only |
| `m4v3_tf_2blocks` | 0.5805 | 0.5822 | 0.5770 | 0.5263 | 0.6382 | balanced candidate |
| `m4v3_tf_sv2stage` | **0.5866** | 0.5793 | **0.6012** | 0.5243 | 0.6344 | best macro |
| `m4v3_tf_detbox` | 0.5634 | 0.5816 | 0.5269 | 0.5241 | 0.6391 | close to TF |

The persistent weak class is **improved** (~0.50-0.53). Worsened is easier (~0.61-0.64). Architecture changes move the score only modestly, suggesting the next gains likely come from label quality, target design, and diagnostic slicing rather than another head variant.

`phase_4/scripts/3-eval.py --diagnostics-json ...` is now implemented and ran on the gold split for
`m4v3_tf`: macro-F1 0.5665, change-only F1 0.6103, stable 0.4789, improved 0.5066, worsened 0.7139
over 2,021 valid cells. Confusion matrix rows=true, cols=pred for `[stable, improved, worsened]`:
`[[289,178,337],[28,193,80],[86,90,740]]`. The most obvious weak disease slices with n>=20 are
Cardiomegaly, Pneumonia, Pleural Other, Pleural Effusion, Lung Opacity, Consolidation, Edema, and
Atelectasis.

Metric clarification: M4 does **not** predict whether a disease exists from scratch; the disease axis
is the fixed 14 CheXpert label space inherited from M3. M4 predicts the **temporal state** for each
valid `(region, disease)` cell: stable/improved/worsened. `prog_f1_macro` averages F1 over all three
temporal classes. `change_f1_macro` averages only improved+worsened, because a model can look decent
on macro/accuracy by overusing the dominant stable class while failing the clinically important
change-detection task.

### M4 Decisions

- **Done:** v3 temporal-fusion variants beat v2 on change-only F1.
- **Partial:** best run depends on metric: `m4v3_tf` for change-only, `m4v3_tf_sv2stage` for macro.
- **Implemented + ran on gold:** confusion matrix, per-disease progression F1, and per-region progression F1 are logged to JSON.
- **Blocked/P0:** human temporal evaluation set is still missing; silver test numbers are provisional only.

## Roadmap

| # | Item | Priority | Status | Why / next action |
|---:|---|:---:|:---:|---|
| 1 | Lock M3 shipping config: `m3_B_faithful`, detector boxes, global head, attention aggregation | P0 | Done | Use as main M3 checkpoint unless later calibration reveals a class-specific issue. |
| 2 | Add M3 calibration evidence: reliability diagrams, ECE, per-class thresholds | P0 | Partial | JSON ECE + threshold sweep + reliability plots + threshold export are implemented; gold ran; full post-crosswalk test not rerun. |
| 3 | Add per-concept audit: per-concept F1/AUC and good/medium/bad gating | P0 | Partial | Per-concept JSON and concept-gate export implemented; gold diagnostics ran; post-crosswalk full run pending. |
| 4 | Obtain human temporal eval labels, e.g. MS-CXR-T mapping | P0 | Partial | Local MS-CXR-T CSV is integrated and audited; still need decide whether this is the final external test set or only a calibration/audit set. |
| 5 | Diagnose M4 by confusion matrix, disease, region, and prior/current pattern | P1 | Partial | Confusion + disease/region + same-view/cross-view JSON implemented and ran on gold; deeper logit-pattern slicing not implemented. |
| 6 | Improve M4 target/loss/input: label smoothing, TempFuse+M3-delta, ordinal/pairwise, per-disease weighting | P1 | Partial | `--label-smoothing` and `--tempfuse-input-mode feat_logits` implemented; no training result yet; ordinal/pairwise not implemented. |
| 7 | Run M5 end-to-end from best M3/M4 and report verify stats | P1 | Partial | `--stats-json` implemented and M5 demo ran; real M3/M4 pred JSONL audit not run because default pred files are absent. |
| 8 | Add explicit global-grounding label in M5 for GlobalHead findings | P1 | Done | `grounding_type="global"` now appears in findings/provenance for configured global findings. |
| 9 | Update stale results text in phase docs | P1 | Partial | Some docs still say M3 results pending or use older AUC values. |
| 10 | Add bootstrap confidence intervals for headline M3/M4 metrics | P2 | Partial | M3 pred-dump + bootstrap script implemented; post-crosswalk run pending. M4 bootstrap still planned. |
| 11 | Add threshold sweep tables per disease | P2 | Partial | M3 diagnostics and threshold JSON export implemented; post-crosswalk test run pending. |
| 12 | Add M3 deletion/insertion audit with zero and mean-feature baselines | P2 | Planned | Supporting grounding evidence; not the main faithfulness claim. |
| 13 | Compare CheXplus/NIH data policy via ablation | P2 | Planned | Keep weak labels train/pretrain only; decide value by clean MIMIC eval. |
| 14 | Add rule-parser prior-report branch for M4 as separate B11-B experiment | P3 | Planned | Potential accuracy extension; keep outside v1 pure-image core. |
| 15 | Reader-study decision: run small study or downgrade to motivation | P3 | Planned | Only needed if claiming benefit for less experienced clinicians. |

## Commands To Run / Reproduce

Use the repo `.venv`; full test runs may take longer than the gold smoke runs below.

### One-command audit runner

```bash
# Personal machine: RTX 4060 8GB VRAM, 4 workers, conservative batches.
bash audit.sh --profile local4060 --split gold

# Force CPU if CUDA is busy or VRAM is tight.
bash audit.sh --profile local4060 --split gold --device cpu

# Server / H100 mini profile: ~20GB VRAM, 16 workers.
bash audit.sh --profile h100mini --split test --run-full

# Optional: launch the full M4 retrain matrix from the same entrypoint.
bash audit.sh --profile h100mini --split test --run-m4-train-grid
```

Status: **implemented**. `audit.sh --help` was tested. The individual gold commands it runs were
tested locally; full `test` profile was not rerun. MS-CXR-T audit is now included automatically when
`data/MS_CXR_T_temporal_image_classification_v1.0.0.csv` exists.

### Full Phase-3 rerun after approved improvements

```bash
# Full literal Phase 3: crosswalk patch, retrain grid, diagnostics, gates, bootstrap,
# M3 JSONL for Phase 5, and M3 region cache for Phase 4.
bash phase_3.sh --profile h100mini --tag xwalk_v2

# If training already completed and only audits/infer/cache need to be regenerated.
bash phase_3.sh --profile h100mini --tag xwalk_v2 --skip-train

# Small local smoke, not a final run.
bash phase_3.sh --profile local4060 --tag smoke --epochs 2 --audit-splits gold
```

Status: **implemented, not run full**. This supersedes `phase_3/run_experiments.sh` for the final
Phase-3 rerun. The old script remains as a legacy quick grid.

Profile behavior:

| Profile | Device default | Workers | M3 batch | M4 batch | Intended use |
|---|---|---:|---:|---:|---|
| `local4060` | `cuda` | 4 | 64 | 24 | gold/smoke audits and small diagnostics |
| `h100mini` | `cuda` | 16 | 512 | 128 | full test diagnostics / server reruns |

### What should run on RTX 4060 8GB locally?

Likely OK locally:

- `bash audit.sh --profile local4060 --split gold`
- M3 gold diagnostics on CPU or CUDA.
- M3 reliability plotting.
- M3 gold faithfulness diagnostics.
- M4 gold diagnostics for `m4v3_tf`.
- MS-CXR-T audit for one checkpoint; it ran locally on CPU in about one minute.
- M5 demo and M5 verify stats if `m3_pred.jsonl`/`m4_pred.jsonl` already exist.
- M4 short training experiments with small batches, e.g. `--batch 16` or `--batch 24`, but expect slow runs.

Prefer server / H100 mini:

- Full `test` diagnostics for all M3/M4 runs.
- Any full M3 or M4 retraining grid.
- M4 tempfuse training with multiple ablations (`--label-smoothing`, focal, same-view, two-stage).
- Full M3/M4 inference JSONL generation for M5 over test/train-scale splits.
- `bash phase_4/run_m4_retrain_matrix.sh --profile h100mini` full retraining sweep.
- Bootstrap confidence intervals, deletion/insertion, and CheXplus/NIH ablations.

RTX 4060 caution: use `--workers 4`, keep `M3_BATCH<=64`, `M4_BATCH<=24`, and lower M4 batch to 8-16
if CUDA OOM appears.

### M4 label-smoothing experiment: implemented, not trained

```bash
python3 phase_4/scripts/2-train.py \
  --arch tempfuse \
  --features-root data/features/frozen \
  --m3-labels-dir data/m3_labels \
  --m4-labels-dir data/m4_labels \
  --pairs data/m4_labels/m3_pairs.jsonl \
  --box-source gt \
  --name m4v3_tf_smooth005 \
  --loss ce \
  --label-smoothing 0.05 \
  --select-metric change \
  --patience 10 \
  --device cuda
```

Status: **implemented, not run**. This is likely better on the server for a real run; on RTX 4060,
try `--batch 16 --workers 4` for a smaller experiment.

### M3 diagnostics: implemented, gold run completed

```powershell
& ".venv\Scripts\python.exe" phase_3\scripts\5-eval.py `
  --ckpt RUN\m3_B_faithful\best.pt `
  --labels-dir data\m3_labels `
  --features-root data\frozen `
  --split gold `
  --batch 64 `
  --workers 0 `
  --box-source detector `
  --device cpu `
  --diagnostics-json artifacts\diagnostics\m3_B_faithful.gold.diagnostics.json
```

Status: **implemented and ran** on 2026-07-07. Result: image F1 0.8482, image AUC 0.8103,
region F1 0.8306, concept F1 0.8615. Full test diagnostics: **not rerun yet**.

For full test diagnostics on GPU/Kaggle:

```bash
python3 phase_3/scripts/5-eval.py \
  --ckpt data/run/m3_B_faithful/best.pt \
  --labels-dir data/m3_labels \
  --features-root data/features/frozen \
  --split test \
  --box-source detector \
  --workers 16 \
  --device cuda \
  --diagnostics-json artifacts/diagnostics/m3_B_faithful.test.diagnostics.json
```

### M4 diagnostics: implemented, gold run completed

```powershell
& ".venv\Scripts\python.exe" phase_4\scripts\3-eval.py `
  --ckpt RUN\m4v3_tf\best.pt `
  --features-root data\frozen `
  --m3-labels-dir data\m3_labels `
  --m4-labels-dir data\m4_labels `
  --pairs data\m4_labels\m3_pairs.jsonl `
  --split gold `
  --batch 32 `
  --device cpu `
  --diagnostics-json artifacts\diagnostics\m4v3_tf.gold.diagnostics.json
```

Status: **implemented and ran** on 2026-07-07. Result: macro-F1 0.5665, change-only F1 0.6103.
Full test diagnostics: **not rerun yet**.

For full test diagnostics on GPU/Kaggle:

```bash
python3 phase_4/scripts/3-eval.py \
  --ckpt data/run/m4v3_tf/best.pt \
  --features-root data/features/frozen \
  --m3-labels-dir data/m3_labels \
  --m4-labels-dir data/m4_labels \
  --pairs data/m4_labels/m3_pairs.jsonl \
  --split test \
  --device cuda \
  --diagnostics-json artifacts/diagnostics/m4v3_tf.test.diagnostics.json
```

### MS-CXR-T external audit: implemented, local run completed

```bash
python3 phase_4/scripts/5-mscxrt_audit.py \
  --ckpt data/run/m4v3_tf/best.pt \
  --csv data/MS_CXR_T_temporal_image_classification_v1.0.0.csv \
  --features-root data/features/frozen \
  --region-cache data/m4_region_cache \
  --m3-labels-dir data/m3_labels \
  --split all \
  --device cuda \
  --out-json artifacts/diagnostics/m4v3_tf.mscxrt.diagnostics.json
```

Status: **implemented and ran locally** on CPU using `RUN/m4v3_tf/best.pt` and `data/frozen`.
Coverage: 964/1,045 usable pairs. Results by region-to-image aggregation:

| Aggregation | Macro-F1 | Change-only F1 | Stable | Improved | Worsened |
|---|---:|---:|---:|---:|---:|
| `mean` | 0.5693 | 0.6445 | 0.4189 | 0.5657 | 0.7233 |
| `max` | 0.5406 | 0.6334 | 0.3550 | 0.5474 | 0.7195 |
| `lse` | **0.5695** | **0.6463** | 0.4161 | 0.5672 | 0.7253 |

Interpretation: MS-CXR-T is image-level while VERA M4 is region-level, so these numbers are an
external audit through an explicit aggregation, not a region-level temporal localization score.

### M4 retraining matrix: implemented, not run

```bash
# Server H100 mini / 20GB VRAM profile.
bash phase_4/run_m4_retrain_matrix.sh --profile h100mini --eval-split test

# Include optional MS-CXR-T adapter fine-tunes after the main v3/v4 runs.
bash phase_4/run_m4_retrain_matrix.sh --profile h100mini --eval-split test --run-adapters
```

Status: **implemented, not run**. The matrix covers:

- v3 controls: retrain, label smoothing 0.03/0.05/0.10, focal, two-stage, same-view+two-stage,
  flip-consistency KL, same-view curriculum, two fuse blocks, detector boxes.
- v4 hybrid: all equivalent variants with `--tempfuse-input-mode feat_logits`, i.e.
  `[TempFuse region feature ; M3 curr logits ; M3 prior logits ; M3 delta logits]`.
- regiondiff controls: full, logits-only, and diff-only.
- optional MS-CXR-T adapters: head-only and pool+head fine-tuning on subject-hash train/val splits.

### Full Phase-4 runner: implemented, not run

```bash
# Run this only after Phase 3 has produced a faithful M3 checkpoint.
bash phase_3.sh --profile h100mini --tag xwalk_v2
bash phase_4.sh --profile h100mini --tag xwalk_v2

# Audit/plot/infer only after training exists.
bash phase_4.sh --profile h100mini --tag xwalk_v2 --skip-train

# Include MS-CXR-T adapters.
bash phase_4.sh --profile h100mini --tag xwalk_v2 --run-adapters
```

Phase-4 prerequisite checkpoint:

```text
data/run/m3_B_faithful_xwalk_v2/best.pt
artifacts/diagnostics/m3_B_faithful_xwalk_v2.*.faithfulness.json
```

The script checks `why_faithful_allowed` when the JSON is present. If the faithfulness JSON is absent,
it warns; for a final run, regenerate Phase 3 first rather than treating the warning as harmless.

## Immediate Implementation Suggestions

1. Run full `test` audit with `bash audit.sh --profile h100mini --split test --run-full`.
2. Train the first noisy-label M4 ablation with `--label-smoothing 0.05`, then compare to `m4v3_tf`.
3. Add reliability plot selection for rare classes separately, not only top-ECE classes.
4. Add M4 deeper prior/current slicing when logits are available, e.g. prior-high/current-low vs prior-low/current-high.
5. Add real M5 audit once `m3_pred.jsonl` and `m4_pred.jsonl` have been generated.

## Paper-Framing Notes

- Claim M3 "why" through `B-faithful`, not through the free MLP head.
- Treat M4 silver results as development numbers, not final scientific evidence.
- Report detector-box vs GT-box M3 ablation as evidence that B1 is largely closed downstream.
- Do not overclaim per-concept explanations until the per-concept audit is logged and gated.

## Server Data Manifest

Minimum upload for **audit/eval/inference** on the server:

```text
data/
  features/frozen/                 # or set FEAT=...; M1 BioViL-T .pt/.npy [197,C]
  m3_labels/
    manifest.jsonl
    image_chexpert.npy
    region_chexpert.npy
    region_concepts.npy
    present_mask.npy
    present_mask_det.npy
    boxes.npy
    boxes_det.npy
  m4_labels/
    manifest.jsonl
    progression.npy
    m3_pairs.jsonl
  MS_CXR_T_temporal_image_classification_v1.0.0.csv   # optional but recommended for external M4 audit
  run/                             # checkpoints if not using RUN/
    m3_B_faithful/best.pt
    m4v3_tf/best.pt
```

Needed only for specific jobs:

- `data/m4_region_cache/`: required for `regiondiff` M4 runs and for v4
  `tempfuse --tempfuse-input-mode feat_logits`; not required for old v3 `tempfuse` eval/train,
  because old `tempfuse` reads patch features directly.
- `data/m3_pred.jsonl` and `data/m4_pred.jsonl`: required only for real M5 report audit. Generate with
  `phase_3/scripts/7-infer.py` and `phase_4/scripts/4-infer.py`.
- `data/mimic-cxr-448/`, `data/chest-imagenome/`, metadata CSV/JSONL: only needed if rebuilding labels,
  detector boxes, worklists, or features. Not needed for the current audit scripts.
- `RUN/` and `LOGS/`: useful for local audit history, but not required if `data/run/` has checkpoints.

`audit.sh` now auto-detects checkpoint roots: it uses `RUN/...` if present, otherwise `data/run/...`.

## Phase 4 Research Notes Toward F1 > 0.6

External pointers checked on 2026-07-08:

- **MS-CXR-T** provides 1,326 paired-study temporal image-classification labels across five findings
  with classes Improving/Stable/Worsening. This is the cleanest immediate candidate for B2-style human
  temporal eval and for a small supervised calibration/fine-tune layer.
- MS-CXR-T's published supplemental distribution is highly relevant to our weak class: total class mix
  is about 18% improving / 40% stable / 42% worsening across Consolidation, Edema, Pleural Effusion,
  Pneumonia, and Pneumothorax. This aligns with our observation that "improved" is hardest.
- Recent longitudinal CXR work repeatedly uses explicit prior/current modeling, temporal VLP pretraining,
  sentence/report temporal structure, or Siamese transformer-style difference modules. The common lesson:
  the model must learn temporal *direction*, not just detect abnormality twice.

Most promising VERA-compatible ideas:

1. **MS-CXR-T adapter / eval bridge (P0/P1).** Map the five MS-CXR-T findings to CheXpert labels and
   evaluate M4 on that set. Then optionally fine-tune only the small M4 head or calibration layer on
   MS-CXR-T. **Implemented and audit ran**; adapter script is implemented but not trained.
2. **Direction-first auxiliary loss (P1).** Add a binary direction loss only on changed cells:
   improved vs worsened, separate from stable-vs-change. `twostage` already factorizes this structurally,
   but the loss is still ordinary CE over composed logits. Make the decomposition explicit and upweight
   direction errors.
3. **Temporal consistency regularizer (P1).** For time-flip augmentation, enforce
   `P_improved(curr,prior) ~= P_worsened(prior,curr)` and `P_stable` invariant. This directly targets
   temporal direction and should help the improved class. **Implemented** as
   `--flip-consistency-weight`; not trained yet.
4. **Delta-logit prior (P1).** For `tempfuse`, reintroduce M3 disease-logit deltas alongside patch-fused
   features: `[tempfuse_region ; m3_logit_curr ; m3_logit_prior ; m3_logit_delta]`. Current `tempfuse`
   head sees only fused region features, so it may be relearning disease status instead of using M3's
   calibrated disease signal. **Implemented as** `--tempfuse-input-mode feat_logits`; not trained yet.
5. **Change-aware sampling (P1).** Oversample pairs/cells with improved labels or diseases where improved
   is underperforming. The current class weighting helps globally, but it does not guarantee enough
   gradient on rare disease-specific improvement patterns.
6. **Pair-quality gates (P2).** Use same-view plus study-time gap bins. Cross-view pairs and very long
   intervals may be label-noisy; very short intervals may be mostly stable or device-driven. Add eval
   slices first, then train gates if slices show a clear pattern. **Same-view filtering already existed;
   same-view curriculum is now implemented** as `--curriculum-same-view-epochs`.
7. **Uncertainty-aware silver loss (P2).** Use label smoothing (implemented) and possibly lower weights
   for labels from weak comparison cues likely to be ambiguous. This is cheap to test.
8. **Prior-report branch as separate B11-B (P3).** If prior reports are available at deployment, parse
   prior report into a clean prior disease logit and train M4 with a `prior_clean_available` flag. Keep
   this outside v1's pure-image core.

Concrete next server experiment:

```bash
# First cheap train-side test of noisy-label smoothing.
EP=30 BATCH=128 W=16 PAT=10 python3 phase_4/scripts/2-train.py \
  --arch tempfuse \
  --features-root data/features/frozen \
  --m3-labels-dir data/m3_labels \
  --m4-labels-dir data/m4_labels \
  --pairs data/m4_labels/m3_pairs.jsonl \
  --out data/run \
  --name m4v3_tf_smooth005 \
  --box-source gt \
  --loss ce \
  --label-smoothing 0.05 \
  --select-metric change \
  --patience 10 \
  --batch 128 \
  --workers 16 \
  --device cuda
```
