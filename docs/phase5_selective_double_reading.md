# Phase 5 — VERA Selective Double-Reading

## 1. Ý tưởng trung tâm

Mục tiêu lâm sàng không nên được phát biểu là **“VERA thay một bác sĩ nhiều kinh
nghiệm”**. Claim mạnh hơn về mặt khoa học và an toàn là:

> **Một bác sĩ senior, một bác sĩ junior và VERA có thể duy trì độ an toàn không
> kém hơn quy trình hai bác sĩ senior, đồng thời giảm số ca cần senior thứ hai đọc
> toàn bộ hay không?**

Tên làm việc: **VERA Selective Double-Reading (VERA-SDR)**.

VERA-SDR không coi AI là một người đọc độc lập có quyền ký báo cáo. VERA đóng vai
trò một **verifiable second-reader policy**: tạo draft có provenance, so sánh với
nhận định độc lập của junior, lượng hóa bất đồng và quyết định ca nào cần escalated
đến senior thứ hai. Như vậy, senior thứ hai được giải phóng ở các ca đủ an toàn,
nhưng vẫn được giữ lại cho ca khó.

Điểm bán hàng của Phase 5 vì thế không còn chỉ là “sinh report ít hallucination”,
mà là:

> **Biến faithfulness, calibration và abstention thành một ngân sách đọc phim của
> chuyên gia có thể đo được.**

---

## 2. Workflow đề xuất

```text
Current ± prior CXR
        │
        ├── Junior đọc độc lập → structured preliminary findings
        │
        └── M1–M4 → M5 VERA report + evidence ledger
                           │
                           ▼
                Human–AI concordance engine
                           │
             ┌─────────────┴─────────────┐
             │                           │
       GREEN / eligible            AMBER hoặc RED
       junior ↔ VERA đồng ý         bất đồng / bất định / lỗi gate
             │                           │
             ▼                           ▼
      1 senior xác nhận nhanh      1 senior đọc đầy đủ
      và ký report cuối            + gọi senior thứ hai nếu cần
```

Junior phải hoàn thành nhận định **trước khi** thấy output VERA. Nếu junior viết
free text, hệ thống có thể đề xuất cấu trúc bằng parser, nhưng junior phải xác nhận
bảng structured finding trước khi concordance được tính. Không dùng output parser
chưa xác nhận làm “ý kiến của junior”.

Senior luôn là người ký báo cáo cuối. “Giải phóng senior thứ hai” có nghĩa là giảm
tỷ lệ ca cần double-reading bởi hai senior, không phải tự động phát hành report.

---

## 3. Phase 5 phải xuất một **review packet**, không chỉ một đoạn văn

### 3.1. Final draft

Report template ngắn gọn, được lắp ráp 1:1 từ bảng M3/M4. Mọi câu đều có
`claim_id`; văn xuôi không phải nguồn sự thật.

### 3.2. Claim evidence ledger

Mỗi claim hiển thị thành một card:

| Field | Ý nghĩa |
|---|---|
| Finding | Một trong 14 observation |
| Region | Vùng giải phẫu và bounding box tương ứng |
| Current state | assert / hedge / omit và calibrated confidence |
| Temporal state | improved / stable / worsened hoặc unavailable |
| Prior validity | prior thật, same-patient, pair hợp lệ hay không |
| Concept support | Chỉ hiện concept đã qua explanation gate |
| Input reliance | Current / prior / difference / disease-logit contribution từ IGI |
| Provenance | Ô M3/M4 và model version tạo claim |
| Human agreement | junior agrees / junior only / VERA only / conflicting |
| Senior action | accept / edit / reject / escalate |

Không dịch latent feature thành dấu hiệu y khoa. Nếu concept không qua gate, card
chỉ hiển thị regional grounding và confidence.

### 3.3. Discrepancy queue

Thay vì bắt senior đọc lại toàn bộ danh sách, M5 ưu tiên các điểm bất đồng:

- `JUNIOR_ONLY`: junior thấy finding nhưng VERA không assert;
- `VERA_ONLY`: VERA assert nhưng junior không thấy;
- `LOCATION_MISMATCH`: cùng finding nhưng khác vùng;
- `POLARITY_CONFLICT`: present đối nghịch absent;
- `TEMPORAL_CONFLICT`: improved đối nghịch worsened hoặc temporal claim không có
  prior hợp lệ;
- `UNCERTAINTY_CONFLICT`: một bên assert, bên kia hedge/omit;
- `UNSUPPORTED_TEXT`: câu của draft không truy được về bảng M3/M4;
- `QUALITY_OR_OOD`: ảnh, projection, box hoặc input nằm ngoài điều kiện vận hành.

Queue được xếp theo **clinical risk trước, confidence sau**. Danh sách finding nguy
hiểm và trọng số rủi ro phải do hội đồng bác sĩ chốt trước nghiên cứu, không chọn
sau khi xem test result.

---

## 4. Chính sách selective escalation

Một ca chỉ được gắn `single_senior_eligible=True` khi đồng thời thỏa tất cả điều
kiện:

1. ảnh và prior (nếu có) qua quality/pair-validity gate;
2. không có high-risk finding ở band hedge;
3. mọi claim được assert đều qua calibration và provenance gate;
4. concept explanation chỉ được surface khi qua intervention/leakage gate;
5. M4 không vi phạm time-reversal hoặc temporal guard;
6. junior và VERA không có clinically significant disagreement;
7. senior thứ nhất không yêu cầu escalation.

Pseudo-policy:

```python
eligible = (
    image_quality_pass
    and pair_validity_pass
    and calibration_pass
    and provenance_pass
    and temporal_guard_pass
    and not high_risk_hedge
    and not clinically_significant_disagreement
)

if not eligible or senior_requests_help:
    require_second_senior = True
else:
    require_second_senior = False
```

Ngưỡng không được tối ưu chỉ để giảm workload. Phải chọn trên validation bằng
**risk–coverage curve**:

- coverage: tỷ lệ ca không cần senior thứ hai;
- risk: tỷ lệ clinically significant error trên các ca được release;
- operating point: coverage lớn nhất vẫn thỏa non-inferiority margin đã định
  trước.

Đây là headline hợp lý hơn BLEU hoặc chỉ CheXbert-F1: **bao nhiêu phần trăm lượt
đọc của senior thứ hai có thể tránh được tại cùng mức an toàn?**

---

## 5. Chống automation bias

AI sai có thể kéo bác sĩ từ quyết định đúng sang sai. Vì vậy UI và reader study
phải có các rào chắn sau:

1. **Independent first read:** junior commit nhận định trước khi thấy VERA.
2. **Evidence before prose:** hiển thị bbox, prior/current và discrepancy trước
   draft văn xuôi; không dùng một câu chắc chắn nhưng không có provenance.
3. **No silent acceptance:** senior phải accept/edit/reject từng claim quan trọng;
   không có nút “accept all” cho ca RED.
4. **Symmetric disagreement:** UI không mặc định VERA đúng; `JUNIOR_ONLY` và
   `VERA_ONLY` được trình bày ngang hàng.
5. **Harmful override logging:** ghi lại ca bác sĩ ban đầu đúng nhưng đổi thành sai
   sau khi xem VERA, không chỉ ghi beneficial correction.
6. **Fallback:** lỗi verifier, missing prior, OOD hoặc model unavailable phải rơi
   về workflow con người thông thường.

Regional boxes của VERA có giá trị ở đây không chỉ để “giải thích đẹp”, mà để bác
sĩ kiểm tra trực tiếp vị trí AI đang dựa vào. Tuy nhiên, box không được xem là bằng
chứng rằng AI đúng.

---

## 6. Reader study để chứng minh claim

### 6.1. Ba arm cần thiết

| Arm | Thành phần | Câu hỏi trả lời |
|---|---|---|
| A — Standard | senior 1 + senior 2 | Reference workflow |
| B — Skill mix | senior + junior, không VERA | Lợi ích có đơn thuần do thay đổi nhân lực không? |
| C — VERA-SDR | senior + junior + VERA, selective escalation | VERA có bù được khoảng cách và giảm senior workload không? |

Thiết kế phù hợp là randomized multi-reader multi-case crossover, có washout và
đảo thứ tự ca. Reference standard cần một panel senior độc lập và adjudication;
không dùng chính output của arm A làm truth không tranh cãi.

### 6.2. Primary co-endpoints

1. **Safety non-inferiority:** clinically significant report-discrepancy rate của
   arm C so với arm A.
2. **Senior workload superiority:** senior-minutes trên 100 ca, hoặc số full
   second-senior reads trên 100 ca.

Một endpoint không đủ. Chỉ chứng minh accuracy mà không đo thời gian thì chưa chứng
minh “giải phóng senior”; chỉ chứng minh nhanh hơn mà không có non-inferiority thì
không đủ an toàn.

### 6.3. Secondary endpoints

- sensitivity/specificity và macro-F1 cho high-risk findings;
- lỗi localization và temporal-direction;
- RADPEER hoặc thang clinically significant/minor discrepancy;
- report completeness, unsupported-claim rate và temporal-hallucination rate;
- calibration, abstention coverage và selective risk;
- thời gian junior, senior thứ nhất và senior thứ hai tách riêng;
- beneficial correction và harmful override sau khi xem AI;
- tỷ lệ senior chấp nhận, sửa, bác bỏ và escalated từng claim;
- subgroup theo kinh nghiệm, projection, portable study, có/không prior, disease,
  image quality và cơ sở y tế.

Non-inferiority margin, high-risk finding list và sample size phải được định trước
với bác sĩ và statistician. Không chọn margin sau khi biết kết quả.

---

## 7. Schema mở rộng sau lõi M5

Không cho input của junior đi ngược vào M3/M4. Giữ prediction path của VERA thuần
ảnh; thêm một **clinical safety wrapper** sau report core:

```python
@dataclass
class HumanFinding:
    finding: str
    region: str | None
    polarity: str
    progression: str | None
    confirmed_by_junior: bool

@dataclass
class Discrepancy:
    kind: str
    finding: str
    region: str | None
    clinical_risk: str
    vera_claim_id: str | None
    requires_second_senior: bool

@dataclass
class ClinicalReviewPacket:
    vera_report: M5Report
    junior_findings: list[HumanFinding]
    discrepancies: list[Discrepancy]
    single_senior_eligible: bool
    eligibility_failures: list[str]
    final_senior_actions: list[dict]
```

Cách tách này giữ nguyên luận điểm faithfulness: junior không làm VERA “đoán đúng
hơn” một cách mờ; junior chỉ tạo thêm một nguồn kiểm tra độc lập cho quyết định
escalation.

---

## 8. Claim được phép và claim chưa được phép

### Trước reader study

Được phép nói:

> VERA-SDR is designed to support a skill-mixed reporting workflow through
> provenance-aware discrepancy detection and selective escalation.

Chưa được nói:

> VERA replaces a senior radiologist or safely reduces double-reading workload.

### Sau reader study, nếu đạt cả hai co-endpoint

Có thể nói:

> A senior–junior team supported by VERA achieved non-inferior clinically
> significant discrepancy rates relative to double-senior reading while
> reducing full second-senior interpretations by X% and senior reading time by
> Y%.

`X` và `Y` phải là kết quả thực nghiệm kèm confidence interval.

---

## 9. Vì sao hướng này hợp tinh thần VERA

- **Verifiable:** senior thấy nguồn của từng claim và từng lý do escalation.
- **Evidence-grounded:** disagreement dẫn về ảnh, vùng và prior/current pair.
- **Regional:** lỗi location được xem là một loại discrepancy riêng, không bị che
  bởi image-level agreement.
- **Assembly:** report vẫn là readout có cấu trúc; human layer adjudicate chứ không
  cho LLM suy luận tự do.
- **Faithful:** hệ thống có quyền nói “không đủ bằng chứng, cần người thứ hai”.
- **Clinical utility:** faithfulness được chuyển thành workload reduction tại một
  mức risk định trước.

Câu định vị mạnh cho paper:

> **VERA is not evaluated as an autonomous reporter; it is evaluated as a
> selective second-reading policy that converts calibrated evidence and
> claim-level provenance into a measurable senior-review budget.**

---

## 10. Nguồn thiết kế tham khảo

1. Hong et al. cho thấy preliminary AI reports có thể cải thiện thời gian và chất
   lượng báo cáo CXR trong reader study, nhưng đây chưa phải bằng chứng thay thế
   double-senior reading: <https://doi.org/10.1148/radiol.241646>.
2. Bernstein et al. cho thấy output AI sai có thể làm radiologist đổi từ quyết
   định đúng sang sai; việc khoanh vùng nghi ngờ giúp giảm một phần tác hại:
   <https://doi.org/10.1007/s00330-023-09747-1>.
3. MASAI cung cấp precedent về đánh giá một chính sách AI-supported selective
   reading bằng randomized non-inferiority design, nhưng trên mammography nên chỉ
   dùng làm precedent thiết kế, không suy diễn trực tiếp sang CXR:
   <https://doi.org/10.1016/S1470-2045(23)00298-X>.
4. Reader study hoặc pilot lâm sàng nên được báo cáo theo DECIDE-AI và nghiên cứu
   imaging-AI nên đối chiếu CLAIM 2024:
   <https://doi.org/10.1038/s41591-022-01772-9> và
   <https://doi.org/10.1148/ryai.240300>.

