# 📘 SỔ TAY KỸ THUẬT & LỘ TRÌNH THỰC HIỆN DỰ ÁN 2
## Đề tài: Trạm giám sát thiết bị công nghiệp qua MQTT & Store-and-Forward (ESP32 + Mosquitto + Docker)

> **Người thực hiện:** Kỹ sư IoT / Embedded  
> **Thời gian:** 4 tuần (Tháng 03/2027 - Tháng 04/2027)  
> **Ghi chú tác giả:** Dự án tập trung vào độ tin cậy cấp công nghiệp (High Reliability), giải quyết triệt để bài toán mất kết nối mạng và nạp phần mềm từ xa không cần cắm dây (OTA).

---

## PHẦN 1: CƠ SỞ LÝ THUYẾT & NGUYÊN LÝ HOẠT ĐỘNG

### 1.1. Giao thức MQTT (Message Queuing Telemetry Transport)
* **Mô hình Publish / Subscribe:** Thiết bị thu thập dữ liệu (Publisher) và ứng dụng nhận dữ liệu (Subscriber) hoàn toàn tách biệt, không cần biết địa chỉ IP của nhau mà chỉ giao tiếp thông qua máy chủ trung gian (**MQTT Broker**).
* **So sánh kỹ thuật HTTP vs MQTT trong hệ thống nhúng:**
  * *Header Overhead:* HTTP tiêu tốn từ 200–800 bytes mỗi request (chứa header, user-agent, cookies...). MQTT chỉ tốn từ **2 bytes** cố định cho mỗi gói tin.
  * *Duy trì kết nối:* HTTP thường đóng kết nối TCP sau mỗi phiên làm việc. MQTT duy trì một kết nối TCP duy nhất thông qua cơ chế gửi gói tin nhịp tim định kỳ (**Keep-Alive Ping**), giúp giảm tải bắt tay TCP bắt buộc.
* **Cơ chế Chất lượng Dịch vụ (Quality of Service - QoS):**
  * **QoS 0 (At most once - Tối đa một lần):** Gửi gói tin đi mà không chờ phản hồi. Nếu mạng rớt, gói tin bị mất. Phù hợp cho dữ liệu nhiệt độ định kỳ đo mỗi 5 giây.
  * **QoS 1 (At least once - Ít nhất một lần):** Broker bắt buộc phải gửi lại gói tin `PUBACK`. Nếu quá thời gian timeout mà chưa nhận được `PUBACK`, thiết bị sẽ gửi lại. Phù hợp cho các sự kiện khẩn cấp, cảnh báo vượt ngưỡng.
  * **QoS 2 (Exactly once - Đúng một lần):** Bắt tay 4 bước phức tạp (`PUBLISH` -> `PUBREC` -> `PUBREL` -> `PUBCOMP`). Tốn tài nguyên RAM và băng thông vi điều khiển, hiếm khi dùng cho thiết bị IoT đầu cuối.
* **Cơ chế Di chúc (Last Will and Testament - LWT):**
  * Khi ESP32 thiết lập kết nối tới Broker, nó gửi kèm một gói tin "di chúc" đăng ký trước (Topic: `devices/ESP32_01/status`, Payload: `{"status": "offline", "reason": "unexpected_disconnect"}`).
  * Nếu ESP32 bị mất nguồn đột ngột hoặc đứt cáp mạng mà không kịp gửi lệnh ngắt kết nối an toàn (`DISCONNECT`), Broker sẽ tự động phát gói tin di chúc này đến tất cả các client đang theo dõi, giúp hệ thống trung tâm phát hiện sự cố sau đúng 1 chu kỳ Keep-Alive!

### 1.2. Bus truyền thông I2C (Inter-Integrated Circuit)
* Giao tiếp 2 dây: `SDA` (Serial Data - GPIO 21) và `SCL` (Serial Clock - GPIO 22).
* Mọi thiết bị trên bus đều có địa chỉ định danh 7-bit duy nhất:
  * Cảm biến môi trường BME280: Địa chỉ `0x76` (hoặc `0x77` tùy chân SDO).
  * Màn hình OLED SSD1306: Địa chỉ `0x3C` (hoặc `0x3D`).
* Cả 2 thiết bị có thể mắc song song vào cùng một cặp chân GPIO của ESP32 mà không hề xung đột, giúp tiết kiệm tối đa số chân vi điều khiển!

---

## PHẦN 2: THUẬT TOÁN STORE-AND-FORWARD TRÊN BỘ NHỚ FLASH

Trong môi trường nhà xưởng công nghiệp, sóng Wi-Fi thường xuyên bị suy hao hoặc mất tín hiệu tạm thời. Dự án này xây dựng cơ chế **Store-and-Forward** chuẩn xác:

```text
       Đọc cảm biến (Mỗi 5s)
                │
                ▼
     Kiểm tra kết nối MQTT?
        ├── Có ────────► Publish trực tiếp lên Broker (QoS 0/1)
        └── Mất kết nối ─► Ghi gói tin vào Flash (LittleFS Circular Buffer)
                                    │
    Khi có kết nối trở lại ────────┘
    Đọc tuần tự file hàng đợi ──► Publish bù với cờ `is_offline: true`
    Xóa bản ghi đã gửi an toàn
```

* **Cơ chế chống hỏng bộ nhớ Flash (Wear Leveling):** Bộ nhớ Flash của vi điều khiển có giới hạn số lần ghi/xóa (khoảng 10.000 đến 100.000 lần). Do đó, hàng đợi được thiết kế dạng **Buffer bộ nhớ RAM (50 gói tin)**; chỉ khi buffer RAM đầy hoặc chuẩn bị mất nguồn mới ghi thành khối (Block Write) xuống file trên LittleFS, giảm tối đa số lần ghi vào Flash.

---

## PHẦN 3: LỘ TRÌNH THỰC HIỆN CHI TIẾT 4 TUẦN

### Tuần 1: Cấu hình Môi trường Broker & Giao tiếp Cảm biến I2C
* [ ] Cài đặt Docker Desktop. Viết file `docker-compose.yml` khởi chạy Eclipse Mosquitto Broker trên port 1883.
* [ ] Cấu hình file `mosquitto.conf`: Tạo tài khoản username/password xác thực và phân quyền danh sách truy cập (ACL).
* [ ] Nối cảm biến BME280 và màn hình OLED vào bus I2C (GPIO 21, 22).
* [ ] Viết chương trình quét địa chỉ I2C (`I2C Scanner`) để xác định đúng địa chỉ phần cứng `0x76` và `0x3C`.
* [ ] Hiển thị thông số nhiệt độ, độ ẩm và áp suất mượt mà lên màn hình OLED theo thời gian thực.

### Tuần 2: Lập trình ESP32 MQTT Client & Quản lý Trạng thái
* [ ] Cài đặt thư viện `PubSubClient` và `ArduinoJson` trên PlatformIO.
* [ ] Viết module quản lý kết nối Wi-Fi và MQTT: Tự động kết nối lại khi mất mạng (Exponential Backoff reconnect).
* [ ] Cấu hình gói tin Last Will (LWT) báo trạng thái sống còn của thiết bị.
* [ ] Đóng gói dữ liệu cảm biến thành chuỗi JSON và publish định kỳ 5 giây/lần lên topic `devices/ESP32_01/telemetry`.
* [ ] Subscribe topic `devices/ESP32_01/commands` để nhận lệnh điều khiển từ xa (ví dụ: đổi chu kỳ đọc, khởi động lại thiết bị).

### Tuần 3: Xây dựng Cơ chế Store-and-Forward Bền vững
* [ ] Định dạng phân vùng bộ nhớ Flash: Cấu hình `LittleFS` trong PlatformIO.
* [ ] Viết module `OfflineQueue`: Cung cấp 3 hàm cốt lõi: `pushMessage()`, `popMessage()`, `getQueueSize()`.
* [ ] Thử nghiệm thực tế:
  * Rút dây mạng của Router Wi-Fi trong 3 phút.
  * Quan sát màn hình OLED hiển thị số lượng gói tin đang xếp hàng trong bộ nhớ Flash (`Buffered: 36`).
  * Cắm lại dây mạng Wi-Fi: ESP32 kết nối lại Broker và tự động gửi vét toàn bộ 36 gói tin cũ lên topic `devices/ESP32_01/offline_sync` giữ nguyên mốc thời gian gốc.

### Tuần 4: Trực quan hóa Dashboard & Nâng cấp Phần mềm từ xa (OTA)
* [ ] Thêm service **Node-RED** và **Grafana** vào file `docker-compose.yml`.
* [ ] Thiết kế bảng điều khiển trực quan: Biểu đồ đường đo nhiệt độ/độ ẩm, đèn báo trạng thái thiết bị Online/Offline thời gian thực.
* [ ] Tích hợp tính năng **ArduinoOTA**: Cho phép nạp code firmware mới cho ESP32 qua mạng Wi-Fi nội bộ mà không cần cắm cáp USB.
* [ ] Quay video thử nghiệm tính năng rút mạng vẫn lưu trữ dữ liệu, hoàn thiện file `README.md`.
