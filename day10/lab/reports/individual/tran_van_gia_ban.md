# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Trần Văn Gia Bân  
**Vai trò:** Monitoring & Ingestion
**Ngày nộp:** 15/04/2026  
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

Tôi chịu trách nhiệm thiết lập luồng nạp dữ liệu (Ingestion) ở Sprint 1 và cấu hình hợp đồng dữ liệu (Data Contract). Cụ thể, tôi quản lý file khởi chạy etl_pipeline.py.

**Kết nối với thành viên khác:**

Vai trò của tôi nằm ở đầu chuỗi pipeline. Tôi phụ trách chạy thử nghiệm luồng raw data (file policy_export_dirty.csv) để trích xuất ra file artifacts/quarantine/quarantine_sprint1.csv. Sau khi có file cách ly này, tôi bàn giao nó cho thành viên phụ trách Cleaning / Quality Owner để họ có cơ sở phân tích lỗi và viết thêm các rule làm sạch mới cho Sprint 2. Ngoài ra, tôi cũng phối hợp với nhóm để chốt chuẩn schema đầu vào.
_________________

**Bằng chứng (commit / comment trong code):**
5ccdfffb2bba3a27af94d76baec83cc4ad214cff
_________________

---

## 2. Một quyết định kỹ thuật (100–150 từ)

> VD: chọn halt vs warn, chiến lược idempotency, cách đo freshness, format quarantine.

Khi chạy pipeline nghiệm thu ở Sprint 1, tôi quyết định không sử dụng tính năng tự động sinh run_id bằng timestamp mặc định của hệ thống (UTC), mà chủ động gán cứng tham số cờ --run-id sprint1 (python etl_pipeline.py run --run-id sprint1).

Quyết định này nhằm mục đích thiết lập Baseline (đường cơ sở) rõ ràng cho toàn bộ pipeline trước khi các thành viên khác bắt đầu viết code làm sạch. Bằng cách định danh rõ sprint1, các file artifacts sinh ra (như cleaned_sprint1.csv, quarantine_sprint1.csv, và manifest_sprint1.json) sẽ không bị lẫn lộn với các lần test sau này của đội Cleaning. Điều này giúp nhóm dễ dàng rollback (quay lui) và so sánh metric Before/After khi làm Báo cáo chất lượng ở Sprint 3.
_________________

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

> Mô tả triệu chứng → metric/check nào phát hiện → fix.

Trong quá trình chạy Ingestion, thay vì chỉ quan tâm đến việc data có chạy qua hay không, tôi đã giám sát file manifest và phát hiện ra một Anomaly (bất thường) lớn về Data Observability.

Triệu chứng: Pipeline báo PIPELINE_OK, data vẫn được embed thành công 6 records vào ChromaDB, nhưng hệ thống monitoring lại quăng ra một lỗi cảnh báo vi phạm SLA:
freshness_check=FAIL {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": 116.639, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}.

Cách xử lý: Nguyên nhân là do bản export raw CSV từ đầu nguồn (CRM/HR) đã quá cũ (ngày 10/04), có tuổi đời hơn 116 giờ, vượt xa ngưỡng 24 giờ trong Data Contract. Vì đây là cảnh báo mức độ Data Pipeline (chưa gây sập hệ thống RAG), tôi đã ghi nhận log này vào Runbook và báo cáo cho team.
_________________

---

## 4. Bằng chứng trước / sau (80–120 từ)

> Dán ngắn 2 dòng từ `before_after_eval.csv` hoặc tương đương; ghi rõ `run_id`.

question_id,question,top1_doc_id,top1_preview,contains_expected,hits_forbidden,top1_doc_expected,top_k_used
gq_d10_01,"Theo policy hoàn tiền nội bộ, khách có tối đa bao nhiêu ngày làm việc để gửi yêu cầu hoàn tiền sau khi đơn được xác nhận?",policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,no,,3
gq_d10_02,Ticket P1: thời gian resolution SLA là bao nhiêu giờ?,sla_p1_2026,Ticket P1 có SLA phản hồi ban đầu 15 phút và resolution trong 4 giờ.,yes,no,,3
gq_d10_03,"Theo chính sách nghỉ phép hiện hành (2026), nhân viên dưới 3 năm kinh nghiệm được bao nhiêu ngày phép năm?",hr_leave_policy,Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026.,yes,no,yes,3

_________________

---

## 5. Cải tiến tiếp theo (40–80 từ)

> Nếu có thêm 2 giờ — một việc cụ thể (không chung chung).

Nếu có thêm 2 giờ, tôi sẽ tích hợp module freshness_check.py thành một tác vụ chạy tự động (Cronjob/Airflow). Thay vì đợi pipeline chạy xong mới báo FAIL freshness, tôi sẽ đặt một trạm gác (checkpoint) check thẳng tuổi của file policy_export_dirty.csv ngay trước khi nạp vào memory. Nếu file quá 24h, sẽ tự động bắn cảnh báo webhook và Halt luôn luồng Ingest để tiết kiệm compute.
_________________
