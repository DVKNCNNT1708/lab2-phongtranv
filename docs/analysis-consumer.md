# Phân tích yêu cầu — (Nhóm B1 - IoT Ingestion)

- Cặp đàm phán: Pair 05
- Product: B
- Consumer service: IoT Ingestion
- Provider service: Core Business
- Người viết: Trần Văn Phong (1771020536)
- Ngày: 2026-05-12

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `Sensor Event` | Đẩy dữ liệu thô thu thập từ thiết bị vật lý (cảm biến nhiệt độ, chuyển động) về hệ thống Core. | `device_id`, `timestamp`, `sensor_type`, các giá trị đo đạc (`temperature_value` hoặc `is_detected`). | `battery_level` (có thể gửi `null` nếu thiết bị cắm điện trực tiếp). |
| `Device Status` | Báo cáo định kỳ trạng thái thiết bị đang hoạt động (keep-alive/heartbeat). | `device_id`, `status` (online/offline). | `uptime`, `firmware_version`. |

---

## 2. API Consumer cần gọi

| Method | Path | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| POST | `/api/v1/sensors/events` | Ngay khi cảm biến phát hiện sự kiện mới hoặc định kỳ cập nhật thông số môi trường. | `201 Created` (Ghi nhận thành công, không block luồng xử lý). |
| POST | `/api/v1/sensors/{device_id}/heartbeat` | Định kỳ 15-30 phút/lần khi không có event nào để báo thiết bị vẫn "sống". | `200 OK` (Cập nhật trạng thái thành công). |
| GET | `/api/v1/sensors/{device_id}` | Khi khởi động thiết bị, kiểm tra xem Core đã nhận diện và cho phép thiết bị này hoạt động chưa. | `200 OK` + thông tin cấu hình cơ bản, hoặc `404 Not Found`. |

---

## 3. Error case Consumer cần xử lý

Tối thiểu 5 case.

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| 400 | Request sai schema | Ghi log cục bộ, kiểm tra lại cấu trúc JSON được sinh ra từ firmware IoT. |
| 401 | Thiếu token/Token sai | Ngừng gửi, báo lỗi về local gateway, thử kết nối lại với token dự phòng (nếu có). |
| 404 | Thiết bị chưa đăng ký trên Core | Chuyển thiết bị sang trạng thái "Chờ đăng ký", tạm thời đưa event vào local queue. |
| 422 | Dữ liệu đo đạc vô lý (VD: 9999 độ C) | Vứt bỏ (drop) gói tin, kích hoạt còi báo lỗi cảm biến cần bảo trì. |
| 429 | Spam request quá nhanh | Kích hoạt Exponential Backoff (chờ 1s, 2s, 4s... rồi mới gửi lại). |
| 500/503 | Server Core sập hoặc quá tải | Lưu sự kiện vào hàng đợi bộ nhớ đệm (local storage) và retry lại sau khi có mạng. |

---

## 4. Giả định bổ sung

- Giả định 1: Core Business có khả năng chịu tải cao, phản hồi nhanh (dưới 200ms) để không làm treo bộ nhớ hạn hẹp của các bo mạch IoT.
- Giả định 2: Core Business sẽ chịu trách nhiệm đọc trường `sensor_type` (đóng vai trò discriminator) để tự động rẽ nhánh xử lý logic phù hợp.
- Giả định 3: Mạng cục bộ hoặc mạng viễn thông có thể chập chờn, API cần chấp nhận timestamp lùi lại trong quá khứ do dữ liệu được gửi chậm.

---

## 5. Câu hỏi cho Provider

1. Bên Core Business có thiết lập Rate Limit (giới hạn số request/giây) cho từng `device_id` không? Nếu có thì bao nhiêu?
2. Format thời gian bắt buộc cho trường `timestamp` là chuẩn ISO 8601 (VD: `2026-05-12T09:00:00Z`) hay Unix epoch?
3. Với yêu cầu Union type `null` của Lab 02, chúng tôi sẽ cấu hình trường `battery_level: [integer, "null"]`. Phía Core đã sẵn sàng bắt giá trị null chưa?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider đổi URL hoặc đổi kiểu dữ liệu | Toàn bộ thiết bị IoT bị lỗi 404/400, mất dữ liệu | Sử dụng Versioning cứng trên URL (`/api/v1/`) và cam kết tương thích ngược. |
| Provider thiếu mã lỗi chuẩn mực | Thiết bị IoT khó bóc tách nguyên nhân lỗi tự động | Thống nhất sử dụng chung chuẩn RFC 7807 (Problem Details) cho mọi lỗi 4xx/5xx. |
| Mạng chập chờn gây mất kết nối giữa chừng | Mất mát các sự kiện cảnh báo quan trọng (báo cháy, đột nhập) | Xây dựng cơ chế Local Queue tại IoT Edge, retry liên tục cho đến khi nhận được mã 2xx từ Provider. |