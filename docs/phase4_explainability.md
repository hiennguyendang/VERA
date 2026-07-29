# Phase 4 Explainability

## Faithful Temporal Concept Bottleneck và Input-group Intervention

**Phạm vi:** Phase 4 / M4 — dự đoán progression theo từng `(region, disease)`.

**Trạng thái:** đặc tả thiết kế, chưa hiện thực.

**Class order bắt buộc khớp code hiện tại:**

```text
0 = stable
1 = improved
2 = worsened
```

Tài liệu này bổ sung hai cơ chế explainability cho M4:

1. **Faithful Temporal Concept Bottleneck (FTCB):** progression phải đi qua thay đổi của các concept y khoa có tên, với ánh xạ concept→disease bị mask và có ràng buộc dấu.
2. **Input-group Intervention (IGI):** can thiệp trực tiếp lên từng nhóm input thật của M4 để đo model đang dựa vào current, prior, difference hay disease logits ở mức nào.

Hai cơ chế phục vụ hai câu hỏi khác nhau:

| Cơ chế | Câu hỏi trả lời |
|---|---|
| FTCB | “Bệnh này được dự đoán cải thiện/nặng lên **vì dấu hiệu nào đã thay đổi**?” |
| IGI | “M4 thực sự dùng **nguồn tín hiệu nào** để đưa ra prediction này?” |

Mục tiêu là tạo explanation phản ánh đúng phép tính của model, không chỉ tạo một heatmap hoặc câu giải thích có vẻ hợp lý.

---

## 1. Bối cảnh M4 hiện tại

M4 hiện có hai kiến trúc:

### 1.1 `regiondiff`

Đọc frozen-M3 region cache và ghép input theo vùng:

```text
feat_current       [29, F]
feat_prior         [29, F]
feat_delta         = feat_current - feat_prior
logit_current      [29, 14]
logit_prior        [29, 14]
```

Với `INPUT_MODE="full"`, input của head là:

```text
[feat_current ; feat_prior ; feat_delta ; logit_current ; logit_prior]
```

### 1.2 `tempfuse`

Đọc hai patch grid BioViL-T `[196, F]`, thực hiện:

```text
current patches <- cross-attention(prior patches)
                -> self-attention
                -> bbox-guided region pool
                -> temporal region feature [29, F]
                -> progression head [29, 14, 3]
```

Với `TEMPFUSE_INPUT_MODE="feat_logits"`, head nhận:

```text
[temporal_region_feature ; M3_current ; M3_prior ; M3_current-M3_prior]
```

### 1.3 Hạn chế explainability hiện tại

- M4 trả `stable/improved/worsened` nhưng chưa chỉ ra dấu hiệu y khoa nào đã đổi.
- MLP/KAN head có thể trộn tín hiệu giữa nhiều disease và nhiều nhóm feature.
- Attention `alpha` của TempFuse chỉ cho biết vùng/pixel được pool mạnh; tự nó chưa chứng minh patch đó quyết định progression.
- Một prediction đúng có thể đến từ shortcut như view, tư thế, prior slot hoặc disease khác.
- `flip_consistency_loss` kiểm tra tính đối xứng theo thời gian ở mức training, nhưng chưa được surface thành explanation-validity score cho từng ca.

---

# PHẦN A — Faithful Temporal Concept Bottleneck

## 2. Mục tiêu

FTCB tạo một đường progression có thể truy ngược:

```text
M3 concept ở prior/current
        -> concept delta có hướng
        -> disease-masked temporal head
        -> stable / improved / worsened
        -> top concept contributions
```

Một explanation hợp lệ phải có dạng:

> Pleural Effusion tại left costophrenic angle được dự đoán worsened vì activation của `pleural effusion` tăng 0.31 và `blunting of costophrenic angle` tăng 0.18 so với prior.

Explanation không được nói concept `c` là nguyên nhân nếu prediction progression có thể hoàn toàn bỏ qua `c`.

---

## 3. Dữ liệu đầu vào

### 3.1 Concept activation từ frozen M3

M3 B-faithful đã tạo:

```text
concept_logits [29, 69]
```

FTCB dùng activation sau sigmoid:

```math
c_{r,c}^{t} = \sigma(\ell_{r,c}^{t}) \in [0,1]
```

với:

- `r`: anatomical region;
- `c`: concept;
- `t ∈ {prior, current}`.

### 3.2 Cache contract đề xuất

Không thay đổi format region cache cũ để tránh phá checkpoint. Tạo cache song song:

```text
data/m4_concept_cache/<image_id>.npy
shape = [29, 69]
dtype = float16
content = sigmoid(M3 concept_logits)
```

Chi phí lưu thêm xấp xỉ:

```text
29 × 69 × 2 byte ≈ 4 KB / image
≈ 0.8 GB / 200,000 images
```

Script `phase_3/scripts/8-precompute_regions.py` có thể nhận thêm:

```bash
--concept-cache-out data/m4_concept_cache_<tag>
```

Chỉ checkpoint M3 có concept head mới được phép tạo cache này. Với mode A, FTCB không khả dụng.

---

## 4. Concept delta có hướng lâm sàng

### 4.1 Delta thô

```math
\Delta c_{r,c} = c_{r,c}^{current} - c_{r,c}^{prior}
```

Delta dương chỉ có nghĩa concept tăng; chưa chắc đã tương đương bệnh nặng lên. Vì vậy cần một bảng chiều lâm sàng:

```text
CONCEPT_SEVERITY_SIGN[c] ∈ {-1, 0, +1}
```

- `+1`: concept tăng thường là nặng lên, ví dụ `pleural effusion`, `pulmonary edema/hazy opacity`.
- `-1`: concept tăng biểu diễn trạng thái tốt hơn hoặc giảm bệnh.
- `0`: concept không có chiều severity rõ ràng; không dùng để giải thích direction.

Evidence có hướng:

```math
e_{r,c} = s_c \cdot \Delta c_{r,c}
```

Trong đó `e > 0` là bằng chứng theo hướng worsened và `e < 0` là bằng chứng theo hướng improved.

### 4.2 Concept không phù hợp

Các concept sau phải được audit riêng hoặc loại khỏi direction head:

- support devices và tubes/lines;
- concept chỉ biểu diễn presence, không biểu diễn severity;
- concept có nghĩa thay đổi phụ thuộc disease;
- concept có F1/AUC ảnh→concept thấp;
- concept có nhãn weak hoặc hedge nhiều.

Không được mặc định mọi concept đều có thể giải thích progression.

---

## 5. Disease-masked temporal head

### 5.1 Mask concept→disease

Tái sử dụng crosswalk của M3:

```text
TEMPORAL_CONCEPT_MASK[d, c] = 1
```

chỉ khi concept `c` được phép cung cấp bằng chứng cho disease `d`.

Điều này ngăn ví dụ `chest tube` trở thành explanation cho `Pneumothorax` chỉ vì có tương quan điều trị.

### 5.2 Direction score

Với mỗi `(region r, disease d)`:

```math
z_{r,d} =
\sum_c
\operatorname{softplus}(W^{dir}_{d,c})
\cdot M_{d,c}
\cdot e_{r,c}
```

Ràng buộc:

- trọng số hiệu dụng luôn không âm;
- mỗi disease chỉ nhận concept được map;
- dấu của contribution đến từ delta concept có hướng, không đến từ trọng số tùy ý.

Contribution chính xác của concept `c`:

```math
contribution^{dir}_{r,d,c} =
\operatorname{softplus}(W^{dir}_{d,c})
\cdot M_{d,c}
\cdot e_{r,c}
```

Do đó:

- contribution dương hỗ trợ `worsened`;
- contribution âm hỗ trợ `improved`;
- contribution gần 0 không ảnh hưởng direction.

### 5.3 Change magnitude score

Direction và “có thay đổi hay không” là hai bài toán khác nhau. Dùng magnitude head riêng:

```math
m_{r,d} = b_d +
\sum_c
\operatorname{softplus}(W^{mag}_{d,c})
\cdot M_{d,c}
\cdot |e_{r,c}|
+ \operatorname{softplus}(w_d^{logit})|\Delta \ell_{r,d}^{M3}|
```

```math
P(change) = \sigma(m_{r,d})
```

Magnitude head cũng có trọng số không âm: thay đổi concept lớn hơn không thể làm bằng chứng “có change” giảm xuống.

### 5.4 Ghép thành ba lớp

```math
P(stable) = 1 - P(change)
```

```math
P(worsened) = P(change) \cdot \sigma(z_{r,d})
```

```math
P(improved) = P(change) \cdot (1-\sigma(z_{r,d}))
```

Thứ tự tensor cuối cùng vẫn là:

```text
[stable, improved, worsened]
```

Thiết kế này tương thích về ý tưởng với `head_mode="twostage"`, nhưng thay free head bằng một head có cấu trúc giải thích được.

---

## 6. Time-reversal faithfulness

Khi đảo thứ tự ảnh:

```text
(current, prior) -> (prior, current)
```

ta có:

```math
\Delta c' = -\Delta c
```

Do đó, theo kiến trúc:

```text
z' = -z
|delta|' = |delta|
```

Suy ra:

```text
Pstable(current, prior)   = Pstable(prior, current)
Pimproved(current, prior) = Pworsened(prior, current)
Pworsened(current, prior) = Pimproved(prior, current)
```

Nếu không thêm đường residual bất đối xứng, tính chất này đạt **by construction**, thay vì chỉ được khuyến khích bằng KL regularization.

Vẫn nên tính `reversal_error` ở inference để phát hiện lỗi dữ liệu/cache:

```math
E_{rev} =
\frac{1}{2}
\left\|
P(current,prior) - swap(P(prior,current))
\right\|_1
```

Nếu `E_rev > tau_reversal`, explanation progression không được surface ở M5.

---

## 7. Visual residual branch — tùy chọn, không phải mặc định

Concept space có thể không đầy đủ. Có thể thêm một visual residual branch từ TempFuse/region feature:

```text
concept temporal logits + residual visual logits -> fused progression
```

Tuy nhiên residual tạo đường bypass giống CBM leakage. Nếu dùng:

1. phải đo prediction khi zero/randomize concept branch;
2. phải tính `concept_explanation_coverage`;
3. chỉ surface explanation “vì concept” khi concept branch đủ chi phối;
4. trường hợp residual chi phối phải ghi `explanation_type="visual_residual"`, không được gán một concept plausible.

Một gate đề xuất:

```math
coverage_{concept} =
\frac{|margin_{full} - margin_{without\ concept}|}
{|margin_{full}| + \epsilon}
```

Chỉ cho phép explanation concept khi:

```text
coverage_concept >= tau_concept_coverage
AND concept prediction quality đạt gate per-concept
AND reversal_error <= tau_reversal
```

Shipping FTCB đầu tiên nên dùng pure concept path để có claim sạch; visual residual là ablation.

---

## 8. Loss

Loss chính giữ masked, class-weighted CE:

```math
L_{prog} = CE(P_{r,d}, y_{r,d})
```

Chỉ tính khi:

- progression target khác `-100`;
- region có mặt ở current;
- nếu `REQUIRE_PRIOR_PRESENT=True`, region cũng có mặt ở prior.

Loss bổ sung:

```math
L = L_{prog}
+ \lambda_{rev}L_{reversal}
+ \lambda_{sparse}L_{sparsity}
+ \lambda_{stable}L_{stable-margin}
```

Trong đó:

- `L_reversal`: chỉ cần cho residual/hybrid; pure FTCB đã đối xứng by construction.
- `L_sparsity`: khuyến khích explanation ngắn, nhưng không ép quá mạnh gây mất recall.
- `L_stable-margin`: ổn định khi concept delta nhỏ, tránh gọi change vì nhiễu rất nhỏ.

Không dùng concept từ report lúc inference. Report/scene graph chỉ cung cấp nhãn train.

---

## 9. Output: Progression Evidence Ledger

Mỗi prediction nên xuất đầy đủ provenance:

```json
{
  "region": "left costophrenic angle",
  "disease": "Pleural Effusion",
  "progression": "worsened",
  "confidence": 0.83,
  "change_probability": 0.88,
  "direction_probability": 0.94,
  "m3_prior_probability": 0.42,
  "m3_current_probability": 0.78,
  "m3_delta": 0.36,
  "explanation_type": "temporal_concept_bottleneck",
  "supporting_concepts": [
    {
      "concept": "pleural effusion",
      "prior": 0.39,
      "current": 0.81,
      "delta": 0.42,
      "direction_contribution": 0.31
    },
    {
      "concept": "blunting of costophrenic angle",
      "prior": 0.20,
      "current": 0.55,
      "delta": 0.35,
      "direction_contribution": 0.18
    }
  ],
  "opposing_concepts": [],
  "reversal_error": 0.01,
  "explanation_allowed": true
}
```

Không chỉ lưu top positive concept. Cần giữ cả `opposing_concepts` để bác sĩ thấy bằng chứng mâu thuẫn.

---

## 10. Evaluation FTCB

### 10.1 Task performance

- macro-F1 ba lớp;
- F1 `stable`, `improved`, `worsened`;
- change-only F1;
- per-disease và per-region F1;
- MS-CXR-T image-level external audit.

### 10.2 Concept quality

- F1/AUC của concept tại prior và current;
- sign accuracy của `delta_concept` trên tập temporal có annotation;
- performance theo nhóm concept good/medium/bad;
- explanation coverage sau concept gate.

### 10.3 Faithfulness

**Concept intervention:** tăng một mapped concept ở current, giữ prior cố định:

- worsened score không được giảm;
- improved score không được tăng.

Giảm concept ở current phải cho chiều ngược lại.

**Time reversal:** báo mean/P95 reversal error và tỷ lệ pass.

**Comprehensiveness:** xóa top-k supporting concepts, predicted-class margin phải giảm.

**Sufficiency:** chỉ giữ top-k concepts, prediction direction nên được bảo toàn.

**Random concept control:** top-k thật phải ảnh hưởng mạnh hơn k concept ngẫu nhiên.

### 10.4 Go/no-go đề xuất

Chỉ cho phép claim “why progression” khi:

```text
concept_intervention_pass >= 0.99
time_reversal_pass        >= 0.99
top-k comprehensiveness   > random control có ý nghĩa
concept_explanation_gate  = true cho chính prediction đó
```

Nếu không đạt, M4 vẫn được phép surface:

- progression class;
- calibrated confidence;
- region grounding;
- input-group intervention;

nhưng không được nói “vì concept X”.

---

# PHẦN B — Input-group Intervention

## 11. Mục tiêu

IGI đo ảnh hưởng của từng nhóm input bằng cách can thiệp trực tiếp và forward lại chính model:

```text
prediction gốc
vs
prediction khi bỏ/thay một nhóm input
```

Đây là phép audit mechanistic trên input thật của M4. Nó không cố gán nhãn y khoa cho latent feature.

Ví dụ output:

```text
Prediction: worsened, confidence 0.83

Nguồn ảnh hưởng lên predicted-class margin:
  explicit feature difference : +0.42
  current disease logits      : +0.25
  prior disease logits        : +0.18
  current absolute feature    : +0.09
  prior absolute feature      : +0.04
```

---

## 12. Hai protocol intervention

### 12.1 Composed-group intervention

Can thiệp sau khi `_compose()` đã tạo vector đầu vào head. Với `regiondiff/full`:

```text
G1 = feat_current
G2 = feat_prior
G3 = feat_current - feat_prior
G4 = logit_current
G5 = logit_prior
```

Thay từng slice độc lập rồi chạy lại head. Protocol này trả lời:

> Head progression đang dùng nhánh biểu diễn nào?

Ưu điểm: rẻ, rõ, cô lập đúng đường input của head.

Hạn chế: `G1`, `G2`, `G3` không độc lập về nguồn gốc vì `G3` được tạo từ hai nhóm đầu.

### 12.2 Source intervention

Can thiệp trước `_compose()`, sau đó tính lại mọi dependent feature.

Ví dụ xóa current feature:

```text
feat_current' = reference_current
feat_delta'   = feat_current' - feat_prior
```

Protocol này trả lời:

> Toàn bộ thông tin đến từ ảnh current ảnh hưởng prediction bao nhiêu?

Nên báo cả hai protocol:

- composed-group: giải thích head;
- source intervention: kiểm tra nguồn dữ liệu.

Không trộn hai kết quả vào cùng một phần trăm mà không ghi rõ protocol.

---

## 13. Nhóm input theo architecture

### 13.1 `regiondiff`

| Group | Nội dung |
|---|---|
| `feat_current` | regional feature của ảnh current |
| `feat_prior` | regional feature của ảnh prior |
| `feat_delta` | hiệu regional feature |
| `logit_current` | 14 M3 disease logits current |
| `logit_prior` | 14 M3 disease logits prior |

Với `input_mode=diff`, chỉ có:

```text
feat_delta
logit_delta
```

Với `input_mode=logits`, chỉ có:

```text
logit_current
logit_prior
```

Evaluator phải đọc input mode từ checkpoint, không hard-code `full`.

### 13.2 `tempfuse/feat`

Source-level groups:

```text
current_patches
prior_patches
cross-attention temporal update
bbox-region pooling
```

Can thiệp `cross-attention temporal update` bằng cách bypass các `CrossAttnFuse` block nhưng giữ current patches. Đây là ablation quan trọng:

> Model có thật sự dùng prior hay chỉ phân loại current image?

### 13.3 `tempfuse/feat_logits`

Thêm các group ở head:

```text
temporal_region_feature
M3_logit_current
M3_logit_prior
M3_logit_delta
```

Đây là cấu hình dễ giải thích nhất của TempFuse vì có thể tách phần visual temporal fusion khỏi phần symbolic disease transition.

---

## 14. Replacement baseline

Xóa một group bằng zero có thể tạo input ngoài phân phối. Mọi số headline phải kiểm tra ít nhất hai baseline.

### 14.1 Baseline A — standardized zero

Chỉ hợp lệ nếu feature đã được LayerNorm/standardize và zero gần trung tâm phân phối.

### 14.2 Baseline B — train mean

Thay bằng mean feature/logit của train split theo:

```text
region
disease nếu là logit
architecture/input mode
```

### 14.3 Baseline C — matched neutral reference, tùy chọn

Với source intervention, có thể thay prior/current bằng feature của một ca tương đồng nhưng không có change. Dùng làm robustness audit, không phải baseline mặc định vì khó kiểm soát confounder.

Kết luận chỉ ổn định nếu ranking nhóm tín hiệu không đổi đáng kể giữa baseline A và B.

---

## 15. Intervention score

Không chỉ đo xác suất predicted class vì softmax coupling có thể gây hiểu nhầm. Dùng predicted-class margin:

```math
margin(x) = logit_{k^*}(x) - \max_{j \ne k^*}logit_j(x)
```

Với group `g`:

```math
I_g = margin(x) - margin(x \setminus g)
```

- `I_g > 0`: group hỗ trợ prediction gốc.
- `I_g < 0`: group chống lại prediction gốc.
- `I_g ≈ 0`: group gần như không ảnh hưởng.

Không được bỏ các contribution âm khỏi artifact gốc.

Có thể thêm normalized positive share để hiển thị:

```math
share_g = \frac{\max(I_g,0)}{\sum_h \max(I_h,0)+\epsilon}
```

Nhưng `share` chỉ là visualization; số signed `I_g` mới là bằng chứng chính.

---

## 16. Interaction giữa các group

Các group có thể tương tác phi tuyến. Tổng `I_g` riêng lẻ không nhất thiết bằng toàn bộ margin.

Đo thêm pairwise synergy cho các cặp quan trọng:

```math
S_{g,h} = I_{g+h} - I_g - I_h
```

Ưu tiên:

- `feat_current × feat_prior`;
- `feat_delta × logit_delta`;
- `temporal_region_feature × M3_logit_delta`;
- `prior_patches × cross-attention update`.

Không cần tính toàn bộ power set. Chỉ tính một danh sách interaction được định nghĩa trước để tránh chi phí tổ hợp.

---

## 17. Sanity tests

### 17.1 Prior-usage test

Thay prior đúng bằng:

- prior của bệnh nhân khác;
- prior cùng bệnh nhân nhưng sai thời điểm;
- zero/mean prior.

Prediction hoặc confidence phải thay đổi hợp lý. Nếu không, M4 đang gần như bỏ qua prior.

### 17.2 Time-order test

Đảo current/prior:

- stable phải gần bất biến;
- improved/worsened phải đổi chỗ;
- IGI ranking của current/prior nên đổi vai tương ứng.

### 17.3 Model-randomization test

Randomize progression head. Explanation ranking phải mất cấu trúc. Nếu ranking gần như không đổi, evaluator đang đo input magnitude thay vì decision của model.

### 17.4 Label-shuffle control

Train một control với progression label bị shuffle trên subset nhỏ. Explanation không được còn pattern lâm sàng có hệ thống.

### 17.5 View-confounder slice

So sánh IGI trên:

- same-view pairs;
- AP↔PA;
- frontal↔lateral nếu còn trong dữ liệu.

Nếu prior/current absolute feature chi phối ở cross-view nhưng delta/logit signal chi phối ở same-view, cần hạ pair-quality hoặc abstain cho cross-view.

---

## 18. Output schema IGI

```json
{
  "region": "left lower lung zone",
  "disease": "Edema",
  "prediction": "improved",
  "confidence": 0.76,
  "intervention_protocol": "composed_group",
  "replacement_baseline": "train_mean",
  "original_margin": 1.42,
  "groups": {
    "feat_current": {
      "ablated_margin": 1.21,
      "signed_importance": 0.21
    },
    "feat_prior": {
      "ablated_margin": 1.31,
      "signed_importance": 0.11
    },
    "feat_delta": {
      "ablated_margin": 0.63,
      "signed_importance": 0.79
    },
    "logit_current": {
      "ablated_margin": 1.02,
      "signed_importance": 0.40
    },
    "logit_prior": {
      "ablated_margin": 1.17,
      "signed_importance": 0.25
    }
  },
  "dominant_group": "feat_delta",
  "baseline_stability": 0.92,
  "prior_used": true
}
```

`baseline_stability` đo độ tương đồng ranking giữa zero và train-mean interventions.

---

## 19. Computational plan

Với `G` group, naive IGI cần `G+1` forward passes. Tối ưu:

### `regiondiff`

- cache composed tensor;
- stack bản gốc và tất cả ablated variants trên batch dimension;
- chạy progression head một lần;
- không đọc lại feature cache.

### `tempfuse`

- composed-head intervention: cache temporal region feature, chỉ chạy head variants;
- source intervention: phải chạy lại fuse blocks;
- chỉ chạy source intervention trên audit subset hoặc các ca được flag;
- không chạy exhaustive patch deletion trên toàn bộ train/test.

Chế độ CLI đề xuất:

```bash
python phase_4/scripts/7-explain.py \
  --ckpt <best.pt> \
  --split test \
  --method input-groups \
  --protocol composed \
  --baselines zero train_mean \
  --out artifacts/explanations/m4_input_groups.test.jsonl
```

---

## 20. Evaluation IGI

### 20.1 Fidelity

- predicted-class margin drop khi xóa dominant group;
- tỷ lệ prediction flip sau khi xóa dominant group;
- dominant group vs random group control;
- prior-usage rate.

### 20.2 Stability

- rank correlation giữa zero và train-mean baseline;
- stability qua bootstrap;
- stability khi thêm nhiễu nhỏ vào feature;
- stability giữa các seed/checkpoint tương đương.

### 20.3 Temporal validity

- IGI time-reversal consistency;
- same-view vs cross-view breakdown;
- true-prior vs wrong-prior sensitivity;
- performance và explanation theo time-gap bins.

### 20.4 Go/no-go đề xuất

IGI chỉ được dùng như explanation per-case khi:

```text
baseline rank correlation >= 0.8
dominant-group deletion > random deletion
model-randomization sanity test pass
prior_used = true đối với progression claim
```

Nếu `prior_used=false`, prediction phải bị flag:

```text
"progression explanation invalid: model decision is current-image dominated"
```

---

# PHẦN C — Kết hợp FTCB và IGI

## 21. Explanation hierarchy

Mỗi prediction được phân tầng:

### Tier 1 — Temporal concept explanation

Cho phép khi:

- FTCB/concept gate pass;
- concept intervention pass;
- time reversal pass;
- concept branch thật sự chi phối prediction.

Output:

> Worsened vì concept A và B tăng tại region R.

### Tier 2 — Mechanistic input-group explanation

Khi concept không đủ tin nhưng IGI pass:

> Prediction chủ yếu được dẫn dắt bởi explicit temporal difference và M3 disease-logit delta.

Không dịch latent feature thành dấu hiệu y khoa.

### Tier 3 — Spatial grounding only

Khi chỉ region/pool evidence đáng tin:

> Model phát hiện thay đổi tại region R, nhưng không có concept-level explanation đủ tin cậy.

### Tier 4 — Abstain

Khi:

- prior gần như không được dùng;
- time reversal fail;
- pair quality thấp;
- confidence/calibration thấp;
- intervention ranking không ổn định.

Output:

> Không đủ bằng chứng để mô tả chiều tiến triển.

---

## 22. Integration với M5

M5 chỉ được readout các field đã được M4 xác nhận:

```text
progression
confidence
region
supporting_concepts nếu explanation_allowed=true
dominant_input_group
reversal_error
pair_quality
abstain_reason
```

M5 không được tự suy ra concept từ prose và không được biến `feat_delta` thành một dấu hiệu y khoa có tên.

Ví dụ văn bản hợp lệ:

> Left pleural effusion has worsened compared with the prior study. The progression decision was driven by increased pleural-effusion-related concepts in the left costophrenic angle. Temporal reversal consistency passed.

Ví dụ khi chỉ IGI pass:

> Left pleural effusion is predicted to have worsened. The prediction was primarily driven by the current–prior feature difference and disease-logit change; no concept-level rationale met the explanation gate.

---

## 23. File-level implementation plan

| File | Thay đổi đề xuất |
|---|---|
| `phase_3/scripts/8-precompute_regions.py` | xuất thêm concept activation cache |
| `phase_4/src/config.py` | thêm `ARCH=ftcb`, concept-cache path, explanation thresholds |
| `phase_4/src/dataset.py` | `ConceptCache`, load concept prior/current |
| `phase_4/src/heads.py` | `FaithfulTemporalConceptHead` với mask và non-negative weights |
| `phase_4/src/model.py` | `FTCBTKAN`, trả logits + contribution tensors |
| `phase_4/src/losses.py` | reversal/stable-margin/sparsity losses nếu cần |
| `phase_4/scripts/2-train.py` | CLI cho FTCB và loss weights |
| `phase_4/src/eval.py` | concept intervention, reversal, explanation coverage metrics |
| `phase_4/scripts/4-infer.py` | xuất Progression Evidence Ledger |
| `phase_4/scripts/7-explain.py` | entry point cho FTCB audit và IGI |
| `src/phase_5` hoặc M5 hiện hành | chỉ readout explanation đã gated |

Để tránh phá checkpoint cũ, mọi field mới phải có default và architecture name/version rõ ràng trong checkpoint.

---

## 24. Ablation matrix tối thiểu

| Run | Architecture | Explainability |
|---|---|---|
| A | current best TempFuse/regiondiff | none, task baseline |
| B | A + IGI audit | post-training mechanistic audit |
| C | FTCB pure concept | why-faithful candidate |
| D | FTCB không mask | kiểm tra cross-disease shortcut |
| E | FTCB cho phép signed free weights | kiểm tra monotonicity |
| F | FTCB + visual residual | accuracy/coverage trade-off |
| G | FTCB không reversal constraint | đo giá trị time symmetry |

Headline table phải báo đồng thời:

- task F1;
- change-only F1;
- concept explanation coverage;
- intervention pass rate;
- reversal pass rate;
- dominant-group deletion fidelity;
- external MS-CXR-T score.

Không chọn model chỉ theo accuracy nếu model đó không đạt explanation gate.

---

## 25. Rủi ro

1. **Concept delta nhiễu:** sai số concept ở hai thời điểm có thể cộng dồn.
2. **Concept không đầy đủ:** pure FTCB có thể giảm accuracy hoặc coverage.
3. **Weak temporal label:** comparison cues phù hợp để train nhưng không đủ làm bằng chứng cuối.
4. **View confounding:** feature delta có thể phản ánh AP/PA thay vì bệnh.
5. **Intervention ngoài phân phối:** zero ablation có thể phóng đại importance.
6. **Device semantics:** improved/worsened không luôn có nghĩa rõ cho Support Devices.
7. **Global finding:** một số disease cần grounding toàn ảnh thay vì một region duy nhất.

Các rủi ro này phải được thể hiện bằng gate/ablation, không che bằng một explanation câu chữ.

---

## 26. Quyết định khuyến nghị

Đường triển khai ưu tiên:

1. Hiện thực **IGI trước** vì không cần retrain và giúp audit model hiện tại.
2. Kiểm tra model có dùng prior, diff và M3 logits đúng như kỳ vọng không.
3. Mở rộng frozen-M3 cache với 69 concept activations.
4. Train **pure FTCB** và so với current best M4.
5. Chỉ thêm visual residual nếu pure FTCB mất task performance đáng kể.
6. Surface concept explanation ở M5 theo prediction-level gate, không theo architecture name đơn thuần.

Claim mong muốn cho paper:

> M4 không chỉ định vị và phân loại progression. Đối với các prediction qua explanation gate, chiều tiến triển được tính từ thay đổi của các concept y khoa được map trước, với ràng buộc monotonic và time-reversal; đồng thời input-group interventions xác nhận model thực sự sử dụng prior và temporal difference.

---

## 27. Tài liệu liên quan

- Koh et al., **Concept Bottleneck Models**, ICML 2020: <https://proceedings.mlr.press/v119/koh20a.html>
- Shin et al., **A Closer Look at the Intervention Procedure of Concept Bottleneck Models**, ICML 2023: <https://proceedings.mlr.press/v202/shin23a.html>
- Hu et al., **Learning Directional Semantic Transitions for Longitudinal Chest X-ray Analysis**, 2026: <https://arxiv.org/abs/2606.15938>
- Prakash et al., **CheXTemporal: A Dataset for Temporally-Grounded Reasoning in Chest Radiography**, 2026: <https://arxiv.org/abs/2605.11304>
- Aranya and Desai, **TRACE: Temporal Radiology with Anatomical Change Explanation**, 2026: <https://arxiv.org/abs/2602.02963>

