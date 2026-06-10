# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** C1
**Thành viên:**
| Tên | MSSV | Vai trò (Day 10) | Email |
|-----|------|------------------|-------|
| Bùi Văn Tuân | 2A202601006 | Ingestion + Cleaning/Quality + Embed/Idempotency + Monitoring/Docs (solo) | tuanbvtlu@gmail.com |

**Ngày nộp:** 2026-06-10
**Repo:** `day10/lab`
**Độ dài khuyến nghị:** 600–1000 từ

---

> run_id canonical: **`day10-run`** · run inject: **`inject-bad`**
> Artifact: `artifacts/manifests/`, `artifacts/quarantine/`, `artifacts/eval/`

---

## 1. Pipeline tổng quan

**Tóm tắt luồng:** `policy_export_dirty.csv` (247 rows, 5 hệ thống nguồn) → clean (allowlist, date ISO, HR stale, noise strip, dedupe, refund fix, PII mask) → validate (E1..E9, halt có kiểm soát) → embed Chroma (upsert + prune theo `chunk_id`) → manifest + freshness. Serving lại cho retrieval Day 08/09 (cùng `data/docs/`).

**Lệnh chạy một dòng:**

```bash
python etl_pipeline.py run --run-id day10-run && python grading_run.py --out artifacts/eval/grading_run.jsonl
```

`run_id` lấy từ dòng `run_id=...` đầu log (`artifacts/logs/run_day10-run.log`) và trong manifest.

---

## 2. Cleaning & expectation

Baseline đã có 6 hành vi (allowlist, date, HR-date, missing, dedupe, refund). Nhóm **thêm 3 rule + 3 expectation** (1 warn, còn lại halt).

### 2a. Bảng metric_impact (chống trivial)

| Rule / Expectation mới                 | Trước                                      | Sau / khi inject                         | Chứng cứ                                                        |
| -------------------------------------- | ------------------------------------------ | ---------------------------------------- | --------------------------------------------------------------- |
| [FIX] allowlist += access_control_sop  | 0 chunk acl (counterfactual)               | 6 chunk acl trong cleaned                | gq_d10_10 top1=access_control_sop                               |
| [R1] stale_hr_version_marker           | E6 **FAIL** (10 ngày leak qua filter ngày) | 8 chunk → quarantine, E6 **OK**          | `quarantine_day10-run.csv` reason=`stale_hr_version_marker` (8) |
| [R2] strip_noise_markers               | 19 row có noise/doubled                    | strip → tăng dedup + đẩy về missing_text | `no_noise_marker` OK; dup=58                                    |
| [R3] mask_pii_email                    | 2 chunk có @company.internal               | 0 (→ `[email-redacted]`)                 | E8 `no_internal_email_pii` OK (pii=0)                           |
| [E7] access_control_sop_present (halt) | n/a                                        | rows=6 OK                                | log expectation                                                 |
| [E8] no_internal_email_pii (halt)      | leak 2                                     | 0 OK                                     | log expectation                                                 |
| [E9] no_noise_marker (warn)            | noise>0                                    | 0 OK                                     | log expectation                                                 |

**Ví dụ expectation fail (inject):** `expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1` → bị `--skip-validate` bỏ qua để demo data bẩn.

---

## 3. Before / after ảnh hưởng retrieval

**Kịch bản inject:** `--no-refund-fix --skip-validate` → chunk "14 ngày làm việc" lọt index.

**Kết quả định lượng** (`artifacts/eval/before_after_eval.csv`):

| Câu             | before forbidden | after forbidden |
| --------------- | ---------------- | --------------- |
| q_refund_window | **yes**          | **no**          |

Grading sau fix: **10/10 PASS** (`gq_d10_01..10`), gồm gq_09 (HR 12 ngày) và gq_10 (access control).

---

## 4. Freshness & monitoring

SLA 24h, đo ở **publish boundary** (`latest_exported_at`). Data mẫu → `freshness_check=FAIL` (age ≈1470h) — đúng kỳ vọng vì export cũ; production alert qua `#data-quality`.

---

## 5. Liên hệ Day 09

Corpus sau embed phục vụ lại retrieval/multi-agent Day 09 (cùng 5 doc `data/docs/`). Tách collection `day10_kb` để inject corruption không phá index Day 09.

---

## 6. Peer review (3 câu hỏi slide 42)

1. **Rerun có duplicate không?** Không — upsert theo `chunk_id` + prune; count giữ 35 sau 2 lần.
2. **Freshness đo ở đâu?** Publish boundary (`latest_exported_at` trong manifest).
3. **Record bị flag đi đâu?** `artifacts/quarantine/quarantine_<run_id>.csv` kèm `reason` (không silent drop).

## 7. Rủi ro còn lại

- Embedding cần model multilingual (đã set trong `.env.example`); MiniLM-L6 English-only fail gq_06.
- Chưa có LLM-judge; freshness mới 1 boundary.
