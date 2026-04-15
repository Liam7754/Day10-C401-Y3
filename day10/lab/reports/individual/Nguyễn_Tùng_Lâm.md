# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Tùng Lâm  
**Vai trò:** Cleaning & Quality Owner — Sprint 2  
**Ngày nộp:** 2026-04-15  
**Độ dài yêu cầu:** **400–650 từ**

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `transform/cleaning_rules.py` — thêm Rule A (`internal_migration_note`), Rule B (`missing_or_invalid_exported_at`), Rule C (`chunk_text_too_long`, ngưỡng 800 ký tự)
- `quality/expectations.py` — thêm E7 (`exported_at_not_empty`, halt) và E8 (`all_doc_ids_in_allowlist`, halt)

Tôi phụ trách Sprint 2: mở rộng bộ cleaning rule và expectation suite từ baseline.

**Kết nối với thành viên khác:**

Rule B và E7 liên kết trực tiếp với phần monitoring của Kiều Đức Lâm — trường `exported_at` là đầu vào để tính `latest_exported_at` trong manifest và freshness check. Nếu Rule B không lọc chunk thiếu `exported_at`, freshness check sẽ trả về giá trị sai.

**Bằng chứng:**
Thêm 3 rules và 2 expectations trong lần lượt 

## 2. Một quyết định kỹ thuật (100–150 từ)

**Quyết định: đặt E7 và E8 là severity `halt` thay vì `warn`.**

Ban đầu tôi cân nhắc đặt E7 (`exported_at_not_empty`) là `warn` vì có thể chunk thiếu `exported_at` là dữ liệu cũ hợp lệ. Tuy nhiên sau khi phân tích, tôi quyết định đặt `halt` vì hai lý do:

1. `exported_at` là trường bắt buộc để tính `latest_exported_at` trong manifest, từ đó freshness check mới hoạt động đúng. Chunk thiếu trường này làm lệch kết quả freshness — có thể khiến FAIL bị bỏ qua hoặc PASS khi thực ra dữ liệu stale.

2. E7 và E8 là "belt-and-suspenders" — chúng chỉ kích hoạt nếu cleaning rule B hoặc allowlist check có bug. Nếu pipeline hoạt động đúng, hai expectation này luôn PASS. Đặt `halt` đảm bảo bất kỳ lỗi lập trình nào trong cleaning cũng bị chặn trước khi embed, tránh đưa dữ liệu xấu vào vector store một cách thầm lặng.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Anomaly: chunk row 3 chứa "14 ngày làm việc" nhưng KHÔNG bị bắt bởi rule refund-fix.**

Triệu chứng: khi tôi xem `quarantine_sprint2.csv`, row 3 bị quarantine với reason `internal_migration_note`, không phải `stale_refund_window`. Điều này có nghĩa Rule A (migration marker) bắt chunk này trước khi rule refund-fix có cơ hội chạy.

Phân tích: thứ tự kiểm tra trong `clean_rows()` đặt dedup và Rule A (migration marker) trước bước `apply_refund_window_fix`. Row 3 có `(ghi chú:` trong text → Rule A quarantine ngay, không còn đến bước fix refund.

Hệ quả: kịch bản `--no-refund-fix` một mình không đủ để inject chunk "14 ngày" vào vector store vì Rule A đã chặn trước. Tôi ghi nhận điều này vào group report (mục 3) và đề xuất kịch bản inject đúng là thêm một dòng mới không có migration marker, sau đó mới dùng `--no-refund-fix --skip-validate`.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**run_id:** `sprint2`

Trước khi thêm Rule A/B/C (chỉ với baseline): raw 12 dòng → cleaned 4, quarantine 4.  
Sau khi thêm Rule A/B/C (sprint2): raw 12 dòng → **cleaned 5, quarantine 7**.

3 dòng tăng thêm trong quarantine đến từ rule mới:

```
row 3  → internal_migration_note      (Rule A)
row 11 → missing_or_invalid_exported_at  (Rule B)
row 12 → chunk_text_too_long_1021_chars  (Rule C)
```

Kết quả eval (kịch bản inject so với clean run):

| question_id | Inject | Clean |
|-------------|--------|-------|
| q_refund_window | hits_forbidden=yes | hits_forbidden=no |
| q_leave_version | contains_expected=no | contains_expected=yes |

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ mở rộng `_MIGRATION_MARKERS` thành một danh sách được load từ file config (`.yaml` hoặc `.env`) thay vì hardcode trong `cleaning_rules.py`. Điều này giúp team thêm marker mới (vd `[staging]`, `[test-only]`) mà không cần sửa code, chỉ cần cập nhật config và rerun pipeline — giảm rủi ro migration artifact không có marker chuẩn lọt vào cleaned.