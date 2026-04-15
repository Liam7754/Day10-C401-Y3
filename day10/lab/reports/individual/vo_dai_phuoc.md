# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Võ Đại Phước  
**Vai trò:** Quality Owner
**Ngày nộp:** 15/04/2026  
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

Chịu trách nhiệm chính cho sprint3, chạy pipeline chuẩn với file gốc và sau đó chạy thí nghiệm với file policy export đã bị chỉnh sửa, thêm lỗi

**Kết nối với thành viên khác:**

phải chờ người hoàn thành sprint 1 và 2 để done pipeline ở sprint 1, chỉnh expectation và cleaning rule trong sprint 2 thì mới chạy được đánh giá trên 2 phiên bản của policy export dirty csv file
_________________

**Bằng chứng (commit / comment trong code):**
b8b0c3bc3ab5aaa721e55752e131523bf90020b7
dafcfe2b45c9dafaed6666e59c2e03dcdf5d453d
5ad596b7f931ab898a9db284be486647683ab0a1
_________________

---

## 2. Một quyết định kỹ thuật (100–150 từ)

> VD: chọn halt vs warn, chiến lược idempotency, cách đo freshness, format quarantine.

Tinh chỉnh [file](../../data/raw/inject_policy_export_dirty.csv) một số lỗi sai phổ biến của dữ liệu để test xem pipeline có đang chuẫn hay không ví dụ như thiếu effective_date, dữ liệu bị duplicate
_________________

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)
_________________

---

## 4. Bằng chứng trước / sau (80–120 từ)

> Dán ngắn 2 dòng từ `before_after_eval.csv` hoặc tương đương; ghi rõ `run_id`.

`before_after_eval.csv` chạy với `granding_questions.json`
gq_d10_01,"Theo policy hoàn tiền nội bộ, khách có tối đa bao nhiêu ngày làm việc để gửi yêu cầu hoàn tiền sau khi đơn được xác nhận?",policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,no,,3
gq_d10_02,Ticket P1: thời gian resolution SLA là bao nhiêu giờ?,sla_p1_2026,Ticket P1 có SLA phản hồi ban đầu 15 phút và resolution trong 4 giờ.,yes,no,,3

_________________

---

## 5. Cải tiến tiếp theo (40–80 từ)

> Nếu có thêm 2 giờ — một việc cụ thể (không chung chung).
Hoàn thành lại file quality report, vì chưa tổng hơp lại được kết quả giữa 2 lần chạy
_________________
