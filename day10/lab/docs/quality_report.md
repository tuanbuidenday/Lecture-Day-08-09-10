# Quality report — Lab Day 10 (nhóm)

**run_id:** `day10-run` (after) · `inject-bad` (before)
**Ngày:** 2026-06-10
**Embedding:** `paraphrase-multilingual-MiniLM-L12-v2` · collection `day10_kb`

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (inject-bad) | Sau (day10-run) | Ghi chú |
|--------|--------------------|-----------------|---------|
| raw_records | 247 | 247 | cùng nguồn |
| cleaned_records | 35 | 35 | refund chunk khác nội dung (14 vs 7) |
| quarantine_records | 212 | 212 | unknown 109, dup 58, hr_date 22, missing_text 9, hr_marker 8, missing_date 6 |
| Expectation halt? | **Có** (`refund_no_stale_14d_window` FAIL, bị `--skip-validate` bỏ qua) | **Không** | E1..E9 pass |

---

## 2. Before / after retrieval (bắt buộc)

**Câu hỏi then chốt:** refund window (`q_refund_window`)

| | top1_doc_id | contains_expected | hits_forbidden |
|--|-------------|-------------------|----------------|
| **Trước** | policy_refund_v4 | yes | **yes** ← chunk "14 ngày" lọt top-k |
| **Sau** | policy_refund_v4 | yes | **no** ← đã fix 14→7 |

Nguồn: `artifacts/eval/before_after_eval.csv` (21 câu; đúng 1 câu đổi trạng thái — câu refund).

**Versioning HR — `gq_d10_09`:** `contains_expected=true`, `hits_forbidden=false` ("10 ngày phép năm" đã quarantine), `top1_doc_matches=true`.
**Access control — `gq_d10_10`:** trước [FIX] allowlist, `access_control_sop` bị quarantine `unknown_doc_id` → 0 chunk → fail; sau fix top1 = `access_control_sop`.

---

## 3. Freshness & monitor

`freshness_check = FAIL` — `latest_exported_at = 2026-04-10`, age ≈ **1470h** > SLA 24h.
**Giải thích:** data mẫu có `exported_at` cũ → FAIL hợp lý. SLA áp cho *độ tươi snapshot publish*, không phải giờ chạy. Production: alert khi vượt SLA.

---

## 4. Corruption inject (Sprint 3)

Lệnh: `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`
→ bỏ rule fix refund (14 ngày leak) + bỏ qua halt → embed bản bẩn.
Phát hiện: expectation `refund_no_stale_14d_window` FAIL + eval `hits_forbidden=yes`.
Khôi phục: chạy lại `run` chuẩn → `embed_prune_removed=1` xoá chunk "14 ngày" cũ.

---

## 5. Hạn chế & việc chưa làm

- Eval là keyword-over-top-k, chưa có LLM-judge.
- Freshness mới đo 1 boundary (publish); chưa đo ingest boundary riêng.
- Cutoff versioning đã mirror sang contract nhưng code vẫn đọc hằng số nội bộ.
