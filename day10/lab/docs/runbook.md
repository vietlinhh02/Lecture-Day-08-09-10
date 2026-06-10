# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

> User / agent thấy gì?

- Agent trả lời "14 ngày làm việc" cho câu hỏi refund window (sai: đúng là 7 ngày).
- Agent trả lời "10 ngày phép năm" cho nhân viên dưới 3 năm (sai: đúng là 12 ngày theo HR 2026).
- Agent không trả lời được câu hỏi về access control (thiếu `access_control_sop` trong index).
- Agent trả lời thông tin từ document cũ/stale dù đã sync mới.

---

## Detection

> Metric nào báo?

- `freshness_check=FAIL`: `latest_exported_at` quá cũ so với `FRESHNESS_SLA_HOURS`.
- `expectation[...] FAIL`: expectation suite phát hiện dữ liệu sai (refund stale, HR version conflict, missing doc types).
- `eval_retrieval.py`: `hits_forbidden=yes` hoặc `contains_expected=no` trên golden questions.
- `grading_run.py`: `contains_expected=false` trên grading questions.

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Xem `run_id`, `raw_records`, `cleaned_records`, `quarantine_records`, `latest_exported_at` |
| 2 | Mở `artifacts/quarantine/*.csv` | Xem record nào bị loại, lý do gì (`reason` column) |
| 3 | Chạy `python eval_retrieval.py` | Xem `contains_expected` và `hits_forbidden` trên từng câu |
| 4 | Kiểm tra `artifacts/logs/*.log` | Xem expectation nào FAIL, chi tiết violations |
| 5 | So sánh `cleaned CSV` với `data/docs/*.txt` | Xem nội dung chunk có khớp bản canonical không |

---

## Freshness check (kết quả mẫu)

**Lệnh:** `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_fix-good.json`

**Kết quả:** `FAIL`
- `latest_exported_at`: 2026-04-10T00:00:00
- `age_hours`: 1472.36
- `sla_hours`: 24
- `reason`: freshness_sla_exceeded

**Giải thích:**
- **PASS**: `age_hours` < `sla_hours` → data tươi, pipeline healthy.
- **WARN**: (không dùng trong baseline — có thể mở rộng cho SLA mềm).
- **FAIL**: `age_hours` >= `sla_hours` → data quá cũ, cần cập nhật từ nguồn.
- Trên data mẫu, FAIL là **hợp lý** vì timestamp export là tháng 4/2026 (cũ ~61 ngày). Trong production, cần data source export định kỳ (batch daily/hourly) để freshness PASS.
- Có thể điều chỉnh `FRESHNESS_SLA_HOURS` trong `.env` hoặc cập nhật `exported_at` trong raw CSV để test PASS scenario.

---

## Mitigation

> Rerun pipeline, rollback embed, tạm banner "data stale", …

1. **Rerun pipeline chuẩn**: `python etl_pipeline.py run` → nếu exit 0 thì data đã sạch.
2. **Nếu expectation FAIL**: kiểm tra quarantine CSV, xác định record nào bị flag nhầm → cập nhật cleaning rule hoặc allowlist.
3. **Nếu stale data**: kiểm tra `exported_at` trong raw CSV → cần data mới từ nguồn.
4. **Rollback embed**: nếu cần quay lại version trước, chạy lại pipeline với raw CSV cũ (nếu có backup).
5. **Banner "data stale"**: nếu freshness FAIL kéo dài, thông báo user agent đang dùng data cũ.

---

## Prevention

> Thêm expectation, alert, owner — nối sang Day 11 nếu có guardrail.

- **Expectation `required_doc_types_present`**: phát hiện pipeline bỏ sót nguồn dữ liệu.
- **Expectation `hr_leave_no_stale_10d_annual`**: phát hiện HR version conflict.
- **Expectation `refund_no_stale_14d_window`**: phát hiện refund policy stale.
- **Freshness alert**: `freshness_check.py` chạy định kỳ, alert nếu FAIL.
- **Owner**: mỗi doc_id có owner chịu trách nhiệm cập nhật khi source thay đổi.
- **Golden eval**: chạy `eval_retrieval.py` sau mỗi pipeline run để verify retrieval quality.
