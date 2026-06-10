# Kiến trúc pipeline — Lab Day 10

**Nhóm:** Day10-Group  
**Cập nhật:** 2026-06-10

---

## 1. Sơ đồ luồng (bắt buộc có 1 diagram: Mermaid / ASCII)

```
┌─────────────┐    ┌──────────────┐    ┌──────────────────┐    ┌─────────────┐    ┌──────────────┐
│  Raw CSV    │───▶│  Clean       │───▶│  Validate        │───▶│  Embed      │───▶│  Serving     │
│  (5 nguồn)  │    │  (rules.py)  │    │  (expectations)  │    │  (Chroma)   │    │  (Day 09)    │
└─────────────┘    └──────┬───────┘    └────────┬─────────┘    └─────────────┘    └──────────────┘
                         │                      │
                         ▼                      ▼
                  ┌──────────────┐    ┌──────────────────┐
                  │  Quarantine  │    │  Log + Manifest  │
                  │  (audit CSV) │    │  (run_id, count) │
                  └──────────────┘    └──────────────────┘
```

> **Freshness** đo ở manifest (latest_exported_at vs now). **run_id** ghi trên mọi artifact và metadata Chroma.

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | Owner nhóm |
|------------|-------|--------|--------------|
| Ingest | `data/raw/*.csv` | List[Dict] rows | Ingestion Owner |
| Transform | Raw rows | cleaned CSV + quarantine CSV | Cleaning/Quality Owner |
| Quality | Cleaned rows | Expectation results (pass/fail) | Cleaning/Quality Owner |
| Embed | Cleaned CSV | Chroma collection (upsert) | Embed Owner |
| Monitor | Manifest JSON | freshness PASS/WARN/FAIL | Monitoring/Docs Owner |

---

## 3. Idempotency & rerun

> Mô tả: upsert theo `chunk_id` hay strategy khác? Rerun 2 lần có duplicate vector không?

- **chunk_id** = `{doc_id}_{seq}_{sha256_16}` — hash ổn định theo nội dung, không đổi giữa các lần rerun.
- Chroma `upsert` theo chunk_id → idempotent: rerun 2 lần không tạo duplicate.
- **Prune**: sau embed, xóa vector id không còn trong cleaned run hiện tại (snapshot publish). Log `embed_prune_removed=N`.
- Rerun 2 lần → `embed_upsert count` giữ nguyên, `embed_prune_removed=0` ở lần 2.

---

## 4. Liên hệ Day 09

> Pipeline này cung cấp / làm mới corpus cho retrieval trong `day09/lab` như thế nào? (cùng `data/docs/` hay export riêng?)

- Cùng `data/docs/` (5 tài liệu gốc: policy, SLA, FAQ, HR, access control).
- Pipeline Day 10 embed vào collection `day10_kb` — agent Day 09 query collection này để lấy context.
- Khi pipeline re-run → Chroma upsert → agent tự động thấy data mới (cùng collection).

---

## 5. Rủi ro đã biết

- **Freshness SLA FAIL**: data sample có `exported_at` cũ (tháng 4/2026) — SLA 24h sẽ luôn FAIL. Cần cập nhật timestamp hoặc điều chỉnh SLA cho phù hợp với data snapshot.
- **Embedding model CPU**: `all-MiniLM-L6-v2` chạy CPU, chậm hơn GPU nhưng đủ cho lab (~36 chunks).
- **Dedup aggressive**: normalize repeated words có thể merge nhầm chunk có nội dung tương tự nhưng khác ngữ cảnh.
