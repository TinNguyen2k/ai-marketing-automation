# Budget Allocation Algorithm

Tài liệu này giải thích logic tính toán phân bổ ngân sách media plan.

---

## Các hằng số

| Tên | Giá trị | Ý nghĩa |
|---|---|---|
| `REACH_FLOOR` | 60% | Reach tối thiểu để campaign có hiệu quả |
| `REACH_IDEAL` | 70% | Reach mục tiêu lý tưởng |
| `REACH_MAX` | 80% | Reach tối đa, không phân bổ thêm sau ngưỡng này |
| `FREQ_IDEAL` | `duration_weeks × 2` | Frequency lý tưởng theo thời gian campaign |
| `FREQ_HARD_CAP` | `FREQ_IDEAL × 1.3` | Frequency tối đa tuyệt đối |
| `REACH_OTHER_FLOOR` | 30% | Reach tối thiểu cho các objective không phải Awareness |

---

## Benchmark priority

Khi không có benchmark theo usecase/BU, hệ thống fallback theo thứ tự:

```
usecase_metric (BU + usecase khớp)
    ↓ không có
bu_metric (chỉ cần BU khớp)
    ↓ không có
fallback_benchmark (trung bình toàn platform 2025-nay)
```

Cost per result được markup thêm **30%** để tính net → gross.

---

## Phase 1 — Quyết định Awareness budget

Tính tổng chi phí cần thiết để đạt reach trên toàn bộ segments:

```
gross_at_ideal = Σ (seg_size × 70% × FREQ_IDEAL × CPM / 1000) / (1 - WHT)
gross_at_floor = Σ (seg_size × 60% × FREQ_IDEAL × CPM / 1000) / (1 - WHT)
```

Quyết định:

| Điều kiện | Kết quả |
|---|---|
| `budget ≥ gross_at_ideal` | Chạy 70% reach, freq = FREQ_IDEAL |
| `budget ≥ gross_at_floor` | Chạy 60% reach, freq = FREQ_IDEAL |
| `budget < gross_at_floor` | Bỏ qua Awareness, dồn toàn bộ sang objective khác |

---

## Phase 2 — 4-Pass Awareness

Với từng segment, ngân sách Awareness được phân bổ qua 4 pass theo thứ tự ưu tiên:

### Pass 1 — Đảm bảo 60% reach với freq_floor

```
net_needed  = reach_60% × freq_floor × CPM / 1000
gross_needed = net_needed / (1 - WHT)
```

Nếu không đủ budget → dùng hết phần còn lại, reach có thể dưới 60% và flag cảnh báo.

### Pass 2A — Tăng reach lên reach_target (60% hoặc 70%)

Dùng budget còn lại để tăng reach từ mức hiện tại lên target, giữ nguyên freq.

### Pass 2B — Tăng freq lên effective_freq_cap

Sau khi đạt reach target, dùng budget còn lại để tăng frequency.

### Pass 2C — Tăng reach lên 80% (REACH_MAX)

Sau khi đạt freq cap, tiếp tục mở rộng reach đến tối đa 80%.

### Pass 2D — Tăng freq lên FREQ_HARD_CAP

Budget cuối cùng dùng để đẩy freq lên FREQ_HARD_CAP nếu còn.

### Normalize reach (multi-segment)

Khi có nhiều segment, sau 4 pass sẽ normalize để tất cả segments đạt cùng `pct_reach`:

```
min_pct = min(pct_reach của tất cả segments)
→ các segment cao hơn bị cắt xuống min_pct
→ phần budget dư được trả lại vào remaining
```

Mục đích: đảm bảo công bằng giữa các nhóm target audience.

---

## Phase 3 — Phân bổ các objective còn lại

Budget còn lại sau Awareness được chia đều cho các objective (Traffic, Engagement, View, Install).

Với mỗi objective và mỗi segment:

```
seg_budget = budget_per_objective × (seg_size / total_seg_size)
net        = seg_budget × (1 - WHT)
impression = net / cost_per_result / rate     # rate = CTR, engagement_rate, view_rate...
```

Sau đó kiểm tra reach:

- Nếu `freq_natural < freq_floor` → cắt reach để enforce freq tối thiểu
- Nếu `freq_natural ≤ FREQ_HARD_CAP` → giữ nguyên, cap reach tại 80%
- Nếu `freq_natural > FREQ_HARD_CAP` → cắt reach về `impression / FREQ_HARD_CAP`

Tương tự Awareness, normalize reach về `min_pct` nếu có nhiều segments.

---

## Phase 4 — AVG metric adjust

So sánh kết quả dự tính với benchmark trung bình 30 ngày thực tế:

```
ratio = projected_result / avg_per_campaign
```

Nếu `ratio > 1` (dự tính vượt quá thực tế):

```
adjusted_result = projected_result / sqrt(ratio)
```

Dùng square root để điều chỉnh mềm — không cắt thẳng về avg mà giảm dần theo tỉ lệ. Budget dư ra được dồn vào Awareness để tăng frequency.

---

## Phase 5 — Phân bổ remaining budget

Sau tất cả các phase, nếu còn remaining:

1. Ưu tiên tăng freq Awareness (đến FREQ_HARD_CAP)
2. Nếu Awareness đã cap → phân đều vào các objective khác

Mục tiêu: **sử dụng hết 100% budget**, `pct_check = 1.0`.

---

## Additional phases

Nếu campaign description có chia phase với budget riêng (ví dụ: pre-launch, main, retargeting):

- Mỗi phase chạy 4-pass Awareness độc lập với `freq_ideal` riêng theo duration của phase đó
- Budget dư từ phase được trả về `main_budget`
- Nếu additional phase đã có Awareness → bỏ qua Awareness main, toàn bộ main budget dành cho objective khác

---

## Output structure

```json
{
  "allocation": {
    "Facebook_Awareness": [{ "segment_id", "gross", "net", "reach_actual", "pct_reach", "freq", "impression", "pct_budget" }],
    "Facebook_Traffic":   [{ "segment_id", "gross", "net", "clicks", "ctr", "cpc", "reach", "pct_reach" }],
    "additional_phases":  [{ "phase_name", "gross", "reach", "freq" }]
  },
  "summary": {
    "budget_total", "main_channel", "awareness_budget", "awareness_pct",
    "total_allocated", "remaining", "pct_check"
  },
  "flags": ["ℹ️ ...", "⚠️ ..."]
}
```

`flags` chứa các cảnh báo và thông tin debug — ví dụ reach dưới sàn, budget không đủ, freq đã cap.
