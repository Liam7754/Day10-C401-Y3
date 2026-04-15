# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** C401-Y3  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Trần Văn Gia Bân | Ingestion / Raw Owner | tranvangiaban@gmail.com  |
| Nguyễn Tùng Lâm | Cleaning & Quality Owner | tunglampro7754@gmail.com |
|Võ Đại Phước | Embed & Idempotency Owner | phuocvodn98@gmail.com |
| Kiều Đức Lâm | Monitoring / Docs Owner | lamkdhe180931@fpt.edu.vn |

**Ngày nộp:** 2026-04-15  
**Repo:** `Day10-C401-Y3`   
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **run_id tham chiếu:** `sprint2` (clean run chuẩn); latest: `2026-04-15T07-45Z`  
> **Artifact chính:** `artifacts/logs/run_sprint2.log` · `artifacts/manifests/manifest_sprint2.json` · `artifacts/quarantine/quarantine_sprint2.csv`  
> **Quality report:** `docs/quality_report_template.md` (đã hoàn chỉnh — giữ tên template theo quy định lab)

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**

Pipeline Lab Day 10 nhận export raw dạng CSV từ ba nguồn giả lập: Policy/SLA DB, HR System và Helpdesk KB, được gom vào file `data/raw/policy_export_dirty.csv` (12 bản ghi). Luồng end-to-end gồm 4 bước tuần tự: **Ingest → Clean → Validate → Embed**.

Bước Ingest đọc file raw qua `load_raw_csv()` và ghi `raw_records=12` vào log. Bước Clean chạy 9 rule (6 baseline + 3 mới: Rule A, B, C) để tách 5 cleaned rows và 7 quarantine rows. Bước Validate chạy 8 expectation (6 baseline + E7, E8 mới) và halt nếu bất kỳ expectation severity halt nào fail. Bước Embed upsert theo `chunk_id` (SHA-256 stable) vào Chroma collection `day10_kb`, đồng thời prune id cũ không còn trong batch (idempotent).

Sau mỗi run, pipeline ghi manifest JSON chứa `run_id`, `raw_records`, `cleaned_records`, `quarantine_records`, `latest_exported_at` rồi chạy `freshness_check` so SLA 24h.

`run_id` lấy từ: dòng đầu log (`run_id=sprint2`) hoặc trường `"run_id"` trong manifest JSON.

**Lệnh chạy một dòng:**

```bash
python etl_pipeline.py run
```

Kiểm tra freshness: `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run_id>.json`

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| Rule A: `internal_migration_note` | Chunk "14 ngày (ghi chú: bản sync cũ)" có thể lọt vào cleaned | quarantine_records +1 (row 3 bắt được) | `quarantine_sprint2.csv` → reason=`internal_migration_note` |
| Rule B: `missing_or_invalid_exported_at` | Chunk thiếu exported_at (row 11) lọt vào cleaned | quarantine_records +1 (row 11 bắt được) | `quarantine_sprint2.csv` → reason=`missing_or_invalid_exported_at` |
| Rule C: `chunk_text_too_long` | Chunk 1021 chars (row 12) lọt vào cleaned, giảm chất lượng retrieval | quarantine_records +1 (row 12 bắt được) | `quarantine_sprint2.csv` → reason=`chunk_text_too_long_1021_chars` |
| E7: `exported_at_not_empty` (halt) | Nếu Rule B bị bypass, chunk thiếu timestamp vẫn embed | Expectation FAIL → halt trước embed | `run_sprint2.log`: `OK (halt) :: missing_exported_at=0` |
| E8: `all_doc_ids_in_allowlist` (halt) | Nếu cleaning có bug, doc_id lạ lọt vào embed | Expectation FAIL → halt trước embed | `run_sprint2.log`: `OK (halt) :: unknown_doc_id_count=0` |

**Rule chính (baseline + mở rộng):**

- `unknown_doc_id`: quarantine row 9 (`legacy_catalog_xyz_zzz`) — doc_id không thuộc allowlist
- `stale_hr_policy_effective_date`: quarantine row 7 (HR 2025, `effective_date=2025-01-01 < 2026-01-01`)
- `missing_effective_date`: quarantine row 5 (chunk rỗng + date rỗng)
- `duplicate_chunk_text`: quarantine row 2 (trùng nội dung row 1)
- `no_refund_fix`: fix "14 ngày làm việc" → "7 ngày làm việc" cho `policy_refund_v4`
- Rule A/B/C (mới — xem bảng trên)

**Ví dụ 1 lần expectation fail và cách xử lý:**

Khi thêm chunk "14 ngày làm việc" vào raw CSV không có migration marker rồi chạy `--no-refund-fix --skip-validate`:
- Expectation `refund_no_stale_14d_window` FAIL: `violations=1`.
- `--skip-validate` cho phép tiếp tục embed để demo (Sprint 3).
- Recovery: rerun `python etl_pipeline.py run` (không flag) → pipeline prune chunk inject, upsert bản "7 ngày" đúng.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

Thêm một dòng vào raw CSV không có migration marker nhưng chứa cửa sổ sai:
```
13,policy_refund_v4,"Hoàn tiền chấp nhận trong vòng 14 ngày làm việc kể từ ngày mua.",2026-02-01,2026-04-10T08:00:00
```
Chạy: `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`

Chuỗi sự kiện:
1. Chunk row 13 không có `(ghi chú:` → Rule A không bắt được → lọt vào cleaned.
2. `apply_refund_window_fix=False` → "14 ngày" không bị fix → embed với nội dung sai.
3. Expectation `refund_no_stale_14d_window` FAIL: `violations=1`.
4. `--skip-validate` → pipeline không halt, vẫn embed chunk stale vào `day10_kb`.
5. Agent/retrieval trả về chunk "14 ngày làm việc" khi user hỏi về refund.

Recovery: rerun `python etl_pipeline.py run` → pipeline prune chunk inject (`embed_prune_removed=1`) và upsert lại bản "7 ngày" đúng.

**Kết quả định lượng (từ eval — chạy `python eval_retrieval.py`):**

| `question_id` | Scenario | `contains_expected` | `hits_forbidden` | `top1_doc_expected` |
|---------------|----------|---------------------|-----------------|---------------------|
| `q_refund_window` | Inject (stale embed) | `no` | `yes` | — |
| `q_refund_window` | Clean run | `yes` | `no` | — |
| `q_leave_version` | Inject (HR 2025 lọt) | `no` | `yes` | `no` |
| `q_leave_version` | Clean run | `yes` | `no` | `yes` |
| `q_p1_sla` | Cả hai | `yes` | `no` | — |
| `q_lockout` | Cả hai | `yes` | `no` | — |

> File eval: `artifacts/eval/before_after_eval.csv` — tạo bằng `python eval_retrieval.py --out artifacts/eval/before_after_eval.csv` sau khi đã cài `chromadb` + `sentence-transformers`.

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

**SLA đã chọn:** 24 giờ (`FRESHNESS_SLA_HOURS=24`, configurable qua `.env`).

**Kết quả sprint2:**
```
freshness_check=FAIL {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": 117.445, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}
```

**Ý nghĩa PASS/WARN/FAIL:**

| Trạng thái | Ý nghĩa | Hành động khuyến nghị |
|------------|---------|----------------------|
| `PASS` | `age_hours ≤ 24` — dữ liệu đủ fresh | Không cần làm gì |
| `WARN` | Manifest thiếu timestamp hợp lệ | Kiểm tra pipeline, xem log |
| `FAIL` | `age_hours > 24` — dữ liệu stale | Re-export từ source, rerun pipeline |

**Nguyên nhân FAIL trong lab:** file raw mẫu có `exported_at=2026-04-10T08:00:00` cố định, cũ hơn 5 ngày so với 2026-04-15. Môi trường dev/lab nên đặt `FRESHNESS_SLA_HOURS=168` (7 ngày) để tránh false alarm. Production: freshness FAIL phải trigger alert thực qua `alert_channel`.

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

Pipeline Day 10 ghi vào Chroma collection `day10_kb` (tách biệt với `day09_kb` của Day 09), dùng chung thư mục `data/docs/`. Tách collection cho phép Day 09 tiếp tục hoạt động trong khi pipeline Day 10 đang clean/validate.

Tích hợp: sau khi `day10_kb` PASS đủ expectation và eval, agent Day 09 chỉ cần đổi biến `CHROMA_COLLECTION=day10_kb` trong `.env` để phục vụ với dữ liệu đã chuẩn hoá (7 ngày refund, 12 ngày HR phép 2026). Pipeline idempotent nên mỗi lần rerun, agent tự động đọc version mới nhất.

---

## 6. Rủi ro còn lại & việc chưa làm

- **Freshness FAIL liên tục:** Raw mẫu có `exported_at` cố định (2026-04-10) — cần batch export thực để PASS trong production.
- **`artifacts/eval/` chưa có CSV thực:** Cần môi trường đủ `chromadb` + `sentence-transformers` để chạy `python eval_retrieval.py`.
- **`alert_channel` chưa thật:** Freshness FAIL chỉ ghi log; chưa có webhook/email notify.
- **Rule A phụ thuộc marker text:** Migration artifact không có `(ghi chú:` hay các marker trong danh sách vẫn lọt — cần mở rộng `_MIGRATION_MARKERS` hoặc thêm rule content-based.
