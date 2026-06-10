# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Bùi Văn Tuân
**MSSV:** 2A202601006
**Nhóm:** C1
**Email:** tuanbvtlu@gmail.com
**Vai trò:** Ingestion + Cleaning/Quality + Embed/Idempotency + Monitoring/Docs
**Ngày nộp:** 2026-06-10 · **run_id:** `day10-run` (inject: `inject-bad`)

---

## 1. Tôi phụ trách phần nào?

Vì làm một mình, tôi đi hết pipeline: ingest CSV (`etl_pipeline.py`), cleaning + expectation (`transform/cleaning_rules.py`, `quality/expectations.py`), embed idempotent vào Chroma (`day10_kb`), và monitoring/docs (`monitoring/freshness_check.py`, `docs/*`, `contracts/data_contract.yaml`). Trọng tâm công sức nằm ở tầng **clean + validate** — nơi quyết định data nào được embed. Tôi thêm **3 cleaning rule** và **3 expectation** so với baseline, đánh dấu `[FIX]/[NEW Rn]/[NEW En]` trong code và ghi bảng `metric_impact` ở `reports/group_report.md`.

## 2. Một quyết định kỹ thuật

Tôi để expectation **`access_control_sop_present` ở mức `halt`** (không phải `warn`). Lý do: allowlist là điểm dễ hồi quy nhất — chỉ cần ai đó bỏ `access_control_sop` khỏi `ALLOWED_DOC_IDS`, toàn bộ nguồn này bị quarantine `unknown_doc_id`, `gq_d10_10` fail **âm thầm** mà pipeline vẫn xanh. Đặt `halt` biến lỗi cấu hình thành lỗi dừng pipeline ngay tầng validate, không để lọt xuống grading. Ngược lại, `no_noise_marker` tôi để `warn` vì nhiễu sót lại không làm sai nghiệp vụ — đúng tinh thần "không phải lỗi nào cũng halt" (slide warn/quarantine/halt).

## 3. Một lỗi / anomaly đã xử lý

**Triệu chứng:** sau khi đã thêm `access_control_sop` vào allowlist, pipeline **vẫn HALT**. **Phát hiện:** log báo `expectation[hr_leave_no_stale_10d_annual] FAIL (halt)`. **Chẩn đoán:** baseline chỉ quarantine HR theo `effective_date < 2026-01-01`, nhưng raw CSV có chunk `"10 ngày phép năm (bản HR 2025)"` lại gắn **ngày 2026** (vd `2026-03-30`, `2026-01-09`) nên lọt qua filter ngày. **Fix:** thêm rule R1 `stale_hr_version_marker` quarantine theo **nội dung** (marker), độc lập với ngày → 8 chunk vào quarantine, E6 chuyển OK, `gq_d10_09` trả đúng "12 ngày".

## 4. Bằng chứng trước / sau

Từ `artifacts/eval/before_after_eval.csv` (chạy `inject-bad` → `day10-run`):

```
q_refund_window:  before_forbidden = yes   →   after_forbidden = no   (top1 = policy_refund_v4)
```

Trước fix, chunk "14 ngày làm việc" lọt top-k (`expectation[refund_no_stale_14d_window] FAIL :: violations=1`); sau khi chạy chuẩn, `embed_prune_removed=1` xoá vector cũ, grading đạt **10/10 PASS**. Idempotency: chạy `run` 2 lần, `collection.count()` giữ nguyên **35**.

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ: cho `cleaning_rules.py` đọc `hr_leave_min_effective_date` và `hr_leave_stale_markers` **trực tiếp từ `contracts/data_contract.yaml`** lúc runtime, thay vì hằng số trong code — bỏ hard-code, đổi version chỉ cần sửa contract (tiêu chí Distinction (d)).
