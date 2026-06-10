# Runbook — Lab Day 10 (incident tối giản)

Sự cố mẫu: **agent trả lời "14 ngày" cho câu hỏi hoàn tiền** (đúng phải là 7 ngày).

---

## Symptom

- User/CS agent thấy câu trả lời hoàn tiền nói **"14 ngày làm việc"** thay vì "7 ngày".
- Hoặc agent nói nhân viên < 3 năm được **"10 ngày phép"** (bản HR 2025 cũ) thay vì 12 ngày.

---

## Detection

- `eval_retrieval.py` → cột `hits_forbidden = yes` cho `q_refund_window` (chunk "14 ngày" lọt top-k).
- `expectations.py` → `refund_no_stale_14d_window` **FAIL (halt)** hoặc `hr_leave_no_stale_10d_annual` **FAIL**.
- Grading → `gq_d10_01` / `gq_d10_09` có `hits_forbidden = true`.

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi (run day10-run) |
|------|----------|----------------------------------|
| 1 | Mở `artifacts/manifests/manifest_<run_id>.json` | `no_refund_fix: true` hoặc `skipped_validate: true` → biết run "xấu" |
| 2 | Mở `artifacts/quarantine/quarantine_<run_id>.csv` | đếm `reason`: `stale_hr_version_marker=8`, `unknown_doc_id=109`… |
| 3 | `python eval_retrieval.py --out artifacts/eval/eval_after_fix.csv` | `q_refund_window`: `hits_forbidden=no` sau khi fix |

**Thứ tự debug (slide Day 10):** Freshness → Volume → Schema → Lineage → mới đến model/prompt.

---

## Mitigation

- **Ngay lập tức:** chạy lại pipeline chuẩn `python etl_pipeline.py run` (bỏ `--no-refund-fix --skip-validate`) → re-embed bản sạch; `embed_prune` xoá vector cũ.
- **Nếu cần thời gian điều tra:** treo banner "dữ liệu đang cập nhật" / rollback về collection snapshot trước.
- **Xác nhận:** `python grading_run.py` → 10/10 pass trước khi mở lại serving.

---

## Prevention

- **Expectation halt** cho từng failure mode (E3 refund, E6 HR, E8 PII) → pipeline không publish được data bẩn.
- **Cleaning rule R1** chặn HR stale theo nội dung (không chỉ theo ngày).
- **Freshness alert** khi `latest_exported_at` vượt SLA → cảnh báo trước khi user phát hiện.
- Nối Day 11: runbook này là một guardrail vận hành; mở rộng thành post-mortem có action item trên pipeline.
