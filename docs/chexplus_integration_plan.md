# CheXpert Plus integration plan for VERA

## Quyết định ngắn gọn

**Chèn CheXpert Plus ở trước Phase 3 như một weak-supervision pretraining stage,
sau khi baseline M3 trên MIMIC/Chest ImaGenome đã chạy ổn, và trước khi đóng băng
M3 để tạo cache cho Phase 4.**

```text
MIMIC-only M3 baseline                 CheXpert Plus preparation
        │                              image + report + 14 labels
        │                                       │
        │                         fixed YOLO + report parser
        │                                       │
        │                         weak 29-region scene graphs
        │                                       │
        └──────────── ablation design ──────────┤
                                                ▼
                              M3 pretrain on CheXpert Plus weak
                                                ▼
                              reset optimizer and scheduler
                                                ▼
                    M3 finetune on MIMIC/Chest ImaGenome clean
                                                ▼
                         calibration + faithfulness gates on MIMIC
                                                ▼
                              freeze M3 → rebuild M4 cache
                                                ▼
                                         train M4 → M5
```

Không trộn CheXpert Plus trực tiếp vào final evaluation và không đưa report của
nó vào inference path.

---

## Vai trò theo từng phase

| Phase | Có dùng CheXpert Plus? | Vai trò |
|---|---:|---|
| M0 preprocessing | Có | Chuẩn hóa ảnh giống MIMIC, giữ patient/study provenance |
| M1 BioViL-T | Có | Chỉ extract frozen features; không cần fine-tune encoder ở v1 |
| M2 detector | Inference only | Dùng YOLO đã học từ ImaGenome để sinh 29 boxes; không pseudo-train lại detector |
| M2 report parser | Có | Report → weak regional concepts/comparison cues |
| **M3** | **Có — vị trí chính** | Weak pretraining rồi clean finetuning |
| M4 | Chưa ở thí nghiệm đầu | Nhận lợi ích gián tiếp qua M3 tốt hơn; temporal pretraining là ablation riêng sau |
| M5 | Không train | Không dùng CheXpert Plus để cho phép thêm claim hoặc tạo proof |
| Evaluation | Không | CheXpert Plus là train/pretrain-only; headline từ human-validated sets |

---

## Thứ tự thực hiện

### Bước 1 — Giữ nguyên baseline sạch

Chạy và lưu M3 `MIMIC-only` trước. Đây là control bắt buộc; nếu thêm CheXpert Plus
ngay từ đầu thì không thể biết dataset mới giúp hay làm nhiễm pipeline.

### Bước 2 — Chuẩn bị CheXpert Plus bằng module đã có

Repo đã có:

- metadata builders trong `preprocess/metadata_generator/`;
- patient-level split trong `preprocess/scene_graph/build_chexplus_splits.py` và
  `resplit_chexplus_by_images.py`;
- YOLO inference và pseudo scene graph trong Phase 2;
- `phase_3/scripts/1-labels.py --dataset chexplus` để chuyển scene graph về tensor
  `[29,69]`, `[29,14]` và image-level labels.

CheXpert Plus phải đi qua đúng preprocessing 448 và frozen BioViL-T extractor như
MIMIC. Không dùng gold/privileged boxes không tồn tại lúc launch.

### Bước 3 — Audit weak labels trước khi train

Không coi toàn bộ pseudo label ngang nhau. Trên một subset được bác sĩ kiểm hoặc
được đối chiếu với nguồn tin cậy hơn, báo:

- precision/recall/F1 per concept;
- coverage của 69 concepts và 29 regions;
- uncertainty/negation error;
- location/laterality error;
- temporal-cue precision;
- tỷ lệ parser conflict hoặc malformed scene graph.

Concept/class có precision thấp phải bị mask hoặc giảm loss weight. Giữ image-level
14 labels chính thức tách biệt với pseudo regional concepts để biết nguồn label.

### Bước 4 — Pretrain M3 trên CheXpert Plus weak

Train đúng architecture định ship (`B-faithful`) để checkpoint tương thích. Trong
pretraining:

- image-level 14-label loss có trọng số bình thường;
- regional/concept loss dùng mask và source-quality weight;
- uncertain/not-mentioned vẫn là `-100`, không ép thành negative;
- không surface concept explanation từ kết quả CheXpert Plus.

### Bước 5 — Finetune kết thúc trên MIMIC/Chest ImaGenome

Khởi tạo model từ CheXpert Plus checkpoint nhưng:

- **reset optimizer**;
- **reset scheduler**;
- fit lại pos-weight từ MIMIC;
- model selection trên MIMIC validation;
- calibration/threshold/concept gate chỉ fit bằng MIMIC validation;
- final metrics trên MIMIC gold/human-validated test và external sets đã chốt.

Không dùng `--resume` hiện tại cho handoff này vì nó nạp cả optimizer/scheduler và
tiếp tục cùng run. Cần một option mới như `--init-ckpt` chỉ nạp `model.state_dict()`.

### Bước 6 — Chỉ sau khi chốt M3 mới chạy lại M4

Nếu M3 có CheXpert Plus thắng ablation và không làm hỏng faithfulness/calibration:

1. chọn checkpoint M3 mới;
2. chạy lại `8-precompute_regions.py` cho toàn bộ MIMIC current/prior;
3. tạo frozen M3 cache mới có tag riêng;
4. retrain M4 từ đầu;
5. chạy lại M4 explainability và M5 proof/report audits.

Không dùng cache M3 cũ với checkpoint M3 mới.

---

## Ablation bắt buộc

| Run | Pretrain | Final finetune | Mục đích |
|---|---|---|---|
| A | Không | MIMIC clean | Baseline chính |
| B | CheXpert Plus weak | MIMIC clean | Đo giá trị thực của CheXpert Plus |
| C, tùy chọn | Không | MIMIC clean với compute tương đương | Kiểm soát lợi ích chỉ do train lâu hơn |

So sánh A và B trên cùng MIMIC test/gold:

- M3 image/region macro-AUC và macro-F1;
- per-concept F1, nhất là concept hiếm;
- ECE/reliability;
- số concept qua explanation gate;
- concept intervention và leakage;
- detector-box robustness;
- sau đó mới xem M4 macro-F1/change-only F1 và M5 report metrics.

Quy tắc quyết định:

- accuracy tăng nhưng calibration/faithfulness giảm: không ship;
- không có khác biệt có ý nghĩa: bỏ để pipeline gọn;
- tăng trên clean MIMIC và explanation gates không giảm: giữ;
- chỉ giúp một nhóm concept: cân nhắc class-selective pretraining thay vì dùng toàn bộ.

---

## Có nên dùng trực tiếp cho M4 không?

CheXpert Plus có nhiều study trên cùng patient và có reports/RadGraph annotations,
nên về kỹ thuật có thể dựng chronological pairs và mine comparison cues. Tuy
nhiên, đây vẫn là weak temporal supervision và mapping prior được nhắc trong report
không nhất thiết trùng hoàn toàn với prior image được chọn bằng metadata.

Vì vậy:

1. không đưa CheXpert Plus vào M4 ngay ở thí nghiệm đầu;
2. trước tiên chứng minh nó giúp M3;
3. nếu muốn mở rộng, làm một ablation riêng:
   `CheXpert Plus temporal pretrain → MIMIC temporal finetune → MS-CXR-T/human eval`;
4. tuyệt đối không báo temporal performance trên pseudo labels CheXpert Plus như
   clinical evidence.

---

## Kết luận

Điểm chèn đúng là:

> **M2 weak-label generation → CheXpert Plus M3 pretraining → MIMIC clean M3
> finetuning → freeze M3 → M4 → M5.**

CheXpert Plus là một phép thử về **cross-institution diversity**, không phải nguồn
ground truth mới. Nó chỉ được giữ trong paper nếu ablation trên clean human-validated
evaluation chứng minh có lợi.

