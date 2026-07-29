# VERA — Kiến trúc & Lộ trình Hiện thực

**VERA: Verifiable, Evidence-grounded Regional Assembly for Faithful Temporal Chest X-ray Reporting**
*(Lắp ráp theo vùng — có kiểm chứng, dựa trên bằng chứng — để sinh báo cáo X-quang ngực theo thời gian một cách trung thực.)*

> Bản cập nhật phản ánh **hiện trạng thực nghiệm** mới nhất: M2 (detector 29 vùng) và M3 (đầu B-faithful) đã có
> kết quả; nhãn ngữ nghĩa lấy trực tiếp từ ImaGenome (bỏ mọi parser/LLM bóc finding); M4 đang hoàn thiện với
> đánh giá ngoài trên MS-CXR-T; M5 = decision-support 2 bảng + verify xác định.

---

## 0. Tóm tắt cho người đọc nhanh

VERA là một hệ thống AI đa phương thức cho ảnh X-quang lồng ngực (CXR) với một mục tiêu khác biệt: **không
chỉ chẩn đoán, mà giải thích được và kiểm chứng được từng câu trong báo cáo**.

Khác biệt cốt lõi nằm ở **cách sinh báo cáo**. Hầu hết hệ thống hiện nay để một mô hình ngôn ngữ (LLM) tự
do "viết" báo cáo từ ảnh — cách này trôi chảy nhưng dễ **bịa** (hallucinate): nói ra bệnh, vị trí, hay diễn
biến mà ảnh không hề chứng minh. VERA lật ngược: báo cáo cuối cùng chỉ là **bản đọc lại (readout) của một
bảng dự đoán có cấu trúc và verify được**. Không có khâu "sinh chẩn đoán tự do". Hệ quả: **ảo giác bị triệt
tiêu ngay từ thiết kế (by construction)**, chứ không phải "giảm bớt" hay "bắt lỗi sau".

Bốn chữ trong tên gánh bốn ý:
- **V**erifiable — mọi mệnh đề trong báo cáo truy ngược được về một ô dữ liệu cụ thể.
- **E**vidence-grounded — nội dung neo vào bằng chứng có cấu trúc, không tự bịa.
- **R**egional — suy luận trên **29 vùng giải phẫu** của lồng ngực.
- **A**ssembly — báo cáo được **lắp ráp** từ output các mô-đun, không sinh ra một cách tự do.

Phụ đề: *Faithful* (trung thực — phản ánh đúng lý do thật của mô hình) và *Temporal* (so sánh ảnh cũ ↔ ảnh
mới để mô tả diễn biến bệnh).

---

## 1. Vấn đề & Động cơ

Trong thực hành lâm sàng, một báo cáo X-quang sai cách nguy hiểm nhất không phải là báo cáo *thiếu trôi
chảy*, mà là báo cáo **nghe rất hợp lý nhưng sai sự thật**. Một bác sĩ ít kinh nghiệm rất dễ bị một câu văn
tự tin dẫn đi sai hướng. Vì vậy VERA ưu tiên **tính trung thực và khả năng kiểm chứng** hơn là độ trôi chảy
ngôn ngữ — và chấp nhận đây là một **điểm vận hành (operating point) có chủ đích vì an toàn người bệnh**.

Hai loại ảo giác mà VERA nhắm tới:
1. **Ảo giác nội dung** — nói ra một bệnh/vị trí/thiết bị mà ảnh không có.
2. **Ảo giác thời gian** — mô tả "bệnh nặng lên / cải thiện" trong khi **không hề có ảnh cũ** để so sánh.

VERA giải quyết cả hai bằng *thiết kế*, không bằng *hậu kiểm*.

---

## 2. Đóng góp chính

1. **Báo cáo = readout của bảng verify được.** Triệt tiêu ảo giác *by construction*: nếu một mệnh đề không
   truy được về một ô trong bảng dự đoán (M3/M4), nó **không thể** xuất hiện trong báo cáo.
2. **Suy luận theo vùng (29 vùng giải phẫu).** Mọi khẳng định bệnh lý gắn với câu trả lời "ở đâu", giúp bác
   sĩ định vị và đối chiếu nhanh.
3. **Giải thích "vì sao" trung thực qua concept (ở M3).** Bệnh hiện tại được suy ra **qua một tầng concept y
   khoa** (nút thắt giải thích); tầng này đã **vượt kiểm định can thiệp (intervention) 100% ở mức cấu trúc** →
   lời giải thích phản ánh đúng lý do thật của mô hình, không phải trang trí.
4. **So sánh thời gian trung thực + định vị "đổi ở đâu" (ở M4).** Diễn biến (cải thiện / ổn định / nặng lên)
   được suy ra từ **ảnh prior thật của chính bệnh nhân**, gắn với **vùng dẫn dắt (lead region) đọc chính xác
   từ logit M4** — trung thực về *"đổi ở đâu"* (không tuyên bố *"vì concept nào"* cho phần thời gian, vì hướng
   concept-thời-gian đã thử và bị loại). Mọi ngôn ngữ thời gian **tự động tắt sạch** khi không có ảnh cũ.
5. **Toàn bộ hệ nghiêng về "glass-box".** Ở những chỗ có thể, VERA chọn giải pháp **xác định, audit được**:
   nhãn finding-theo-vùng lấy **trực tiếp từ ImaGenome** (không để một model tự "bóc" finding ra), và verifier
   ở M5 là **xác định (không phải LLM)** — để mọi bước truy vết lại được.

---

## 3. Kiến trúc tổng thể

VERA gồm **một bước tiền xử lý (M0)** và **năm mô-đun (M1–M5)** chảy nối tiếp nhau:

```
            ẢNH X-QUANG (current) + (tuỳ chọn) ẢNH PRIOR
                              │
        ┌─────────────────────▼──────────────────────┐
        │ M0  TIỀN XỬ LÝ                              │
        │  resize 512 → center-crop 448 · 14 nhãn     │
        │  CheXpert {1/0/-100} · bbox 29 vùng · cặp   │
        │  prior↔current · nhãn tiến triển            │
        └─────────────────────┬──────────────────────┘
                              │
        ┌─────────────────────▼──────────────────────┐
        │ M1  ENCODER  (BioViL-T, đóng băng)          │
        │   ảnh → lưới đặc trưng 196×512 + 1 vector   │
        │   toàn ảnh  →  [197 × 512]                   │
        └───────────┬───────────────────┬─────────────┘
                    │                   │
   ┌────────────────▼───────┐           │
   │ M2  SCENE GRAPH        │           │
   │  (a) Detector 29 vùng  │  bbox     │  lưới đặc trưng
   │  (b) Nhãn finding/vùng │  29 vùng  │
   │      từ ImaGenome silver│          │
   └────────────────┬───────┘           │
                    │  29 bbox          │
        ┌───────────▼───────────────────▼─────────────┐
        │ M3  PHÂN LOẠI THEO VÙNG                      │
        │  attention-pool 196→29 (mask theo bbox)      │
        │  → 69 concept → 14 bệnh CheXpert / vùng      │
        │  + nhánh toàn-ảnh cho finding quan hệ        │
        │  Ra:  region_logit[29,14] · region_feat[29,·]│
        └───────────┬───────────────────┬─────────────┘
        (current)   │                   │  (chạy lại cho ảnh prior)
                    │                   ▼
                    │       ┌───────────────────────────┐
                    │       │ M4  TIẾN TRIỂN THỜI GIAN   │
                    │       │  Siamese current ↔ prior   │
                    │       │  → 29×14×3                 │
                    │       │  {cải thiện/ổn định/nặng}  │
                    │       └───────────┬───────────────┘
                    │                   │
        ┌───────────▼───────────────────▼─────────────┐
        │ M5  LẮP RÁP BÁO CÁO FAITHFUL (6 tầng)        │
        │  bảng M3/M4 → assert/hedge/abstain/omit      │
        │  → realize (template / bảng) → VERIFY        │
        │  (xác định, KHÔNG dùng LLM để verify)        │
        └─────────────────────┬───────────────────────┘
                              ▼
              BÁO CÁO + provenance từng câu
              + bản đồ phủ 29 vùng + change-ledger
```

> **Nguyên tắc bất biến xuyên suốt:** mọi thứ "chỉ có lúc huấn luyện" (report của bác sĩ, scene graph nhãn-
> từ-report) chỉ được dùng làm **nhãn giám sát khi train**. Lúc vận hành thật (inference), đầu vào **chỉ là
> ảnh** → detector → M3 → M4 → M5. Không thành phần nào được phụ thuộc một input "đôi khi vắng mặt".

---

## 4. M0 — Tiền xử lý dữ liệu

M0 chuẩn hoá ba thứ về một dạng đồng nhất: **ảnh**, **hộp giới hạn vùng (bbox)**, và **nhãn**.

### 4.1 Chuẩn hoá ảnh (geometry duy nhất, dùng chung mọi nơi)
- **Bước 1 — Resize:** thu/phóng ảnh sao cho **cạnh ngắn = 512 px** (giữ nguyên tỷ lệ, nội suy BILINEAR).
- **Bước 2 — Center-crop:** cắt **448 × 448** ở chính giữa. Nếu một cạnh ngắn hơn 448 (hiếm) thì đệm đen.
- **Vì sao crop vuông 448:** đồng nhất kích thước đầu vào, khớp với encoder M1, và loại bớt viền nhiễu.
- Geometry này là **một chuẩn duy nhất** áp cho *mọi* ảnh và *mọi* dataset — ảnh current và ảnh prior phải
  qua **đúng cùng một phép biến đổi**, nếu không phép so sánh thời gian ở M4 sẽ vô nghĩa.

### 4.2 Toạ độ hộp giới hạn (bbox) của 29 vùng
- Bbox giải phẫu từ Chest ImaGenome được **rescale qua đúng geometry trên** → toạ độ trong không gian crop
  448 × 448.
- Hộp nào bị đẩy **hoàn toàn ra ngoài** khung crop → quy về **sentinel `(0,0,0,0)`**; toàn bộ pipeline phía
  sau **lọc bỏ** sentinel này (vùng đó coi như "không đánh giá được" trên ảnh đã crop).

### 4.3 Nhãn bệnh — quy ước 3 trạng thái (RẤT QUAN TRỌNG)
14 lớp bệnh theo chuẩn **CheXpert** (thứ tự cố định, *No Finding* ở vị trí 8). Mỗi nhãn nhận **một trong ba**
giá trị:

| Giá trị | Ý nghĩa | Cách dùng khi train |
|--------:|---------|---------------------|
| `1` | **Dương tính** (report khẳng định có) | tính loss bình thường |
| `0` | **Âm tính** (report khẳng định không) | tính loss bình thường |
| `-100` | **Không rõ / không nhắc tới** | **bị mask — KHÔNG đưa vào loss** |

- **Chính sách "uncertain → unknown" (chủ đích):** nhãn **uncertain** (`-1`) của CheXpert được **quy về
  `-100`**, tức gộp chung với "không nhắc tới". Đây là lựa chọn **U-Ignore** có chủ ý: ta **không** ép mô
  hình học một câu trả lời cứng cho những trường hợp bản thân report cũng mơ hồ.
- **Tuyệt đối KHÔNG gộp `-100 → 0`.** Coi "không nhắc tới" = "âm tính" sẽ ép mô hình học âm tính giả cho rất
  nhiều nhãn (đặc biệt với các report không đề cập đủ 14 lớp) → thiên lệch âm tính + nhiễu lớp hiếm.
- Hệ quả kỹ thuật: hàm mất mát là **masked BCE** — chỉ tính trên các ô `1`/`0`, bỏ qua mọi ô `-100`.

### 4.4 Ghép cặp thời gian & nhãn tiến triển
- **Ghép cặp prior ↔ current:** theo cùng bệnh nhân, sắp theo thời điểm chụp; ảnh cũ là *prior*, ảnh mới là
  *current*. Ảnh đầu tiên của một bệnh nhân không có prior → **không tạo cặp** (sẽ chảy xuống tầng "tắt ngôn
  ngữ thời gian" ở M5, đây là hành vi đúng, không phải lỗi dữ liệu).
- **Nhãn tiến triển (cho M4):** lấy từ **`comparison_cues`** của Chest ImaGenome — tín hiệu so sánh mà NLP đã
  bóc tách sẵn từ report. Chỉ tồn tại đúng **ba giá trị**, ánh xạ 1–1 sang ba lớp tiến triển:
  `no change → ổn định` · `improved → cải thiện` · `worsened → nặng lên`.
- **Cổng giám sát:** một ô `(vùng, bệnh)` chỉ được tính khi có cue **và** vùng hiện diện ở **cả** ảnh current
  lẫn prior. Trong tensor tiến triển `[N, 29, 14]`, ~98.9% ô là `-100` (không cue → mask).

### 4.5 Thống kê dữ liệu (tham khảo)
| Nguồn | Ảnh | Bệnh nhân | Có report? | Có scene graph (bbox + finding)? |
|-------|----:|----------:|:----------:|:--------------------------------:|
| **MIMIC-CXR + ImaGenome** | 222,168 | 63,334 | có | **có** (silver toàn bộ + gold người-gán 833 ảnh) |
| **CheXplus** | 191,046 | 64,686 | có | sinh từ report (nhãn yếu) |

- **Cặp thời gian (MIMIC):** 253,306 cặp prior↔current; số cặp **thực sự dùng cho M4** (có cue + vùng hiện
  diện cả hai) = **93,472**.
- **Phân bố lớp tiến triển (mức ô, present-masked):** ổn định 45.1% · cải thiện 19.4% · nặng lên 35.5%.
  "Đổi trạng thái" (cải thiện + nặng) chiếm **54.9%** → phải đọc **change-only F1**, không đọc accuracy.

---

## 5. M1 — Encoder (BioViL-T, đóng băng)

- **Vai trò:** chuyển mỗi ảnh CXR thành biểu diễn vector không gian để các mô-đun sau khai thác.
- **Backbone:** **BioViL-T** — một encoder thị giác-ngôn ngữ chuyên cho ảnh y khoa ngực, **được đóng băng**
  (frozen): VERA không huấn luyện lại nó, chỉ dùng nó như một bộ trích đặc trưng ổn định.
- **Đầu ra cho mỗi ảnh:**
  - một **lưới đặc trưng không gian 196 × 512** (tương ứng lưới 14 × 14 ô, mỗi ô là vector 512 chiều) —
    "ảnh nhìn thấy gì, ở đâu";
  - một **vector toàn ảnh 512 chiều** — tóm tắt ngữ cảnh toàn cục.
  - Gộp lại: tensor `[197 × 512]` cho mỗi ảnh.
- Vì M1 đóng băng nên đặc trưng của mỗi ảnh là **xác định (deterministic)** → được **tính trước & lưu cache
  một lần**, dùng lại cho cả M3 và M4. Nhờ đó chi phí train M3/M4 rất rẻ (head nhẹ, đọc feature từ cache).

---

## 6. M2 — Scene Graph (bản đồ giải phẫu — ngữ nghĩa)

M2 dựng một "bản đồ cảnh" của lồng ngực, gồm **hai nhánh độc lập**:

### 6.1 Nhánh không gian — Detector 29 vùng
- Một detector (họ YOLO) được tinh chỉnh để **khoanh 29 vùng giải phẫu** trên ảnh CXR (phổi, thuỳ phổi,
  trung thất, tim, góc sườn hoành, khí quản, carina, cơ hoành, xương đòn, cột sống, cung động mạch chủ…).
- 29 hộp này chính là **mặt nạ định vị** để M3 gom đặc trưng theo vùng. **Đây là điểm khớp giữa M2 và M3.**
- **Kết quả:** mAP50 **0.931**, mAP50-95 **0.694** (val). Detector **vượt hẳn baseline "đặt hộp trung bình"**
  (+0.38 IoU toàn cục, +0.45–0.55 ở các landmark nhỏ), và **thêm giá trị nhiều nhất ở đúng ca giải phẫu bất
  thường** — bằng chứng nó *đọc nội dung ảnh*, không đoán theo vị trí trung bình.
- **Quy ước quan trọng:** dùng **hộp do detector dự đoán** cho cả lúc train lẫn lúc infer M3 (để phân phối dữ
  liệu khớp nhau). Hộp "gold" người-gán chỉ dùng để huấn luyện detector. *(Đối chứng ở M3 cho thấy dùng hộp
  gold chỉ hơn ~0.004 AUC → hộp detector là đủ.)*

### 6.2 Nhánh ngữ nghĩa — nhãn finding theo vùng (từ ImaGenome silver)
- Nhãn giám sát cho M3/M4 (bệnh/finding/tiến triển **theo từng vùng**, trên 69 concept + `comparison_cues`)
  đến **trực tiếp từ scene graph silver của Chest ImaGenome** — bộ này đã **phủ toàn bộ MIMIC train** nên
  không cần một bộ phân tích report riêng.
- **Không dùng model để "bóc" finding.** Ta **không** dùng LLM và cũng **không** dùng parser luật để suy
  finding từ report — nhãn đã có sẵn từ ImaGenome cho MIMIC. Đây là lựa chọn glass-box: nhãn train truy được
  về một nguồn cố định, không qua một mô hình có thể bịa.
- **Vai trò khi vận hành:** nhãn finding chỉ tồn tại **lúc huấn luyện**. Lúc vận hành thật, M3/M4 **thuần ảnh**
  — không cần report của ảnh current.

---

## 7. M3 — Phân loại bệnh theo vùng

**Đầu vào:** lưới đặc trưng `196×512` (từ M1) + 29 bbox (từ M2). **Đầu ra:** bảng bệnh-theo-vùng + các tín
hiệu định vị/giải thích chảy xuống M4/M5.

### 7.1 Attention-pool: gom 196 ô đặc trưng về 29 vùng
- Mỗi vùng có một **truy vấn học được (learned query)**; truy vấn này "chú ý" (attend) lên 196 ô đặc trưng,
  nhưng **bị mask theo bbox của vùng đó** (ô ngoài hộp bị triệt). Kết quả: một vector cho mỗi vùng → `29×512`.
- **Vì sao attention-pool thay vì lấy trung bình hộp:** (a) cứu được các **tổn thương nhỏ, khu trú** (lấy
  trung bình sẽ làm loãng tín hiệu nhỏ trong một hộp lớn); (b) trọng số chú ý `α` **chính là** tín hiệu định
  vị *trung thực* "mô hình lấy tín hiệu từ chỗ nào trong vùng" — không phải bản đồ nhiệt hậu kiểm.
- **Giữ nguyên cấu trúc 29 vùng** xuyên suốt — tuyệt đối không "ép phẳng" về mức toàn ảnh (sẽ phá M4 và M5).
- Đặc trưng vùng được giữ **giàu (512 chiều)** làm biểu diễn chia sẻ cho cả head bệnh lẫn M4 (không nén qua
  "neck").

### 7.2 Tầng concept → tầng bệnh (nút thắt giải thích)
- Từ đặc trưng vùng, M3 dự đoán **69 concept y khoa** (43 finding giải phẫu + 10 bệnh + 12 ống/đường truyền +
  4 thiết bị), rồi từ đó ra **14 nhãn bệnh CheXpert cho mỗi vùng**.
- 69 concept đóng vai một **"nút thắt giải thích"**: bệnh được suy ra **qua** concept, nên ta nói được "bệnh d
  *vì* concept c". Head concept→bệnh là **masked + non-negative** (chỉ cho phép các concept liên quan y khoa
  đẩy bệnh lên, theo một bảng ánh xạ concept→CheXpert) → biến "nút thắt" thành **cấu trúc thật**, không phải
  hình thức.

### 7.3 Ba hướng head & kết quả kiểm định *faithfulness*
M3 hỗ trợ **ba hướng**, khác nhau **không chỉ ở độ chính xác mà ở việc "giải thích bằng concept có trung
thực không"**:

| Hướng | Đường dự đoán bệnh | Tính trung thực | Vai trò |
|------|--------------------|-----------------|---------|
| **A — Direct** | đặc trưng vùng → bệnh | **where-faithful** vô điều kiện (giải thích = "ở đâu") | fallback an toàn |
| **B — Concept Bottleneck (faithful)** | bệnh **chỉ** qua 69 concept, head masked non-negative | **why-faithful** nếu qua kiểm định | **cấu hình đang ship** |
| **C — Hybrid** | concept ⊕ đặc trưng ảnh | rủi ro **rò rỉ** (concept có thể chỉ trang trí) | chỉ để đối chứng |

- **Quy tắc quyết định theo faithfulness, KHÔNG theo accuracy:** hướng B chỉ được tuyên bố "vì sao" trung thực
  nếu qua **concept-intervention test** (can thiệp bật/tắt concept → dự đoán bệnh đổi đúng hướng); hướng C phải
  qua **leakage test** (xoá kênh concept mà accuracy gần như không đổi ⇒ concept chỉ trang trí ⇒ loại khỏi vai
  "vì sao").
- **Kết quả (đã đo):** đầu **B-faithful** đạt **concept-intervention 100% đúng hướng** và **concept-F1 ~0.89**
  → **"vì sao" bằng concept được phép dùng**. Accuracy gần như ngang hướng A (feature-bound), nên ta chọn
  B-faithful làm cấu hình ship — vừa giữ độ chính xác, vừa có kênh giải thích trung thực.
- **Lưu ý trung thực (caveat):** bảng ánh xạ *concept→CheXpert* là **do nhóm tự xây (curated), chưa phải chuẩn
  chính thức** → "faithful" (phản ánh đúng lý do model) **không đồng nghĩa** "valid" (đúng y khoa tuyệt đối);
  cần **bác sĩ thẩm định** bảng này và trích dẫn nguồn khi viết bài.

### 7.4 Nhánh toàn-ảnh cho finding *quan hệ* (để riêng)
- Vài finding mang tính **quan hệ/toàn cục**, không nằm gọn trong một hộp: tim to (cardiomegaly), phù lan
  toả, thể tích phổi thấp. Chúng đi qua một **head toàn-ảnh riêng**.
- **Cách trình bày (đã chốt):** **không** cố nhồi các finding này vào một vùng cụ thể và **không** trộn/gate
  vào đường-vùng. Trong báo cáo, chúng được **dán nhãn `grounding = global`** — tức trung thực rằng đây là
  bệnh *đánh giá trên toàn ảnh, không chắc nằm ở vùng nào*, thay vì gán bừa một toạ độ giả.
- Đối chứng cho thấy **bỏ nhánh global làm tụt AUC ~0.018** → nhánh này đáng giữ.

### 7.5 Xử lý mất cân bằng & metric
- Dữ liệu lệch nặng (lớp hiếm rất ít dương tính) → dùng **pos_weight log-scale** (kiểu RADAR) / focal loss cho
  mỗi số hạng BCE; **tinh chỉnh ngưỡng per-disease** trên val (nối thẳng vào calibration/abstention của M5).
- **Metric chuẩn = macro-F1 + per-class + AUC.** **Không** dùng accuracy (bị lớp đa số kéo lệch).

### 7.6 Lưu ý kiến trúc
- **Head có thể hoán đổi MLP ↔ KAN:** mặc định là **MLP** (nhìn chung ngang-hoặc-hơn KAN trên task này, ổn
  định hơn, nhẹ hơn); biến thể **KAN** (Kolmogorov-Arnold Network) giữ ở *tầng suy luận có cấu trúc* như một
  đóng góp/ablation — đổi một cờ là so sánh được.

---

## 8. M4 — Tiến triển bệnh theo thời gian

**Mục tiêu:** với mỗi `(vùng, bệnh)`, xác định diễn biến giữa ảnh prior và ảnh current: **cải thiện / ổn
định / nặng lên** → tensor `29 × 14 × 3`.

### 8.1 Cấu trúc Siamese (chia sẻ trọng số)
- "Siamese" = **một nhánh dùng chung trọng số**, chạy **hai lần** — một cho ảnh current, một cho ảnh prior —
  để hai ảnh nằm **cùng một không gian biểu diễn**, so sánh mới có nghĩa.
- Nhánh chia sẻ đó **chính là** đường M1 → attention-pool của M3 (đã đóng băng sau khi train M3). Vì đặc trưng
  vùng xác định nên được **tính trước & cache**; M4 chỉ **tra cache**. ⇒ Siamese gần như *miễn phí* về kiến
  trúc, và giữ được **α/định vị của M3 nguyên vẹn** (M4 không làm hỏng tính faithful của M3).

### 8.2 Đầu vào head — giữ cả hai vế lẫn hiệu
Với mỗi vùng, head tiến triển nhận đặc trưng hai ảnh **và cả hiệu của chúng**, kèm logit bệnh hai thời điểm:

```
[ feat_curr ; feat_prior ; (feat_curr − feat_prior) ]   +   [ logit_curr(14) ; logit_prior(14) ]
```

- Giữ **cả hai vế lẫn hiệu** (không chỉ lấy phép trừ) để head **tự học** cách so sánh, không ép sẵn dấu trừ.
  Tín hiệu cốt lõi là **hiệu đặc trưng `curr − prior`** — chênh lệch tường minh giữa hai thời điểm.
- **Kiến trúc đang ship = "temporal fusion + M3-delta":** patch của current *chú ý chéo* (cross-attention)
  sang patch của prior → gộp theo vùng → head; đồng thời **nối thêm logit bệnh current/prior/hiệu từ M3** để
  head tận dụng tín hiệu bệnh đã hiệu chỉnh thay vì học lại từ đầu.

### 8.3 Tăng cường dữ liệu bằng đảo thời gian (time-flip) + ràng buộc nhất quán
- **Time-flip augment:** **hoán đổi thứ tự hai ảnh** (prior ↔ current) **và đảo nhãn tương ứng** (*cải thiện ↔
  nặng lên*; *ổn định* giữ nguyên) — nhân đôi tín hiệu hướng, buộc mô hình học diễn biến đối xứng. *Loại trừ
  lớp **Support Devices*** (gắn/rút thiết bị không đối xứng theo thời gian).
- **Ràng buộc nhất quán khi đảo (flip-consistency):** phạt để `P(cải thiện | curr,prior) ≈ P(nặng | prior,
  curr)` và `P(ổn định)` bất biến — nhắm thẳng vào việc học **hướng** thay vì chỉ phát hiện bất thường hai lần.

### 8.4 Giải thích M4 = "đổi ở đâu" (lead-region, exact-by-construction)
- Đây là **đóng góp giải thích chính của M4**, và là *where-faithful* (không phải *why-concept*). Khi một bệnh
  đổi trạng thái và trải nhiều vùng, ta nêu **vùng dẫn dắt (lead region)** vào quyết định gộp đó.
- **Cơ chế chính xác:** call mức-bệnh = gộp các vùng bằng **log-sum-exp (LSE)**; với LSE, trọng số đóng góp
  của vùng `r` **đúng bằng `softmax(logit_r)`** → attribution share exact, cộng lại = 100%, **không post-hoc,
  không đoán**. Lead region = vùng có share cao nhất.
- **Luật abstain:** chỉ nêu lead region khi một vùng trội rõ (`share_top1 − share_top2 ≥ δ` và đủ sàn); nếu
  đóng góp **trải mỏng nhiều vùng** → gọi "multifocal/diffuse", không ép chọn. Tất cả **tất định**.
- **Không tuyên bố "vì concept nào" cho phần thời gian:** hướng *concept-delta bottleneck* (bắt tiến triển đi
  qua thay đổi concept) **đã thử và bị loại** — nó sụp về "ổn định" (change-F1 ~0.4) vì giả định "concept tăng
  = bệnh nặng" không đúng y khoa. Ghi lại như **negative result có nguyên tắc**; M4 chỉ trung thực ở mức "đổi ở
  đâu".

### 8.5 Đầu ra, mất cân bằng, an toàn thứ tự (ordinal safety) & kết quả
- Ra `29×14×3`. Chỉ giám sát ô có cue và vùng hiện diện ở **cả** hai ảnh; ô khác để mask.
- Lớp **"ổn định" áp đảo** → class-weight / label-smoothing. **Metric = macro-F1 + change-only F1** + **bộ an
  toàn thứ tự**: **% lỗi ngược hướng** (dự đoán cải thiện khi thật ra nặng, và ngược lại — mục tiêu ≈ 0) và
  **QWK** (phạt lỗi "cách 2" gấp 4× lỗi kề "ổn định"). Accuracy ≈ tỉ lệ "ổn định" là **cờ đỏ**.
- **Lưu ý loss:** loss thuần "phạt theo khoảng cách thứ tự" (CDW) **làm mô hình co về lớp giữa "ổn định"** — một
  failure mode đã có tên trong y văn. Cách dùng đúng là **lai `CE + λ·(phạt hướng/khoảng cách)`** rồi báo cáo
  cả **đường cong an toàn–độ nhạy** (frontier), chọn điểm vận hành, không tối ưu mù một con số.
- **Kết quả hiện tại:** trên **MS-CXR-T (nhãn người, đánh giá ngoài)** đạt **accuracy ~0.64** — vượt backbone
  BioViL-T (0.602) và **sát SOTA CoCa-CXR (0.650)**, dù M4 là **zero-shot** trên MS-CXR-T (không train trên
  đó). Trên silver test: change-only F1 ~**0.58**. Lớp **"cải thiện" khó nhất** (~0.50–0.53); "nặng lên" dễ hơn.

### 8.6 Dự phòng cho vận hành: cờ "prior có report" & khả năng hai bộ trọng số
Lúc vận hành thật, đầu-prior **bất đối xứng**: ảnh prior **có khi đã kèm report** (lần khám cũ đã được đọc), có
khi **chỉ có ảnh**. Hai chế độ này cho M4 chất lượng đầu-prior khác hẳn nhau, nên M4 khả năng cần:
- **Một cờ `prior_report_available`** nối vào đầu vào, để mô hình **biết mình đang ở chế độ nào** thay vì tự
  đoán.
- **Khả năng hai bộ trọng số** (hoặc một bộ huấn luyện kiểu *modality-dropout* ở đầu-prior để chịu được cả hai
  chế độ). Lý do: một M4 chỉ quen đầu-prior-thuần-ảnh có thể đã ngầm học **bù trừ** cho nhiễu đó; nạp đột ngột
  một đầu-prior *sạch* (từ report) sẽ khiến nó **bù nhầm** → có thể *tệ hơn* dù input "tốt hơn". Muốn dùng
  report-prior để nâng chất lượng thì **M4 phải được train cho biết chế độ đó**.

> Đây là phần **dự phòng/mở rộng**, gắn với nhánh prior-report ở **mục 12**; chỉ chạm **đầu-prior** (kênh
> `logit_prior`), không đụng đầu-current (current khi launch không bao giờ có report).

---

## 9. M5 — Lắp ráp báo cáo trung thực (decision-support **table-first**)

M5 biến bảng dự đoán M3 (bệnh theo vùng) + M4 (tiến triển) thành một sản phẩm hỗ trợ quyết định **là bản đọc
lại của bảng — KHÔNG sinh chẩn đoán nào mới**. Phần lớn xác định (deterministic), chạy trên CPU. Định vị: đây
là công cụ để bác sĩ **soi nhanh + tự verify**, không phải "AI viết report thay bác sĩ".

Hai con số mục tiêu: **out-of-table ≈ 0** (không có finding nào lọt ra ngoài bảng) và **temporal-halluc = 0**
(không có ngôn ngữ thời gian khi thiếu prior).

### 9.0 Sản phẩm chính = HAI BẢNG (văn xuôi là tùy chọn)
Đơn vị chân lý duy nhất = **một ô nguyên tử `(vùng, bệnh)`**; mọi bảng/câu đều là *view* của nó. Bác sĩ thấy
hai bảng:

- **Bảng 1 — HIỆN CÓ GÌ (theo vùng).** Mỗi dòng = một ô `(vùng, bệnh)` đáng nói, kèm cột **Evidence (concepts)**
  (các finding model thấy ở vùng đó — bằng chứng *mô tả* để bác sĩ soi ảnh, **không** suy severity) và **Conf**
  (độ tin đã calibrate, tô màu theo band assert/hedge). Luôn hiển thị.
- **Bảng 2 — CÁI GÌ ĐỔI (theo bệnh).** Mỗi dòng = một bệnh có diễn biến, cột **Change** {nặng lên · cải thiện ·
  ổn định · **mới (new)** · **hết (resolved)**}, **Region(s)** liên quan, **Lead region** (§8.4), **Conf**.
  new/resolved suy tất định từ "vùng hiện diện ở current XOR prior". **Không có prior ⇒ không có bảng này.**
- **Lớp 3 (drill-down):** bấm một ô → hiện provenance đầy đủ (concept prior→current, band, dấu verify).

**Văn xuôi = render tất định của hai bảng** (mặc định template; LLM constrained-paraphrase chỉ là tùy chọn).
Khối verify và drill-down chạy ngầm/khi cần, **không** nhồi vào báo cáo chính.

### 9.1 Sáu tầng theo độ tin cậy (cơ chế bên trong dựng nên hai bảng)
| Tầng | Nội dung |
|------|----------|
| **1 — Lõi cấu trúc** | Từ `region_logit[29,14]` (M3) + tiến triển `29×14×3` (M4), áp ngưỡng → mỗi finding nhận **assert / hedge / abstain / omit**. Mọi câu về sau phải truy về một mục ở đây. |
| **2 — Định vị "ở đâu"** | Vùng dẫn dắt cho mỗi bệnh (chính xác, faithful, đọc từ softmax theo vùng) + trọng số `α` (tín hiệu nội vùng) + **bản đồ phủ 29 vùng**. Finding quan hệ được dán `grounding = global`. |
| **3 — Hiệu chỉnh độ tin (calibration) + abstain** | Temperature scaling theo từng lớp; đặt ngưỡng `τ`. Dưới ngưỡng → **hedge** (ngôn ngữ dè dặt) hoặc **abstain** ("nhường radiologist"). Không chắc thì nói không chắc — đó là một phần của trung thực. |
| **4 — Cổng thời gian (rủi ro #1)** | **Không có prior ⇒ TẮT SẠCH mọi ngôn ngữ thời gian** (không tồn tại đường code nào sinh ra từ thời gian). **Có prior ⇒ cụm tiến triển = đọc thẳng argmax của M4 (1:1)**, không để mô hình tự "đoán diễn biến". |
| **5 — Hiện thực hoá** | Sản phẩm chính là **hai bảng** (§9.0). Văn xuôi là *view thứ hai*: **template** (mặc định, tất định) hoặc **paraphraser có ràng buộc** (LLM viết mượt nhưng *prose-from-table*: chỉ diễn đạt lại finding trong bảng, **cấm thêm/bớt**, phải qua tầng 6). |
| **6 — Kiểm chứng (verify)** | **Round-trip:** trích lại nhãn từ báo cáo đã sinh → **đối chiếu với bảng M3/M4** → loại mọi finding lọt ra ngoài (out-of-table). **Luật phủ (coverage):** mọi ô assert-dương phải được xử lý; thiếu ô nào → bật cờ (bắt lỗi bỏ-sót). |

### 9.2 Hai ràng buộc bất biến của M5
- **Verifier luôn xác định, KHÔNG bao giờ là LLM.** (Một LLM verifier sẽ tự bịa, phá vỡ chính mục đích.) Đây
  là chỗ sau này cắm CheXbert/RadGraph vào (giữ nguyên kiểu trả về).
- **Temporal-halluc = 0 by construction:** một cụm tiến triển chỉ được phát ra khi tồn tại ô M4 cho ảnh đó.
  Không prior → không có ô M4 → **không có đường code** nào sinh ra từ ngữ thời gian.

### 9.3 Định dạng đầu ra & trực quan hoá
- **Provenance từng câu** — mỗi mệnh đề trỏ về ô nguồn `(vùng, bệnh, độ tin, tiến triển M4)`; câu nào không
  truy được → tô đỏ (vừa là viz, vừa là cơ chế bắt ảo giác).
- **Bản đồ phủ 29 vùng** (bình thường / bất thường / không chắc / không-đánh-giá-được) — biến "sự im lặng"
  thành một khẳng định verify được.
- **Change-ledger** prior→current (finding | prior | current | hướng) — đọc thẳng argmax M4.

---

## 10. Dữ liệu & phân vai (data provenance)

Nguyên tắc bất di bất dịch: **nhãn yếu (do mô hình khác sinh) chỉ được dùng để TRAIN/PRETRAIN, TUYỆT ĐỐI
không dùng để đánh giá.** Mọi con số trong bảng kết quả đến từ **nhãn người-gán**.

| Dataset | Vai trò huấn luyện | Vai trò đánh giá |
|---------|--------------------|------------------|
| **MIMIC + ImaGenome** | trục chính: train M3 + M4 (nhãn tiến triển từ `comparison_cues`) | **eval chính** (trên phần người-gán / gold) |
| **CheXplus** | pretrain/augment (scene graph yếu) — giá trị là **đa dạng phân phối**, quyết giữ/bỏ bằng ablation | **eval-never** |

- **Nhãn CheXplus** lấy sẵn từ bộ dữ liệu gốc, chỉ **sắp lại đúng thứ tự 14 lớp như MIMIC**.
- **Vòng lặp nhãn yếu cần cảnh giác:** nhãn yếu cho CheXplus rồi train M3/M4 là một dạng *chưng cất ngầm*; rủi
  ro là lỗi của bộ sinh nhãn *chính là ảo giác* nên có thể dạy M3/M4 ảo giác theo. Cách kiểm soát: giữ lằn
  ranh "weak → train, người-gán → eval"; cân nhắc **pretrain trên CheXplus-yếu rồi finetune kết thúc trên
  MIMIC-sạch** để dữ liệu sạch có "tiếng nói cuối".

---

## 11. Đánh giá & Kiểm chứng tính trung thực

VERA tách bạch **ba nhóm metric** và chủ động **sở hữu trade-off** (hi sinh fluency lấy faithfulness):

**A. Hiệu quả lâm sàng (đúng/sai):**
- **M3 (14 bệnh):** dẫn **AUC** (SOTA phân loại CXR dùng AUC; F1@0.5 bị lạm phát do mất cân bằng), kèm
  macro-F1 + per-class.
- **M4 (tiến triển):** **change-only F1** + **bộ an toàn thứ tự** (% lỗi ngược hướng ≈ 0, QWK); trên MS-CXR-T
  dẫn **accuracy** (để so công bằng với SOTA). Accuracy ≈ tỉ lệ "ổn định" là cờ đỏ.
- **Báo cáo (M5):** **CheXbert-F1** + **RadGraph-F1** (đúng nội dung lâm sàng) + **Temporal-F1** (câu diễn biến
  đúng hướng) + **RadFact** (grounding đúng vùng). **KHÔNG** để BLEU/ROUGE làm headline (template thua LLM văn
  hoa ở mấy metric vô nghĩa lâm sàng đó).

**B. Tính trung thực (faithfulness) — trục bán hàng của VERA:**
- **out-of-table rate** & **temporal-halluc rate** — đo bằng round-trip verify (tầng 6). Mục tiêu ≈ 0.
- **Kiểm chứng concept:** F1 "concept-từ-ảnh" + **intervention test** (đã đạt 100% đúng hướng cho đầu
  B-faithful → "why" được phép), và **leakage test** cho hướng C.
- **Calibration thực chứng:** **reliability diagram + ECE** theo per-class, đặc biệt soi **lớp hiếm**; lớp nào
  calibrate kém thì mặc định **hedge**. (Đã có công cụ xuất ECE + reliability + ngưỡng per-disease.)
- **Đường cong deletion/insertion mức vùng** *(bằng chứng phụ, đang kế hoạch)*: xoá/chèn dần đặc trưng vùng
  theo độ quan trọng để đo độ faithful của tín hiệu định vị. Kết quả nhạy với "giá trị thay thế" (zero / mean /
  noise) → phải nêu rõ baseline và kiểm độ bền với ≥1 baseline khác. Đây là *cần-nhưng-không-đủ*; bằng chứng
  faithfulness **chính** vẫn là cấp-construction (out-of-table, temporal-guard).

**C. Độ trôi chảy (fluency):**
- CheXbert-F1 / BLEU… — báo cáo *cùng* hai nhóm trên, kèm lập luận VERA là **điểm vận hành cố ý** vì an toàn.

**Đánh giá thời gian bằng nhãn người (MS-CXR-T):**
- Nhãn tiến triển dùng để *train* M4 đến từ `comparison_cues` (nhãn yếu) → **không** đủ để *đánh giá*. VERA
  dùng **MS-CXR-T** (1,326 cặp study có nhãn người, 5 finding: Consolidation, Edema, Pleural Effusion,
  Pneumonia, Pneumothorax; lớp Improving/Stable/Worsening) làm **kiểm định ngoài**. Kết quả **zero-shot**
  (M4 không train trên MS-CXR-T): **accuracy ~0.64** — vượt BioViL-T 0.602, sát CoCa-CXR 0.650. *Lưu ý:*
  MS-CXR-T là **mức ảnh** còn M4 là **mức vùng**, nên đây là audit qua một phép gộp tường minh (`logsumexp`),
  chưa phải điểm định vị thời gian mức vùng; và **còn một quyết định mở**: coi MS-CXR-T là *tập test cuối* hay
  chỉ *tập hiệu chỉnh/audit*.

**Các ablation/đối chứng đáng giá:**
- **KAN vs MLP** (parity head) · **có/không CheXplus** (đo trên test MIMIC sạch) · **template vs
  constrained-paraphrase** (đo cả fluency lẫn out-of-table) · **so sánh faithfulness xuyên paradigm** (chạy
  round-trip trên *output baseline sinh tự do* để ước lượng out-of-table/temporal-halluc của chúng).
- *(Tuỳ chọn)* **reader study nhỏ** với bác sĩ ít kinh nghiệm (faithful vs free-prose, đo tỉ lệ bị dẫn sai) —
  biến động cơ an toàn thành bằng chứng; nếu không làm thì hạ xuống *motivation*, không over-claim.
- *(Tuỳ chọn)* kiểm định **độ ổn định thống kê** (bootstrap khoảng tin cậy — đã có công cụ; cross-validation
  cân nhắc) cho các số chính.

> **Tiền lệ cùng lab — OsteoGA:** dùng đúng cấu trúc difference-feature/Siamese, nhưng ảnh thứ hai của họ là
> **counterfactual do GAN bịa** (không verify được). Ảnh thứ hai của VERA-M4 là **prior thật của bệnh nhân**
> (verify được) ⇒ VERA là *phiên bản faithful* của ý tưởng difference-feature đó.

---

## 12. Hạn chế đã biết & Hướng phát triển

VERA v1 cố ý giữ **phạm vi hẹp mà sạch** ("một câu chuyện không dấu hoa thị"). Các hướng dưới đây *có giá
trị* nhưng được **tách riêng** để không làm loãng luận điểm cốt lõi:

1. **Chốt tập đánh giá thời gian người-gán (ưu tiên cao nhất).** MS-CXR-T đã được tích hợp và audit, nhưng cần
   quyết dứt khoát nó là *test cuối* hay chỉ *audit/hiệu chỉnh*, và xử lý khoảng cách "mức ảnh vs mức vùng".
   Toàn bộ claim "temporal faithful" đứng hay sụp ở đây.
2. **Thẩm định y khoa bảng concept→CheXpert.** Đầu B-faithful cho "vì sao" trung thực, nhưng bảng ánh xạ hiện
   do nhóm tự xây → cần bác sĩ thẩm định + trích nguồn để "faithful" đi kèm "valid".
3. **Nâng lớp "cải thiện" của M4.** Đòn bẩy: loss ưu tiên **hướng** (improved vs worsened) tách khỏi
   stable-vs-change; lấy mẫu theo lớp hiếm; lọc chất lượng cặp (cùng tư thế, khoảng thời gian hợp lý).
4. **Tận dụng report của ảnh prior (bất đối xứng thông tin).** Lúc vận hành, ảnh prior **thường đã có report**
   → trích trạng thái prior sạch (cần một bộ trích report, **ngoài phạm vi v1**) để nâng độ chính xác M4 phía
   prior (xem §8.6). Đây là **nhánh mở rộng riêng** (có cờ + chế độ train hai luồng + ablation bắt buộc),
   **không** trộn vào v1 thuần-ảnh.
5. **Chưng cất từ thông tin đặc quyền (privileged information).** Hấp dẫn nhưng có bẫy: nếu teacher kết luận
   nhờ *đọc report* mà ảnh không đủ tín hiệu, sẽ dạy student "nói X mà không nhìn thấy X" — ảo giác bị chưng
   cất vào trọng số. ⇒ Tách thành **study riêng**, cổng gác "chỉ chưng cất finding chứng minh được từ ảnh".

---

## 13. Kết quả hiện tại (ảnh chụp nhanh)

| Thành phần | Cấu hình | Chỉ số chính |
|-----------|----------|--------------|
| **M2 detector** | YOLO 29 vùng, ảnh 448 | mAP50 **0.931** · mAP50-95 **0.694**; vượt static-prior +0.38 IoU |
| **M2 nhãn ngữ nghĩa** | ImaGenome silver (không parser) | phủ toàn bộ MIMIC train; nguồn nhãn finding/tiến triển theo vùng |
| **M3** | `B-faithful` (concept bottleneck, detector-box, global head) | test **AUC 0.829** · image-F1 0.883 · region-F1 0.863 · concept-F1 0.890; **intervention 100% → why-faithful (PASS)** |
| **M4** | temporal-fusion + M3-delta | **MS-CXR-T accuracy ~0.64** (người, zero-shot) — > BioViL-T 0.602, sát CoCa-CXR 0.650; silver change-only F1 ~0.58; giải thích = **lead-region exact** |
| **M5** | 2 bảng + template + verify xác định | demo end-to-end sạch (out-of-table ≈ 0, temporal-halluc = 0 by construction) |

**Định vị SOTA (bản gọn):** detection IoU 0.807 (RGRG 0.887, nhưng ta chỉ 1/4 data) · bệnh AUC 0.83 (frozen
encoder; Anatomy-XNet 0.840 fine-tune cả backbone) · concept-F1 0.89 (≥ các CBM CXR đã xuất bản) · tiến triển
acc 0.64 (sát CoCa 0.650). **Khoảng trống novelty:** *pool theo vùng → concept bottleneck faithful → nhiều
bệnh + tiến triển có lead-region exact* — chưa bài nào chiếm.

> Số M3 gần mức shippable; số MS-CXR-T là audit ngoài (nhãn người); silver là số phát triển. M4 vẫn là rủi ro
> khoa học chính còn mở (chốt tập test thời gian — mục 12.1).

---

## 14. Một câu tóm lại

> VERA không cố làm cho một mô hình sinh báo cáo **bớt** bịa. Nó **thay khâu sinh tự do bằng khâu lắp ráp từ
> một bảng verify được** — nên báo cáo *không thể* nói điều gì mà bảng không chứng minh, và *không thể* nói
> về thời gian khi không có quá khứ để so. Trung thực ở đây là **thuộc tính của kiến trúc**, không phải một
> chỉ số cần tối ưu.
