# M4 (T-KAN) progression-label statistics

Per-`(region, disease)` temporal progression targets built by `phase_4/scripts/1-labels.py` from Chest ImaGenome `comparison_cues`. Regenerate with `python phase_4/scripts/dataset_stats.py`.

## How we count (definitions)

- **Source.** Labels come from the **current** study's scene-graph `comparison_cues` (ImaGenome NLP). A cued phrase's positive findings set the progression of the diseases they feed; conflicts resolve **worsened > improved > stable**.

- **Classes.** `0 stable` ("no change" cue) · `1 improved` · `2 worsened`. A cell with **no cue** is `-100` = **masked** (never trained, never scored).

- **Supervision gate (both granularities below use it).** A `(region, disease)` cell counts only if it has a cue **and** the region is **present in BOTH** the current and prior study (`REQUIRE_PRIOR_PRESENT`). No-prior / no-cue images carry no M4 signal and are excluded (they flow to M5's temporal guard, not a data error).

- **Unit 1 — cell** `(region × disease)`, 29×14: the exact unit the masked cross-entropy loss and macro-F1 operate on. **This is the primary distribution.**

- **Unit 2 — study×disease**: collapse the 29 regions of a disease within a study to ONE label (priority worsened>improved>stable). Reported for finding-level interpretability; a spatially spread finding (effusion/edema) occupies many region-cells, so cell-level over-weights it relative to this collapsed view.

- **`gold`** is the held-out split; note its cues are still NLP-silver (see caveat).


## A. Image funnel (how many studies survive each gate)

| gate | train | val | test | gold | total |
|--|--:|--:|--:|--:|--:|
| ok rows | 155,127 | 22,136 | 44,059 | 833 | 222,155 |
| + has a cue (n_cued>0) | 75,475 | 10,821 | 21,528 | 266 | 108,090 |
| + has a prior (pairable) | 67,340 | 9,645 | 19,164 | 264 | 96,413 |
| + region present in both (**used by M4**) | 65,286 | 9,344 | 18,579 | 263 | 93,472 |

- Temporal pairs available in `m3_pairs.jsonl` (curr→prior): **253,306**. Pairs actually **used by M4** (last funnel row): **93,472**.


## B. Class distribution — CELL `(region × disease)`, present-masked *(primary)*

| split | stable | improved | worsened | total cells |
|--|--:|--:|--:|--:|
| train | 259,826 (45.1%) | 111,461 (19.4%) | 204,413 (35.5%) | 575,700 |
| val | 36,070 (44.8%) | 15,381 (19.1%) | 29,059 (36.1%) | 80,510 |
| test | 73,836 (45.4%) | 32,012 (19.7%) | 56,759 (34.9%) | 162,607 |
| gold | 804 (39.8%) | 301 (14.9%) | 916 (45.3%) | 2,021 |
| **all** | 370,536 (45.1%) | 159,155 (19.4%) | 291,147 (35.5%) | 820,838 |

- **change-only (improved + worsened)** = 450,302 cells = **54.9%**. accuracy ≈ predicting "stable" everywhere would therefore miss the majority of cells — read **change-only F1**, not accuracy.


## C. Class distribution — collapsed to study×disease *(finding-level view)*

| split | stable | improved | worsened | total labels |
|--|--:|--:|--:|--:|
| train | 84,315 (48.0%) | 31,596 (18.0%) | 59,890 (34.1%) | 175,801 |
| val | 11,736 (47.6%) | 4,439 (18.0%) | 8,499 (34.4%) | 24,674 |
| test | 24,099 (48.4%) | 9,073 (18.2%) | 16,669 (33.4%) | 49,841 |
| gold | 305 (45.9%) | 90 (13.5%) | 270 (40.6%) | 665 |
| **all** | 120,455 (48.0%) | 45,198 (18.0%) | 85,328 (34.0%) | 250,981 |

## D. Context — raw tensor occupancy

- `progression.npy` shape **[222,155, 29, 14]** (= 90,194,930 cells). **98.94%** are `-100` (no cue → masked). Only the remaining cued cells — further gated by present-in-both — enter B/C above.


## Caveat (OPEN decision B2)

These labels are **NLP-derived (silver)** from `comparison_cues`: fine to **train** on, but the improved/stable/worsened **test set must be human-annotated** before any faithful temporal claim. All three splits here share the same silver source, so their distributions are near-identical — that consistency is expected, **not** evidence of a clean eval. Numbers reported on this silver test are provisional pending a human temporal set (e.g. MS-CXR-T).
