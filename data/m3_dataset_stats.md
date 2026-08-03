# M3 dataset statistics

## A. Images / patients / studies, by dataset & split
| dataset | images | patients | studies | train | val | test | gold |
|--|--|--|--|--|--|--|--|
| chexplus | 191,046 | 64,686 | 187,650 | 133,726 | 19,109 | 38,211 | 0 |
| mimic | 222,168 | 63,334 | 212,566 | 155,138 | 22,138 | 44,059 | 833 |

## G. Report length (words)
| dataset | n | median | p25 | p75 | mean | empty |
|--|--|--|--|--|--|--|
| chexplus | 191,046 | 110 | 93 | 133 | 118 | 0 |
| mimic | 222,168 | 79 | 62 | 104 | 87 | 0 |

## D. MIMIC image-level CheXpert (pos / neg / unknown)
| # | label | pos | neg | unknown |
|--|--|--|--|--|
| 0 | Atelectasis | 45,141 | 1,438 | 175,589 |
| 1 | Cardiomegaly | 43,519 | 15,030 | 163,619 |
| 2 | Consolidation | 10,566 | 7,763 | 203,839 |
| 3 | Edema | 26,809 | 23,895 | 171,464 |
| 4 | Enlarged Cardiomediastinum | 6,987 | 5,026 | 210,155 |
| 5 | Fracture | 4,331 | 870 | 216,967 |
| 6 | Lung Lesion | 6,098 | 784 | 215,286 |
| 7 | Lung Opacity | 50,576 | 2,769 | 168,823 |
| 8 | No Finding | 73,405 | 0 | 148,763 |
| 9 | Pleural Effusion | 53,330 | 25,302 | 143,536 |
| 10 | Pleural Other | 1,936 | 114 | 220,118 |
| 11 | Pneumonia | 15,883 | 22,202 | 184,083 |
| 12 | Pneumothorax | 10,347 | 41,069 | 170,752 |
| 13 | Support Devices | 65,657 | 3,278 | 153,233 |

## B. Studies per patient (MIMIC-CXR)
- patients: **65,379**  | exactly 1 study: **32,695** (50.0%)  | ≥2 studies: **32,684**
- studies/patient: median 1, p75 3, p99 28, max 158, mean 3.48

## C. Temporal pairs (prior ↔ current)
- total pairs: **253,306**  | same ViewPosition: **165,627** (65.4%)  | distinct current images with a prior: **253,306**
- pairs where BOTH images are concept-labeled (MIMIC+scene): **145,643**
- days apart: median 9, p25 1, p75 104

## E. Comparison cues (M4 progression supervision)
| cue | region-instances |
|--|--|
| no change | 373,696 |
| worsened | 190,708 |
| improved | 130,281 |

## F. Concepts (image-level): mean **3.5** positive concepts/image. Top 15:
| concept | images+ |
|--|--|
| lung opacity | 148,945 |
| atelectasis | 76,285 |
| pleural effusion | 69,608 |
| enlarged cardiac silhouette | 55,708 |
| pulmonary edema/hazy opacity | 34,036 |
| pneumonia | 33,121 |
| enteric tube | 25,332 |
| vascular congestion | 23,788 |
| endotracheal tube | 21,356 |
| consolidation | 17,043 |
| pleural/parenchymal scarring | 13,688 |
| picc | 13,552 |
| lung lesion | 13,028 |
| tortuous aorta | 12,913 |
| linear/patchy atelectasis | 12,337 |

## H. Concept → disease relevance (XGBoost, image-level)
How well the 69 concepts predict each CheXpert disease, + top driving concepts.
| disease | n(pos/neg) | AUC | top concepts (gain) |
|--|--|--|--|
| Atelectasis | 45,138/1,438 | 0.840 | atelectasis, pleural effusion, lobar/segmental collapse, pulmonary edema/hazy opacity |
| Cardiomegaly | 43,514/15,030 | 0.915 | enlarged cardiac silhouette, airspace opacity, fluid overload/heart failure, pericardial effusion |
| Consolidation | 10,565/7,763 | 0.986 | consolidation, pneumonia, lung opacity, ij line |
| Edema | 26,806/23,894 | 0.981 | pulmonary edema/hazy opacity, fluid overload/heart failure, vascular redistribution, infiltration |
| Enlarged Cardiomediastinum | 6,986/5,026 | 0.827 | mediastinal widening, lung opacity, mediastinal displacement, enlarged cardiac silhouette |
| Fracture | 4,330/870 | 0.716 | chest tube, pneumothorax, clavicle fracture, lung opacity |
| Lung Lesion | 6,096/784 | 0.887 | lung lesion, lung opacity, granulomatous disease, multiple masses/nodules |
| Lung Opacity | 50,574/2,769 | 0.882 | lung opacity, pneumonia, airspace opacity, aspiration |
| No Finding | 73,400/0 | — (single-class) | — |
| Pleural Effusion | 53,324/25,301 | 0.984 | pleural effusion, costophrenic angle blunting, lung opacity, pleural/parenchymal scarring |
| Pleural Other | 1,936/114 | 0.870 | pleural/parenchymal scarring, granulomatous disease, copd/emphysema, mass/nodule (not otherwise specified) |
| Pneumonia | 15,883/22,201 | 0.921 | pneumonia, lung opacity, consolidation, infiltration |
| Pneumothorax | 10,346/41,065 | 0.946 | pneumothorax, enteric tube, lung opacity, picc |
| Support Devices | 65,650/3,278 | 0.885 | enteric tube, endotracheal tube, pigtail catheter, chest tube |