# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `policy_refund_v4` | CSV export (batch) | Chunk stale "14 ngày" từ bản cũ; duplicate do sync lại | `refund_no_stale_14d_window` expectation; freshness SLA |
| `sla_p1_2026` | CSV export (batch) | Duplicate chunk; noise prefix "Nội dung không rõ ràng:" | `chunk_min_length_8`; volume anomaly (row count) |
| `it_helpdesk_faq` | CSV export (batch) | Chunk rỗng sau strip; duplicate nội dung | `no_empty_doc_id`; dedup rate |
| `hr_leave_policy` | CSV export (batch) | HR version conflict: bản 2025 (10 ngày) lẫn vào 2026 (12 ngày); ngày không ISO | `hr_leave_no_stale_10d_annual`; `effective_date_iso_yyyy_mm_dd` |
| `access_control_sop` | CSV export (batch) | Bị bỏ sót nếu không cập nhật allowlist; chunk rỗng | `required_doc_types_present`; `min_one_row` |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | Stable hash: `{doc_id}_{seq}_{sha256_16}` — idempotent khi rerun |
| doc_id | string | Có | Phải thuộc `ALLOWED_DOC_IDS` (5 nguồn hợp lệ) |
| chunk_text | string | Có | Đã strip noise prefix, normalize repeated words, merge context |
| effective_date | date | Có | ISO `YYYY-MM-DD`; parse từ nhiều format (DD/MM/YYYY, ISO) |
| exported_at | datetime | Có | Timestamp lần export; dùng cho freshness check |

---

## 3. Quy tắc quarantine vs drop

> Record bị flag đi đâu? Ai approve merge lại?

- **Quarantine** (`artifacts/quarantine/`): record bị loại khỏi pipeline nhưng vẫn giữ lại để audit. Ghi `reason` cho mỗi record.
- Các lý do quarantine: `unknown_doc_id`, `missing_effective_date`, `invalid_effective_date_format`, `stale_hr_policy_effective_date`, `stale_hr_policy_content_10d`, `missing_chunk_text`, `duplicate_chunk_text`.
- **Không silent drop** — mọi record bị loại đều có trong quarantine CSV.
- Merge lại: SME hoặc Data Engineer review quarantine CSV, approve nếu record bị flag nhầm (cập nhật allowlist hoặc cleaning rule).

---

## 4. Phiên bản & canonical

> Source of truth cho policy refund: file nào / version nào?

- **policy_refund_v4**: bản hiện hành, cửa sổ hoàn tiền **7 ngày làm việc**. Chunk cũ ghi "14 ngày" là stale — được fix tự động bởi `apply_refund_window_fix`.
- **hr_leave_policy**: bản 2026 — 12 ngày phép năm (dưới 3 năm), 15 ngày (3-5 năm), 18 ngày (trên 5 năm). Bản 2025 (10 ngày) bị quarantine theo cả date (< 2026-01-01) và content ("10 ngày phép năm").
- **Canonical source**: `data/docs/*.txt` là bản clean tham khảo. Raw CSV mô phỏng export từ nhiều hệ thống nguồn.
