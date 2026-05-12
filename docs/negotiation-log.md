# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: Pair 05
- Product: B
- Provider: Nhóm B6 (Core Business)
- Consumer: Nhóm B1 (IoT Ingestion)
- Phiên: v1.0
- Ngày: 2026-05-12

---

## Issue #1

- Raised by: Provider
- Endpoint: Global (Toàn bộ API)
- Concern: B1 có thói quen gửi thời gian dạng Unix epoch (số nguyên) cho nhẹ băng thông, nhưng Core Business cần chuẩn format để dễ mapping vào Database.
- Proposal: Bắt buộc dùng chuẩn ISO 8601 (`date-time`) cho trường `timestamp`.
- Resolution: Accepted
- Rationale: Tiêu chuẩn hóa dữ liệu toàn hệ thống, hy sinh một chút hiệu năng của vi điều khiển IoT để đổi lấy sự đồng nhất ở tầng Core.
- Impact: Consumer (B1) phải viết thêm hàm convert thời gian sang ISO 8601 trước khi POST.

---

## Issue #2

- Raised by: Consumer
- Endpoint: `POST /api/v1/sensors/events`
- Concern: Nhiều thiết bị IoT được cắm điện lưới (không dùng pin). Nếu gửi `battery_level: 0` hệ thống Core sẽ báo động giả là hết pin.
- Proposal: Cấu hình trường `battery_level` thành union type: `[integer, "null"]`. B1 sẽ gửi `null` khi cắm điện lưới.
- Resolution: Accepted
- Rationale: Thể hiện chính xác tính chất vật lý của thiết bị. Thỏa mãn tiêu chí dùng union type với `null` của bài Lab.
- Impact: Provider (B6) phải thêm logic kiểm tra null trước khi validate khoảng giá trị 0-100.

---

## Issue #3

- Raised by: Provider
- Endpoint: `POST /api/v1/sensors/events`
- Concern: B1 gửi nhiều loại cảm biến (nhiệt độ, chuyển động) vào chung một endpoint. Rất khó validate nếu payload không có cấu trúc rõ ràng.
- Proposal: Sử dụng cấu trúc đa hình (Polymorphism) bằng từ khóa `oneOf`, kết hợp trường `sensor_type` làm `discriminator`.
- Resolution: Accepted
- Rationale: Giúp một endpoint duy nhất có thể validate chặt chẽ nhiều payload khác nhau, tuân thủ OpenAPI 3.1. Thỏa mãn tiêu chí Lab 02.
- Impact: Code gen ra ở hai bên sẽ tự động rẽ nhánh thành các class/struct kế thừa tương ứng.

---

## Issue #4

- Raised by: Consumer
- Endpoint: Global (Toàn bộ API)
- Concern: Nếu B1 gửi sai cấu trúc dữ liệu, B6 thường trả về string text thông báo lỗi. Code của IoT không thể tự động parse các đoạn text này để xử lý rẽ nhánh.
- Proposal: Chuẩn hóa mọi response lỗi 4xx/5xx theo định dạng `application/problem+json` (RFC 7807).
- Resolution: Accepted
- Rationale: Giúp hệ thống tự động bóc tách mã lỗi (`status`, `detail`) chuẩn xác, dễ debug. Thỏa mãn yêu cầu Lab 02.
- Impact: Provider (B6) phải xây dựng Error Handler middleware tuân thủ chuẩn này.

---

## Issue #5

- Raised by: Provider
- Endpoint: `POST /api/v1/sensors/events`
- Concern: B1 đề nghị tên endpoint là `/api/v1/event` (số ít), điều này vi phạm nguyên tắc thiết kế REST.
- Proposal: Đổi tên path thành danh từ số nhiều `/api/v1/sensors/events`.
- Resolution: Accepted
- Rationale: Tuân thủ chặt chẽ tiêu chuẩn thiết kế RESTful API (thể hiện tính phân cấp và danh từ số nhiều).
- Impact: Consumer (B1) phải cấu hình lại base URL trong mã nguồn HTTP Client.

---

## Issue #6

- Raised by: Consumer
- Endpoint: `POST /api/v1/sensors/{device_id}/heartbeat`
- Concern: Nếu cảm biến không phát hiện chuyển động trong nhiều ngày, Core B6 sẽ không biết thiết bị đó còn sống hay đã mất điện/mất mạng.
- Proposal: Thêm một endpoint riêng dành cho tín hiệu Keep-alive (Heartbeat) định kỳ.
- Resolution: Accepted
- Rationale: Đảm bảo giám sát trạng thái thiết bị thời gian thực, đáp ứng tính sẵn sàng cao (High Availability).
- Impact: Provider (B6) phải mở thêm path mới. Consumer (B1) phải lập lịch gửi ping 30 phút/lần.

---

# Chốt hợp đồng v1.0

Provider sign-off: Đại diện nhóm B6 (Đã duyệt)
Consumer sign-off: Trần Văn Phong - Thành viênnhóm B1 (Đã duyệt)
Witness (GV/TA):
Date: 2026-05-12

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| Không có | File `openapi.yaml` đã pass 100% ruleset của campus-spectral. | N/A |