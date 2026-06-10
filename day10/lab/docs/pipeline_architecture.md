# Kiến trúc pipeline — Lab Day 10

**Nhóm:** C1
**Cập nhật:** 2026-06-10 · run_id `day10-run`

---

## 1. Sơ đồ luồng

```
                        data/raw/policy_export_dirty.csv  (247 rows, 5 hệ thống nguồn)
                                         │
                                  load_raw_csv()
                                         ▼
        ┌──────────────────────── CLEAN (transform/cleaning_rules.py) ───────────────────────┐
        │ allowlist doc_id → date ISO → HR date<2026 → [R1] HR stale marker → [R2] strip noise │
        │ → missing text → dedupe → refund 14→7 fix → [R3] mask PII email                      │
        └───────────────┬──────────────────────────────────────────────┬─────────────────────┘
                cleaned (35)                                      quarantine (212)
                        │                                                │
                        ▼                                       artifacts/quarantine/*.csv
        VALIDATE (quality/expectations.py)  E1..E9  ── halt? ──► return 2 (PIPELINE_HALT)
                        │ pass
                        ▼
        EMBED (chromadb upsert by chunk_id + prune id thừa)  ── collection: day10_kb
                        │                          ▲ idempotent: rerun không phình (count=35)
                        ▼
        MANIFEST (artifacts/manifests/manifest_<run_id>.json)  ◄── run_id, record counts
                        │
                        ▼
        FRESHNESS (monitoring/freshness_check.py)  PASS/WARN/FAIL theo latest_exported_at
                        │
                        ▼
        SERVING ──► retrieval Day 08 / multi-agent Day 09 (cùng corpus docs)
```

- **Điểm đo freshness:** sau publish (manifest `latest_exported_at`) — phản ánh độ tươi data thật, không phải giờ chạy cron.
- **run_id:** sinh ở đầu `cmd_run` (UTC timestamp hoặc `--run-id`), ghi vào log + manifest + metadata mỗi vector.
- **quarantine:** mọi record bị loại đều ghi `reason` → audit được tại `artifacts/quarantine/`.

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input                     | Output                                   | Owner nhóm             |
| ---------- | ------------------------- | ---------------------------------------- | ---------------------- |
| Ingest     | `policy_export_dirty.csv` | list dict raw (247)                      | Ingestion Owner        |
| Transform  | raw rows                  | cleaned (35) + quarantine (212)          | Cleaning/Quality Owner |
| Quality    | cleaned rows              | results E1..E9 + cờ halt                 | Cleaning/Quality Owner |
| Embed      | cleaned CSV               | Chroma collection `day10_kb` (35 vector) | Embed Owner            |
| Monitor    | manifest JSON             | freshness PASS/WARN/FAIL                 | Monitoring/Docs Owner  |

---

## 3. Idempotency & rerun

- **Khóa ổn định:** `chunk_id = doc_id + seq + sha256(doc_id|chunk_text|seq)[:16]`.
- **Upsert** theo `chunk_id` → chạy lại cùng dữ liệu không tạo bản trùng.
- **Prune:** trước upsert, xoá id có trong collection nhưng không còn trong cleaned (`embed_prune_removed`).
- **Bằng chứng:** chạy `run` 2 lần liên tiếp → `collection.count()` giữ nguyên **35**.

---

## 4. Liên hệ Day 09

Pipeline làm mới corpus cho retrieval Day 08/09 từ **cùng `data/docs/`** (5 doc: refund, SLA, FAQ, HR, access control).
Khác biệt: Day 10 đi qua lớp **export bẩn (CSV) → clean → validate** trước khi embed, mô phỏng đường dữ liệu thật từ DB/API.
Tách collection `day10_kb` để inject corruption không phá index Day 09.

---

## 5. Rủi ro đã biết

- **Embedding model:** corpus tiếng Việt cần model multilingual (`paraphrase-multilingual-MiniLM-L12-v2`); model English-only `all-MiniLM-L6-v2` làm chunk "10 phút" của gq_d10_06 rớt khỏi top-5.
- **Freshness FAIL** trên data mẫu là cố ý (timestamp cũ); production cần `exported_at` thật.
- **Versioning hard-code một phần:** cutoff `2026-01-01` + stale markers nằm trong code; đã mirror vào `contracts/data_contract.yaml` (`policy_versioning`).
