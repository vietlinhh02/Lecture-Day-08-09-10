# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** ___________  
**Vai trò:** Cleaning & Quality Owner  
**Ngày nộp:** 2026-06-10  
**Độ dài yêu cầu:** **400–650 từ**

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**
- `transform/cleaning_rules.py` — thêm 3 rule mới: strip noise prefix, normalize repeated words, content-based HR stale filter
- `quality/expectations.py` — thêm 2 expectation mới: `required_doc_types_present` (halt), `no_noise_prefix` (warn)
- `contracts/data_contract.yaml` — cập nhật allowed_doc_ids, thêm `access_control_sop`

**Kết nối với thành viên khác:**
- Embed Owner đảm bảo `chunk_id` idempotent sau khi tôi sửa cleaning rule
- Monitoring Owner dùng expectation results để ghi freshness trong runbook

**Bằng chứng:**
- Commit sửa `ALLOWED_DOC_IDS` thêm `access_control_sop` trong `transform/cleaning_rules.py`
- Commit thêm 3 rule mới (strip prefix, normalize, HR content filter)
- Commit thêm 2 expectation trong `quality/expectations.py`

---

## 2. Một quyết định kỹ thuật (100–150 từ)

**Quyết định:** Chọn `required_doc_types_present` là expectation **halt** thay vì warn.

**Lý do:** Nếu pipeline bỏ sót 1 doc type (VD: `access_control_sop`), agent sẽ không trả lời được câu hỏi về access control — ảnh hưởng trực tiếp đến user experience. Đây là lỗi nghiêm trọng hơn chunk ngắn (warn) hay noise prefix (warn).

**Trade-off:** Halt = pipeline sẽ dừng nếu thiếu doc type → có thể block pipeline do data source thay đổi tạm thời. Nhưng an toàn hơn là ship data thiếu → agent trả lời sai hoặc "I don't know".

**Kết quả:** Khi chưa thêm `access_control_sop` vào allowlist, pipeline halt ở expectation `required_doc_types_present`. Sau khi sửa, pipeline exit 0 và grading question 10 pass.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Triệu chứng:** Chạy pipeline lần đầu → `expectation[hr_leave_no_stale_10d_annual] FAIL (halt) :: violations=2`

**Phát hiện:** Expectation `hr_leave_no_stale_10d_annual` kiểm tra cleaned rows có chứa "10 ngày phép năm" hay không. 2 bản HR có `effective_date >= 2026-01-01` nhưng nội dung vẫn ghi "10 ngày phép năm (bản HR 2025)" — lọt qua filter date-based.

**Fix:** Thêm content-based filter trong `cleaning_rules.py`: mọi chunk `hr_leave_policy` chứa "10 ngày phép năm" đều bị quarantine bất kể ngày.

**Kết quả:** `violations=2 → 0`, expectation pass. 2 bản HR stale bị quarantine với reason `stale_hr_policy_content_10d`.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**Trước (inject-bad, run_id=`inject-bad`):**
```
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1
```
Grading `gq_d10_01`: `contains_expected=true`, `hits_forbidden=true` → FAIL

**Sau (fix-good, run_id=`fix-good`):**
```
expectation[refund_no_stale_14d_window] OK (halt) :: violations=0
```
Grading `gq_d10_01`: `contains_expected=true`, `hits_forbidden=false` → OK

**Eval tổng:** 20/21 (trước) → 21/21 (sau). Grading tổng: 9/10 → 10/10.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ mở rộng inject corruption test: thêm test case HR version conflict (inject bản 2025 vào cleaned) và missing doc type (xóa `access_control_sop` khỏi allowlist). Điều này giúp verify tất cả expectation đều bắt được lỗi, không chỉ refund stale.
