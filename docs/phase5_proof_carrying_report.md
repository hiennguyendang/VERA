# Phase 5 — VERA Proof-Carrying Radiology Report

## 1. Luận điểm đột phá

Phase 5 không nên chỉ là một report assembler. Hãy định nghĩa nó như một
**clinical claim compiler**:

> **Mỗi câu trong report phải mang theo một chứng chỉ máy-kiểm-tra được, chứng
> minh câu đó được phép xuất hiện từ đúng regional, concept và temporal decision
> path của VERA.**

Tên làm việc: **VERA-PCR — Proof-Carrying Radiology Report**.

Output của M5 không còn là:

```text
image → plausible report
```

mà là:

```text
M3/M4 prediction tables
        ↓
typed clinical claim graph
        ↓ compile
report text + proof manifest
        ↓ independent checker
verified report hoặc fail closed
```

Một report không có proof manifest hợp lệ chỉ là draft, không phải verified VERA
report.

---

## 2. Vì sao mạnh hơn “grounded report” thông thường

Grounded report thường nối một câu với bounding box hoặc dùng verifier sau khi
text đã được sinh. Điều đó cho biết câu liên quan đến đâu, nhưng chưa chắc câu là
readout trung thành của decision path.

VERA-PCR phân biệt ba khái niệm:

1. **Clinical correctness:** finding có thật trên ảnh hay không — cần ground truth
   và đánh giá của bác sĩ.
2. **Computational faithfulness:** claim có thật sự xuất phát từ phép tính M3/M4
   đã khai báo hay không — proof certificate kiểm tra được.
3. **Textual integrity:** câu văn có giữ nguyên disease, polarity, region và
   temporal state của claim hay không — compiler/checker kiểm tra được.

Certificate chỉ chứng minh (2) và (3), không giả vờ chứng minh (1). Chính sự phân
biệt này phù hợp tinh thần VERA hơn một heatmap hoặc câu giải thích nghe hợp lý.

---

## 3. Typed Clinical Claim Graph

Trước khi tạo văn xuôi, M5 chuyển output M3/M4 thành một intermediate
representation (IR):

```python
@dataclass(frozen=True)
class ClinicalClaim:
    claim_id: str
    observation: str
    polarity: str                 # present | uncertain | absent
    region: str | None
    temporal_state: str | None    # improved | stable | worsened
    current_cell: tuple[int, int]
    temporal_cell: tuple[int, int] | None
    concept_ids: tuple[int, ...]
    claim_type: str               # regional | global | temporal | concept
```

Mỗi loại claim có prerequisite như một type system:

| Claim type | Điều kiện bắt buộc |
|---|---|
| `RegionalClaim` | region tồn tại, detector/grounding hợp lệ, M3 qua threshold |
| `GlobalClaim` | được dán nhãn toàn ảnh; không giả thành regional grounding |
| `TemporalClaim` | prior thật và hợp lệ, `pair_mask=1`, M4 qua temporal gate |
| `ConceptClaim` | concept mapped đúng disease và qua quality/intervention/leakage gate |
| `HedgedClaim` | nằm trong uncertainty band đã calibrate |

Nếu thiếu prerequisite thì constructor của claim thất bại. Ví dụ, không tồn tại
API tạo `TemporalClaim` khi `has_prior=False`; temporal hallucination vì thế trở
thành lỗi type/compile, không phải lỗi được hy vọng bắt sau cùng.

---

## 4. Proof certificate cho từng claim

```python
@dataclass(frozen=True)
class ClaimCertificate:
    claim_id: str
    image_uid_hash: str
    prior_uid_hash: str | None
    model_version: str
    m3_cell: tuple[int, int]
    m3_probability: float
    m3_threshold_rule: str
    m4_cell: tuple[int, int] | None
    m4_probability: float | None
    prior_validity_rule: str | None
    concept_gate_ids: tuple[str, ...]
    compiler_rule_id: str
    surface_form_hash: str
    checker_version: str
```

Certificate phải trả lời được:

- finding nào, ở vùng nào và từ ô M3 nào;
- temporal phrase đến từ ô M4 nào và prior nào;
- confidence đã qua calibration rule nào;
- concept nào được phép surface và gate nào cho phép;
- template/compiler rule nào tạo câu;
- câu đã bị sửa sau khi compile hay chưa;
- model/checker version nào chịu trách nhiệm.

Proof checker là một module nhỏ, độc lập với assembler và không dùng LLM. Nó nạp
prediction tables, claim graph, report và manifest rồi trả:

```text
VERIFIED
REJECTED_UNSUPPORTED_CLAIM
REJECTED_OMITTED_REQUIRED_CLAIM
REJECTED_TEMPORAL_WITHOUT_PRIOR
REJECTED_LOCATION_MISMATCH
REJECTED_SEMANTIC_EDIT
REJECTED_VERSION_OR_HASH_MISMATCH
```

---

## 5. Bốn invariant có thể kiểm định hình thức

Gọi `Allowed(P)` là tập claim được phép từ prediction state `P`, `Required(P)` là
tập claim bắt buộc phải xuất theo policy, và `Claims(R)` là claim trong report.

### Soundness

```text
Claims(R) ⊆ Allowed(P)
```

Không report claim nào ngoài bảng dự đoán và gate.

### Completeness

```text
Required(P) ⊆ Claims(R)
```

Assembler không được âm thầm đánh rơi một finding bắt buộc.

### Temporal safety

```text
has_prior = False  ⇒  TemporalClaims(R) = ∅
```

### Total provenance

```text
∀ c ∈ Claims(R), ∃ exactly one valid certificate cert(c)
```

Bốn invariant này mạnh hơn round-trip CheXbert đơn thuần. CheXbert vẫn có thể dùng
như external audit, nhưng không còn là nền móng của guarantee vì bản thân labeler
có false positive/false negative.

---

## 6. Ý tưởng quan trọng nhất: proof-aware human editing

Trong workflow thật, bác sĩ sẽ sửa report. Nếu sau khi bác sĩ hoặc LLM sửa câu mà
report vẫn hiện huy hiệu “VERA verified”, provenance trở nên sai.

M5 cần một editor có ba trạng thái:

1. **Meaning-preserving edit:** chỉ sửa style nhưng structured claim không đổi;
   compiler tạo lại `surface_form_hash`, certificate vẫn hợp lệ.
2. **Supported semantic edit:** đổi câu sang một claim khác nhưng claim mới vẫn có
   certificate hợp lệ; recompile và recheck.
3. **Human-authored semantic edit:** thêm disease/location/progression ngoài
   `Allowed(P)`; câu được giữ vì bác sĩ có quyền kết luận, nhưng phải dán nhãn
   `HUMAN_OVERRIDE_UNVERIFIED` và lưu người sửa, thời điểm, lý do.

Như vậy hệ thống không ngăn bác sĩ dùng chuyên môn, nhưng không cho claim do con
người thêm vào mượn nhãn “AI-grounded”. Đây là audit trail có giá trị lâm sàng và
pháp lý hơn việc chỉ lưu final prose.

---

## 7. Hai tầng output cho bác sĩ

### Report view

Văn bản lâm sàng sạch, ngắn và dễ ký.

### Verification view

Click vào từng câu để xem:

- current/prior image và bbox;
- observation, region, calibrated confidence;
- temporal direction;
- concept support đã qua gate, nếu có;
- certificate state: green / invalidated / human override;
- exact rule và model version tạo claim.

Không hiển thị toàn bộ kỹ thuật theo mặc định. Proof manifest phục vụ audit;
bác sĩ chỉ mở khi cần kiểm tra.

---

## 8. Evaluation riêng cho Phase 5

### Construction tests

- property-based generation của hàng nghìn prediction tables giả;
- mọi `has_prior=False` phải compile ra zero temporal claim;
- mutation test: đổi disease, laterality, region, polarity hoặc progression trong
  text phải làm checker fail;
- xóa một required claim phải kích hoạt completeness failure;
- thay model version, image UID hoặc certificate hash phải fail closed.

### Scientific metrics

- **Proof coverage:** phần trăm câu có certificate hợp lệ;
- **Certified soundness violation rate:** claim ngoài `Allowed(P)`;
- **Certified omission rate:** required claim bị thiếu;
- **Edit invalidation sensitivity:** phần trăm semantic mutations bị bắt;
- **Human override rate:** số claim bác sĩ thêm ngoài model trên 100 report;
- **Time-to-verify:** thời gian bác sĩ kiểm tra report;
- **Selective risk–coverage:** nếu kết hợp VERA-SDR, workload được giải phóng tại
  từng mức certified risk.

Template compiler lý tưởng phải đạt proof coverage 100% và construction-level
violation bằng 0 trên toàn bộ test suite. Clinical accuracy vẫn phải báo riêng;
certificate hoàn hảo không cứu được một M3/M4 dự đoán sai.

---

## 9. Khác biệt với các hướng gần nhất

- **MAIRA-2** tạo grounded reports và RadFact đánh giá sentence-level entailment;
  VERA-PCR không tự do sinh diagnosis và certificate liên kết trực tiếp đến
  regional/temporal decision cells.
- **ConRad** và **CONRep** tập trung vào calibrated confidence hoặc conformal
  uncertainty của report; VERA-PCR tập trung vào construction-time authorization
  và provenance. Confidence thấp/cao không thay thế proof.
- **Formal verification của VLM report** kiểm tra impression có được suy ra logic
  từ phần findings đã trích hay không; VERA-PCR còn yêu cầu findings bắt nguồn từ
  đúng image decision path và giữ proof qua toàn bộ quá trình surface/edit.

Không nên tuyên bố “first” nếu chưa làm systematic search. Claim an toàn là:

> We introduce a proof-carrying report representation in which every surfaced
> statement is accompanied by a machine-checkable certificate linking its
> semantics to the regional and temporal prediction path that authorized it.

---

## 10. Minimum viable implementation

1. Chốt `ClinicalClaim` IR và controlled vocabulary.
2. Viết deterministic compiler từ IR sang template report.
3. Viết independent checker trước, không dùng chung code path với compiler.
4. Sinh `ClaimCertificate` và report-level manifest.
5. Thêm mutation/property tests cho bốn invariant.
6. Thêm proof-aware edit states.
7. Sau khi lõi chạy sạch mới cho phép constrained LLM paraphrase; mọi paraphrase
   bắt buộc reparse về cùng IR và qua checker, nếu fail thì fallback template.

---

## 11. Quan hệ với VERA-SDR

Hai ý tưởng không cạnh tranh:

- **VERA-PCR** là novelty kỹ thuật của Phase 5: report mang proof.
- **VERA-SDR** là clinical utility study: dùng proof status để quyết định ca nào
  cần senior thứ hai.

Kết hợp lại thành câu chuyện mạnh:

> VERA first compiles proof-carrying radiology reports whose claims are
> machine-checkably linked to regional and temporal evidence; it then uses the
> validity of those proofs, together with human–AI concordance, to allocate
> senior review selectively.

---

## 12. Tài liệu gần hướng này

1. MAIRA-2 và grounded report generation:
   <https://arxiv.org/abs/2406.04449>.
2. RadFact sentence-level correctness, completeness và grounding:
   <https://github.com/microsoft/RadFact>.
3. ConRad sentence/report-level calibrated confidence:
   <https://arxiv.org/abs/2603.29492>.
4. CONRep conformal uncertainty cho report drafting:
   <https://arxiv.org/abs/2602.03910>.
5. Neurosymbolic formal verification cho internal consistency giữa findings và
   impression: <https://arxiv.org/abs/2602.24111>.

