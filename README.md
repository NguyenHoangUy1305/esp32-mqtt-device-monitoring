# 📡 TRẠM GIÁM SÁT THIẾT BỊ CÔNG NGHIỆP QUA MQTT & STORE-AND-FORWARD

[![CI](https://github.com/NguyenHoangUy1305/esp32-mqtt-device-monitoring/actions/workflows/ci.yml/badge.svg)](https://github.com/NguyenHoangUy1305/esp32-mqtt-device-monitoring/actions/workflows/ci.yml)

> **Tên đề tài:** Industrial IoT Device Monitoring Agent with High-Reliability Store-and-Forward Telemetry and Remote OTA  
> **Thời gian:** Tháng 03/2027 - Tháng 04/2027 (4 tuần)  
> **Mục tiêu:** Nắm vững giao thức MQTT chuẩn công nghiệp, thiết kế kiến trúc truyền dữ liệu tin cậy (High Availability) khi kết nối mạng chập chờn.

---

> 📘 **SỔ TAY KỸ THUẬT & LỘ TRÌNH 4 TUẦN CHI TIẾT:** Xem toàn bộ lý thuyết MQTT QoS, thuật toán Store-and-Forward trên Flash và OTA tại [`docs/ROADMAP_KY_THUAT.md`](./docs/ROADMAP_KY_THUAT.md)


## 1. CẤU TRÚC THƯ MỤC DỰ ÁN
```text
02-mqtt-device-monitoring/
├── firmware/       # Mã nguồn C++ ESP32 sử dụng PubSubClient & LittleFS
│   ├── src/        # MQTT manager, Sensor reader, Offline queue, OTA updater
│   └── include/    # Cấu hình MQTT Broker, Topic list
├── docker/         # Môi trường chạy Broker & Giám sát tập trung
│   ├── docker-compose.yml  # Mosquitto Broker + Node-RED + Grafana
│   └── mosquitto/  # File cấu hình mosquitto.conf, mật khẩu & ACL
├── docs/           # Sơ đồ giải thuật Store-and-Forward, sơ đồ giao thức
└── README.md       # Tài liệu đặc tả dự án
```

---

## 2. PHẦN CỨNG & CẢM BIẾN SỬ DỤNG
* **ESP32 DevKit V1**: Vi điều khiển kết nối Wi-Fi.
* **Cảm biến BME280 (I2C)**: Đo nhiệt độ, độ ẩm và áp suất khí quyển độ chính xác cao.
* **Màn hình OLED 0.96 inch (SSD1306 - I2C)**: Hiển thị địa chỉ IP, trạng thái MQTT, chỉ số cảm biến.
* **Bus I2C kết nối chung chân:**
  * `SDA` -> **GPIO 21**
  * `SCL` -> **GPIO 22**

---

## 3. ĐẶC TẢ KIẾN TRÚC MQTT & DANH MỤC TOPIC

### 3.1. Các thông số kết nối
* **Broker:** Eclipse Mosquitto (triển khai trên Docker / Local PC port 1883).
* **QoS Level:** QoS 1 (At least once) cho dữ liệu quan trọng, QoS 0 cho telemetry định kỳ.
* **Last Will and Testament (LWT):** Tự động phát thông điệp thiết bị offline nếu ESP32 mất nguồn hoặc đứt mạng đột ngột.

### 3.2. Cấu trúc Topic
| Chiều giao tiếp | Tên Topic | Nội dung Payload (JSON) | Mục đích |
| :--- | :--- | :--- | :--- |
| ESP32 -> Broker | `devices/ESP32_01/telemetry` | `{"temp": 28.5, "hum": 65.2, "press": 1013, "ts": 1711234567}` | Dữ liệu định kỳ 5s |
| ESP32 -> Broker | `devices/ESP32_01/status` | `{"status": "online", "ip": "192.168.1.50"}` (Retained) | Trạng thái sống còn |
| ESP32 -> Broker | `devices/ESP32_01/status` | `{"status": "offline", "reason": "unexpected"}` (LWT) | Báo mất kết nối tức thì |
| Broker -> ESP32 | `devices/ESP32_01/commands` | `{"command": "REBOOT"}` hoặc `{"command": "SET_SAMPLE_RATE", "val": 2}` | Điều khiển từ xa |

---

## 4. TÍNH NĂNG ĐỘC ĐÁO: STORE-AND-FORWARD BỀN VỮNG
* Khi mất mạng Wi-Fi hoặc MQTT Broker không phản hồi:
  1. ESP32 tự động chuyển sang chế độ **BUFFERING**.
  2. Gói tin JSON được ghi nối tiếp vào file hàng đợi trên bộ nhớ Flash vi điều khiển (`LittleFS`).
  3. Áp dụng cơ chế **Circular Buffer** (giới hạn tối đa 500 bản ghi, tự ghi đè bản ghi cũ nhất khi Flash đầy để tránh tràn bộ nhớ).
* Khi mạng khôi phục:
  1. ESP32 kết nối lại Wi-Fi và Broker.
  2. Đọc từng bản ghi trong Flash, publish bù lên topic `devices/ESP32_01/offline_sync` giữ nguyên mốc thời gian gốc.
  3. Xóa dữ liệu tạm sau khi Broker xác nhận đã nhận (`PUBACK`).

---

## 5. NÂNG CẤP FIRMWARE TỪ XA (OVER-THE-AIR - OTA)
* Tích hợp tính năng ArduinoOTA / Web OTA cho phép cập nhật code mới cho ESP32 qua mạng nội bộ mà không cần cắm cáp nạp.
