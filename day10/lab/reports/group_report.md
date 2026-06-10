# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Day10-Group  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| ___ | Ingestion / Raw Owner | ___ |
| ___ | Cleaning & Quality Owner | ___ |
| ___ | Embed & Idempotency Owner | ___ |
| ___ | Monitoring / Docs Owner | ___ |

**Ngày nộp:** 2026-06-10  
**Repo:** ___________  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

**Nguồn raw:** CSV export mô phỏng 5 hệ thống nguồn (`data/raw/policy_export_dirty.csv` — 247 records, 15+ doc_id). Chứa duplicate, ngày không ISO, HR version conflict, chunk stale, noise prefix.

**Chuỗi lệnh end-to-end:**

```bash
python etl_pipeline.py run                                    # ingest → clean → validate → embed
python eval_retrieval.py --out artifacts/eval/eval_after_fix.csv  # eval 21 câu
python grading_run.py --out artifacts/eval/grading_run.jsonl      # grading 10 câu
```

**run_id:** ghi trong log mỗi lần chạy (VD: `run_id=2026-06-10T08-07Z`). Manifest tại `artifacts/manifests/manifest_<run-id>.json`.

**Kết quả:** 247 raw → 36 cleaned + 211 quarantine. Pipeline exit 0, grading 10/10, eval 21/21.

---

## 2. Cleaning & expectation (150–200 từ)

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| Strip noise prefix ("Nội dung không rõ ràng:", "!!!") | ~8 chunk bị prefix rác, giảm retrieval quality | 0 noise prefix rows (expectation `no_noise_prefix` pass) | `expectation[no_noise_prefix] OK (warn) :: noise_prefix_rows=0` |
| Normalize repeated words ("làm việc làm việc") | 2 chunk có lặp từ, tạo duplicate ảo khi dedup | 0 chunk lặp từ, dedup chính xác hơn | Row 18, 109 raw CSV → normalized trong cleaned |
| Content-based HR stale ("10 ngày phép năm") | 2 bản HR 2026 nhưng nội dung cũ lọt qua filter date | 0 violations (`hr_leave_no_stale_10d_annual` pass) | `expectation[hr_leave_no_stale_10d_annual] OK (halt) :: violations=0` |
| `required_doc_types_present` (expectation mới) | Không kiểm tra đủ 5 doc type → `access_control_sop` bị bỏ sót | 5/5 doc type có trong cleaned | `expectation[required_doc_types_present] OK (halt) :: missing=none` |
| `no_noise_prefix` (expectation mới) | Không phát hiện prefix rác trong cleaned | 0 noise prefix rows | Log: `noise_prefix_rows=0` |

**Rule chính (baseline + mở rộng):**
- Baseline: allowlist doc_id, parse ngày ISO, HR stale date (< 2026-01-01), refund 14→7 ngày fix, dedup nội dung
- Mới: strip noise prefix, normalize repeated words, content-based HR stale, merge SLA escalation context

**Ví dụ 1 lần expectation fail:**
- Chạy pipeline lần đầu (không sửa code): `expectation[hr_leave_no_stale_10d_annual] FAIL (halt) :: violations=2` — 2 bản HR 2026 có "10 ngày phép năm" lọt qua filter date. Fix: thêm content-based filter.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

**Kịch bản inject:** `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`
- Bỏ qua refund window fix → chunk "14 ngày làm việc" vẫn nằm trong cleaned
- Vẫn embed dù expectation fail (`refund_no_stale_14d_window` FAIL)

**Kết quả định lượng:**

| Metric | inject-bad (trước) | fix-good (sau) | Diff |
|--------|-------------------|----------------|------|
| `refund_no_stale_14d_window` | FAIL (violations=1) | OK (violations=0) | FAIL→OK |
| Grading `gq_d10_01` (refund window) | `hits_forbidden=true` → FAIL | `hits_forbidden=false` → OK | FAIL→OK |
| Eval `q_refund_window` | `contains_expected=yes`, `hits_forbidden=yes` → FAIL | `contains_expected=yes`, `hits_forbidden=no` → OK | FAIL→OK |
| Grading tổng | 9/10 | 10/10 | +1 |
| Eval tổng | 20/21 | 21/21 | +1 |

**Artifact eval:**
- `artifacts/eval/after_inject_bad.csv` — eval khi data có chunk stale
- `artifacts/eval/after_fix_good.csv` — eval khi data đã clean
- `artifacts/eval/grading_inject_bad.jsonl` — grading khi data có chunk stale
- `artifacts/eval/grading_fix_good.jsonl` — grading khi data đã clean

**Before (inject-bad):** Chunk refund chứa "14 ngày làm việc" → retrieval trả về câu trả lời sai → `hits_forbidden=true` trên grading.

**After (fix-good):** Chunk refund đã fix thành "7 ngày làm việc" → retrieval trả lời đúng → grading 10/10.

---

## 4. Freshness & monitoring (100–150 từ)

**SLA:** 24 giờ (configurable qua env `FRESHNESS_SLA_HOURS`).

**Kết quả trên data mẫu:** `freshness_check=FAIL` — `latest_exported_at=2026-04-10T00:00:00`, age ~1472 giờ, vượt SLA 24 giờ. **FAIL là hợp lý** vì data sample có timestamp cũ.

**Giải thích:** SLA freshness áp cho "pipeline run" (thời gian từ lần export gần nhất đến lần chạy pipeline). Data mẫu mô phỏng export cũ → FAIL. Trong production, cần data source cập nhật timestamp.

**Monitoring:** `monitoring/freshness_check.py` đọc manifest và so sánh `latest_exported_at` với thời gian hiện tại.

---

## 5. Liên hệ Day 09 (50–100 từ)

Pipeline Day 10 embed vào collection `day10_kb` trong Chroma. Agent Day 09 query collection này để lấy context cho retrieval. Khi pipeline re-run → Chroma upsert → agent tự động thấy data mới (cùng collection, cùng `data/docs/`).

---

## 6. Rủi ro còn lại & việc chưa làm

- **Freshness SLA FAIL**: data sample có timestamp cũ → cần cập nhật data hoặc điều chỉnh SLA
- **Embedding model CPU**: `all-MiniLM-L6-v2` chạy CPU, chậm hơn GPU
- **Dedup aggressive**: normalize repeated words có thể merge nhầm chunk tương tự
- **Inject scope nhỏ**: chỉ test refund stale; có thể mở rộng inject HR version conflict hoặc missing doc type
- **Sprint 4**: chưa hoàn thiện individual report (template sẵn trong `reports/individual/template.md`)

---

## 7. Peer review (3 câu hỏi)

| Câu hỏi | Trả lời |
|---------|---------|
| **1. Rerun 2 lần có duplicate embedding không?** | Không. chunk_id ổn định (hash SHA256 theo nội dung) + Chroma upsert theo chunk_id → idempotent. Rerun 2 lần: `embed_upsert count=36` lần 1, `embed_upsert count=36` lần 2, collection count = 36 (không phình). |
| **2. Freshness đo ở đâu: ingest, cleaned, hay publish?** | Đo ở **publish** (manifest ghi `latest_exported_at` từ cleaned CSV, so sánh với thời gian chạy pipeline). Nếu chỉ đo ingest mà không đo publish → có thể pipeline green nhưng user vẫn thấy data cũ. |
| **3. Flag/quarantine đi đâu? Ai approve merge lại?** | Record bị flag vào `artifacts/quarantine/quarantine_<run-id>.csv` với cột `reason`. Không silent drop — mọi record bị loại đều có trong quarantine CSV. SME hoặc Data Engineer review quarantine CSV, approve nếu record bị flag nhầm (cập nhật allowlist hoặc cleaning rule). |
