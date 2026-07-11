# VERA

**Verifiable, Evidence-grounded Regional Assembly for Faithful Temporal Chest
X-ray Reporting**

VERA is a research framework for producing chest radiograph reports whose
findings, anatomical locations, confidence estimates, and temporal statements
can be traced back to explicit model outputs. Instead of asking a generative
model to infer diagnoses while writing free text, VERA first constructs a
regional and temporal prediction table and then assembles the report from claims
that the table authorizes.

> [!WARNING]
> VERA is a research prototype. It is not a medical device, has not been
> prospectively validated, and must not be used for clinical diagnosis or
> autonomous report release.

The repository directory retains the historical `KAN-TRaCE` name. The current
project and manuscript are named **VERA**. KAN heads remain reproducible
ablations; they are not the central contribution or the default shipping path.

## Why VERA?

Fluent radiology text is not necessarily faithful to image evidence. A report
generator may add an unsupported observation, attach a correct observation to
the wrong location, or describe interval change without a valid prior study.
VERA addresses this by enforcing four principles:

- **Verifiable:** every surfaced claim has structured provenance;
- **Evidence-grounded:** anatomical and temporal language is tied to explicit
  regional model outputs;
- **Regional:** reasoning preserves a 29-region chest anatomy representation;
- **Assembly:** the final report is a constrained readout, not an unconstrained
  diagnostic generation step.

VERA distinguishes **faithfulness to the model's computation** from **clinical
correctness**. Provenance can show why the system emitted a claim, but only
human-validated evaluation can establish whether that claim is medically
correct.

## System overview

```text
current CXR + optional same-patient prior
                    │
                    ▼
┌──────────────────────────────────────────────────────────────────┐
│ M0  Data preparation                                             │
│     aligned 448×448 images · patient split · labels · pairs      │
└─────────────────────────────┬────────────────────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ M1  Frozen BioViL-T encoder                                      │
│     image → global token + 14×14 spatial grid = [197, 512]       │
└────────────────────┬───────────────────────────┬─────────────────┘
                     │                           │
                     ▼                           │
┌────────────────────────────────────┐           │
│ M2  Anatomical supervision         │           │
│     YOLO detector → 29 boxes       │           │
│     report parser → train labels   │           │
└────────────────────┬───────────────┘           │
                     └───────────────┬───────────┘
                                     ▼
┌──────────────────────────────────────────────────────────────────┐
│ M3  Regional observation model                                  │
│     bbox-masked attention pooling                                │
│     29 regions × 69 concepts → 29 regions × 14 observations     │
│     masked non-negative concept-to-observation path              │
└────────────────────┬───────────────────────────┬─────────────────┘
                     │ current                   │ prior
                     └──────────────┬────────────┘
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│ M4  Regional temporal model                                     │
│     current ↔ prior → 29 × 14 × {stable, improved, worsened}    │
└─────────────────────────────┬────────────────────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ M5  Faithful report layer                                       │
│     calibrate → assert / hedge / omit → temporal guard           │
│     template realization → verification → claim provenance      │
└─────────────────────────────┬────────────────────────────────────┘
                              ▼
             structured report + evidence/change ledger
```

Reports and scene graphs may provide supervision during training. The principal
v1 inference path remains image based: current image, an optional real prior
image, predicted anatomical boxes, M3, M4, and M5. When no valid prior exists,
M5 suppresses temporal language by construction.

## Modules

| Module | Responsibility | Main output | Current status |
|---|---|---|---|
| M0 | Metadata, geometry, three-state labels, patient splits, temporal pairing | JSONL manifests and label arrays | Implemented |
| M1 | Frozen BioViL-T feature extraction | one `[197,512]` cache per image | Implemented |
| M2 | 29-region detection and report-to-scene-graph supervision | detector boxes and structured weak labels | Implemented and audited |
| M3 | Regional concepts and 14 CheXpert observations | `region_logits[29,14]`, concepts, features, attention | Ship candidate implemented; final rerun pending |
| M4 | Per-region temporal progression | `prog_logits[29,14,3]` | Implemented; scientific claim remains provisional |
| M5 | Calibrated structured reporting and verification | findings, prose, provenance, review flags | Baseline implemented; real-output audit pending |

The current M3 ship candidate is `m3_B_faithful`: detector boxes, bbox-masked
attention pooling, a global head for relational findings, and a masked
non-negative concept-to-observation mapping. M4 consumes frozen M3 regional
features and logits so temporal training cannot silently alter the single-image
evidence path.

## Explainability and safety mechanisms

### Regional and concept faithfulness

- pooling is spatially masked by predicted anatomical boxes;
- the principal concept-to-observation head permits only curated concept links;
- non-negative weights impose a monotonic concept intervention direction;
- concept explanations are gated by concept quality and intervention tests;
- global findings are explicitly marked as globally grounded instead of being
  assigned a false regional explanation.

### Temporal explainability

- current/prior order is explicit;
- time reversal should preserve `stable` and exchange `improved ↔ worsened`;
- input-group intervention audits reliance on current features, prior features,
  explicit differences, and M3 disease logits;
- no valid prior means no progression claim and no temporal prose.

See [Phase 4 explainability](docs/phase4_explainability.md).

### Report faithfulness

The implemented M5 baseline uses calibrated `assert / hedge / omit` decisions,
deterministic template realization, a hard temporal guard, and round-trip report
checks. The next technical extension is a
[proof-carrying radiology report](docs/phase5_proof_carrying_report.md), in which
each sentence carries a machine-checkable certificate linking it to the M3/M4
cell that authorized it. This extension is specified but is not yet implemented.

A separate [selective double-reading proposal](docs/phase5_selective_double_reading.md)
studies whether proof status and junior–VERA concordance can reduce the number of
cases requiring a second senior reader. No workload-reduction claim is made
without a reader study.

## Data policy

| Dataset | Intended use | Evaluation policy |
|---|---|---|
| MIMIC-CXR + Chest ImaGenome | Primary M3/M4 training; regional and temporal supervision | Main clean/gold evaluation where human annotations exist |
| MS-CXR-T | Human temporal labels for five findings | External M4 audit; final role still being locked |
| CheXpert Plus | Planned cross-institution weak pretraining for M3 | Train/pretrain only; never a headline test set after use in training |
| NIH ChestX-ray14 | Optional disease-level generalization check | M3 disease evaluation only; does not support regional/temporal claims |

CheXpert Plus is not mixed directly into the final MIMIC run. The planned order
is:

```text
CheXpert Plus weak M3 pretraining
    → reset optimizer and scheduler
    → clean MIMIC/Chest ImaGenome M3 fine-tuning
    → calibration and explanation gates on MIMIC validation
    → freeze M3 and rebuild the M4 cache
```

Whether CheXpert Plus remains in the paper is decided by an ablation on clean,
human-validated evaluation. See the
[CheXpert Plus integration plan](docs/chexplus_integration_plan.md).

## Current experimental snapshot

These numbers are development/audit results, not final clinical claims. The full
post-crosswalk Phase 3 and Phase 4 server reruns remain pending.

| Component | Evaluation | Current result | Qualification |
|---|---|---:|---|
| M2 detector | silver MIMIC validation | mAP50 `0.931`; mAP50–95 `0.694` | gold detector evaluation still preferred for the paper |
| M3 `B-faithful` | local gold audit | image F1 `0.8482`; image AUC `0.8103`; region F1 `0.8306`; concept F1 `0.8615` | final post-crosswalk test rerun pending |
| M4 `m4v3_tf` | local gold audit | macro-F1 `0.5665`; change-only F1 `0.6103` | current labels remain a major risk |
| M4 `m4v3_tf` | MS-CXR-T, 964 usable pairs | macro-F1 `0.5695`; change-only F1 `0.6463` | image-level external audit via explicit regional aggregation |
| M5 baseline | synthetic smoke cases | temporal guard, template verification, and JSON round-trip pass | real M3/M4 prediction audit not yet run |

The main unresolved scientific risk is M4. `Improved` remains the weakest temporal
class, and no strong temporal-faithfulness statement should be made until the
human temporal evaluation policy and final rerun are frozen.

For the detailed evidence and current priorities, read the
[experiment audit roadmap](docs/VERA_experiment_audit_roadmap.md).

## Repository layout

```text
.
├── preprocess/          metadata, image, scene-graph, and dataset preparation
├── phase_1/             frozen BioViL-T feature extraction
├── phase_2/             29-region detector and report parsers
├── phase_3/             current regional M3 implementation and experiments
├── phase_4/             current temporal M4 implementation and experiments
├── src/phase_5/         faithful report assembler and smoke verification
├── artifacts/           diagnostics and visualizations safe to keep in the repo
├── docs/                architecture, audits, experiment plans, and paper draft
├── phase_3.sh           full tagged M3 rerun/audit pipeline
├── phase_4.sh           full tagged M4 rerun/audit pipeline
└── pyproject.toml        base package metadata and non-GPU utilities
```

Each phase has its own README with its data contracts and commands:

- [Phase 1 — frozen BioViL-T](phase_1/README.md)
- [Phase 2 — detector and scene graph](phase_2/README.md)
- [Phase 3 — regional concepts and observations](phase_3/README.md)
- [Phase 4 — temporal progression](phase_4/README.md)
- [Phase 5 design](docs/phase_5_new.md)

## Environment

The repository does not yet provide one locked environment for every phase.
`pyproject.toml` contains the base data/analysis dependencies; GPU stages also
need phase-specific packages such as PyTorch, Ultralytics, BioViL-T/
`hi-ml-multimodal`, Transformers, PEFT, and scikit-learn. Use the corresponding
phase README or notebook rather than installing an unversioned global stack.

Data and model checkpoints are intentionally not distributed with the repository.
Most paths under `data/`, `RUN/`, and `LOGS/` are local or remote artifacts.

The root Phase 3/4 runners are Bash scripts. On Windows, use WSL or Git Bash for
the runners; Python-only preparation and verification commands work directly in
PowerShell.

## Reproduction order

### 1. Prepare data and M1/M2 artifacts

Follow the phase-specific READMEs to create:

```text
data/m3_labels/
data/m4_labels/
data/m4_labels/m3_pairs.jsonl
data/features/frozen/       # or data/frozen/ for the local profile
detector predictions aligned to the M3 manifest
```

Do not mix image transforms: the feature grid and anatomical boxes must use the
same 448×448 coordinate frame.

### 2. Run the full Phase 3 grid

```bash
# Server profile: full tagged rerun.
bash phase_3.sh --profile h100mini --tag xwalk_v2

# Audit/inference/cache only when tagged checkpoints already exist.
bash phase_3.sh --profile h100mini --tag xwalk_v2 --skip-train

# Local smoke run; not a final experiment.
bash phase_3.sh --profile local4060 --tag smoke --epochs 2 --audit-splits gold
```

The runner patches the concept crosswalk, trains the ablation grid, exports
diagnostics/calibration/concept gates, produces M3 predictions for M5, and builds
the frozen regional cache required by M4.

### 3. Run Phase 4 only after the M3 faithfulness gate

```bash
bash phase_4.sh --profile h100mini --tag xwalk_v2

# Audit existing tagged runs without retraining.
bash phase_4.sh --profile h100mini --tag xwalk_v2 --skip-train

# Local smoke run.
bash phase_4.sh --profile local4060 --tag smoke --epochs 2 --audit-splits gold
```

Phase 4 checks the M3 faithfulness artifact, rebuilds the frozen-M3 cache when
enabled, runs the temporal ablation matrix, evaluates MS-CXR-T when available,
and emits prediction files for M5.

### 4. Verify the current M5 baseline

Bash:

```bash
PYTHONPATH=src python -m phase_5.verify_m5
```

PowerShell:

```powershell
$env:PYTHONPATH = "src"
python -m phase_5.verify_m5
```

This is a synthetic invariant test, not a clinical evaluation. Real report
evaluation requires M3/M4 outputs from the frozen selected checkpoints.

## Evaluation principles

- never evaluate a regional or temporal claim only against labels produced by
  the same report parser used for training;
- report image/region/concept/temporal metrics separately;
- use macro and per-class temporal metrics so the dominant `stable` class cannot
  hide weak change detection;
- report calibration and risk–coverage, not only discrimination;
- evaluate factuality, faithfulness, and fluency as separate axes;
- reserve concept explanations for concepts that pass their prediction and
  intervention gates;
- treat CheXpert Plus pseudo labels as train-only weak supervision;
- use reader studies before claiming that VERA benefits junior readers or
  reduces senior workload.

## Documentation

- [Architecture and implementation flow](docs/flow.md)
- [Detailed M3/M4/M5 specification](docs/VERA_phase_3_4_5_spec.md)
- [Methodological concerns and scope decisions](docs/VERA_methodology_concerns.md)
- [Experiment audit and roadmap](docs/VERA_experiment_audit_roadmap.md)
- [Phase 4 explainability](docs/phase4_explainability.md)
- [Proof-carrying Phase 5 proposal](docs/phase5_proof_carrying_report.md)
- [Selective double-reading proposal](docs/phase5_selective_double_reading.md)
- [CheXpert Plus integration plan](docs/chexplus_integration_plan.md)
- [Introduction, related work, and structured abstract draft](docs/paper_introduction_related_work.tex)

## Research scope

VERA currently supports adult chest radiograph research with a fixed 29-region
anatomical vocabulary, 69 concepts, 14 CheXpert observations, and three temporal
states. It does not model unrestricted diagnoses, clinical history, treatment
recommendations, or causal disease mechanisms. The final report remains a draft
for radiologist review.

## License and data access

No repository license has been added yet. Dataset access is governed by the
terms of the original providers, including MIMIC-CXR/PhysioNet, Chest ImaGenome,
CheXpert Plus, and MS-CXR-T. Users are responsible for obtaining the required
credentials and following each data-use agreement.
