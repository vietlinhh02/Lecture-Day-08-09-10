# Quality report — Lab Day 10 (nhóm)

**run_id inject-bad:** `inject-bad`  
**run_id fix-good:** `fix-good`  
**Ngày:** 2026-06-10

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (inject-bad) | Sau (fix-good) | Ghi chú |
|--------|-------------------|----------------|---------|
| raw_records | 247 | 247 | Cùng data source |
| cleaned_records | 36 | 36 | Không đổi (cùng cleaned rows) |
| quarantine_records | 211 | 211 | Không đổi |
| Expectation halt? | FAIL (`refund_no_stale_14d_window` violations=1) | OK (tất cả pass) | `--no-refund-fix` giữ chunk "14 ngày" stale |
| Grading pass | 9/10 | 10/10 | `gq_d10_01` FAIL→OK |
| Eval pass | 20/21 | 21/21 | `q_refund_window` FAIL→OK |

---

## 2. Before / after retrieval (bắt buộc)

**Câu hỏi then chốt:** refund window (`q_refund_window`)

**Trước (inject-bad — `--no-refund-fix`):**
- `contains_expected=yes`, `hits_forbidden=yes`
- Top-1: `policy_refund_v4` (đúng doc) nhưng chunk chứa "14 ngày làm việc" → forbidden keyword match
- Grading `gq_d10_01`: `hits_forbidden=true` → FAIL

**Sau (fix-good — pipeline chuẩn):**
- `contains_expected=yes`, `hits_forbidden=no`
- Top-1: `policy_refund_v4` — chunk đã fix thành "7 ngày làm việc"
- Grading `gq_d10_01`: `hits_forbidden=false` → OK

**Merit — versioning HR (`q_hr_annual_leave_under3`):**
- Cả trước và sau: `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`
- Content-based HR stale filter đã loại bản "10 ngày phép năm" từ cả inject lẫn fix

---

## 3. Freshness & monitor

**Kết quả:** `freshness_check=FAIL`

- `latest_exported_at=2026-04-10T00:00:00`
- `age_hours=1472.32`
- `sla_hours=24`
- `reason=freshness_sla_exceeded`

**Giải thích:** Data sample có timestamp cũ (tháng 4/2026). SLA 24 giờ áp cho "thời gian từ lần export gần nhất đến lần chạy pipeline". FAIL là hợp lý trên data mẫu — cần data source cập nhật timestamp trong production.

---

## 4. Corruption inject (Sprint 3)

**Cách inject:** `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`

- `--no-refund-fix`: bỏ qua rule fix "14 ngày làm việc" → "7 ngày làm việc" → chunk stale vẫn nằm trong cleaned
- `--skip-validate`: vẫn embed dù expectation fail (phục vụ demo có chủ đích)

**Phát hiện:** Expectation `refund_no_stale_14d_window` FAIL với `violations=1`. Trong production, pipeline sẽ HALT ở bước này (không có `--skip-validate`).

**Ảnh hưởng retrieval:** Câu `q_refund_window` bị `hits_forbidden=yes` vì chunk chứa "14 ngày" nằm trong top-k retrieved.

---

## 5. Hạn chế & việc chưa làm

- **Inject scope nhỏ:** chỉ test được refund stale (1 expectation fail). Có thể mở rộng inject thêm HR version conflict hoặc missing doc type.
- **Freshness FAIL:** data mẫu có timestamp cũ → cần test với data mới hơn hoặc điều chỉnh SLA.
- **Eval 21 câu:** chưa test hết edge case (VD: câu hỏi kết hợp multi-doc).
