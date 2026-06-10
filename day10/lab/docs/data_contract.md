# Data contract — Lab Day 10

> Đồng bộ với `contracts/data_contract.yaml` (v1.1). run_id `day10-run`.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `policy_refund_v4` (Policy CMS) | export CSV | chunk stale "14 ngày" lẫn "7 ngày"; PII email | `refund_no_stale_14d_window` (halt); `no_internal_email_pii` (halt) |
| `hr_leave_policy` (HR system) | export CSV | xung đột version (10 vs 12 ngày phép); ngày non-ISO | `hr_leave_no_stale_10d_annual` (halt); `effective_date_iso` (halt) |
| `sla_p1_2026` (SLA wiki) | export CSV | duplicate chunk; chunk rỗng | `no_duplicate_chunk_text` (warn) |
| `it_helpdesk_faq` (Helpdesk KB) | export CSV | duplicate; noise prefix | `no_noise_marker` (warn) |
| `access_control_sop` (Security SOP) | export CSV | **thiếu trong allowlist baseline**; chunk rỗng | `access_control_sop_present` (halt) |
| `invalid_doc_*`, `legacy_*`, `security_policy`, `data_privacy_guideline` | export lỗi / chưa đăng ký | doc_id ngoài allowlist | `unknown_doc_id` → quarantine (109 dòng) |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | `doc_id_seq_sha256[:16]` — ổn định, idempotent |
| doc_id | string | Có | Phải ∈ `ALLOWED_DOC_IDS` (5 nguồn canonical) |
| chunk_text | string | Có | ≥8 ký tự; đã strip noise + mask PII |
| effective_date | date | Có | ISO `YYYY-MM-DD` sau chuẩn hoá (DMY → ISO) |
| exported_at | datetime | Có | Dùng cho freshness check |

---

## 3. Quy tắc quarantine vs drop

- **Quarantine (giữ lại + ghi `reason`):** mọi record bị loại → `artifacts/quarantine/quarantine_<run_id>.csv`.
  Lý do quan sát được trên run `day10-run`: `unknown_doc_id` (109), `duplicate_chunk_text` (58),
  `stale_hr_policy_effective_date` (22), `missing_chunk_text` (9), `stale_hr_version_marker` (8),
  `missing_effective_date` (6).
- **Không silent drop:** để audit "vì sao mất dòng" (lineage).
- **Merge lại:** Cleaning/Quality Owner review quarantine; nếu nguồn hợp lệ bị loại nhầm
  (như `access_control_sop` ban đầu) → thêm vào allowlist + đồng bộ contract.

---

## 4. Phiên bản & canonical

- **Refund:** source of truth = `data/docs/policy_refund_v4.txt`, cửa sổ **7 ngày làm việc**. Chunk "14 ngày" là bản cũ → fix hoặc quarantine.
- **HR leave:** canonical = chính sách **2026** (12/15/18 ngày). Marker "10 ngày phép năm" / "(bản HR 2025)" → quarantine bất kể `effective_date`.
- **Cutoff versioning** đặt ở `contracts/data_contract.yaml` → `policy_versioning.hr_leave_min_effective_date: 2026-01-01`.
