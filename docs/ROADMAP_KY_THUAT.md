# 📘 SỔ TAY KỸ THUẬT & LỘ TRÌNH THỰC HIỆN CHI TIẾT DỰ ÁN 2
## Đề tài: Trạm giám sát thiết bị công nghiệp qua MQTT & Store-and-Forward (ESP32 + Mosquitto + Docker)

> **Tác giả:** Kỹ sư IoT & Hệ thống nhúng (NguyenHoangUy1305)  
> **Repository:** [NguyenHoangUy1305/esp32-mqtt-device-monitoring](https://github.com/NguyenHoangUy1305/esp32-mqtt-device-monitoring)  
> **Thời gian:** 6 tuần (Tháng 02/2027 - Tháng 04/2027, bắt đầu sau Tết Nguyên Đán: 15/02/2027)  
> **Mục tiêu:** Cung cấp lộ trình triển khai chi tiết từng tuần, các bẫy kỹ thuật thực chiến trong môi trường công nghiệp, thiết kế kiến trúc độ tin cậy cao (Store-and-Forward) và nạp nâng cấp phần mềm từ xa FOTA.

---

## MỤC LỤC
1. [PHẦN 1: TỔNG QUAN LỘ TRÌNH & ĐẶC TẢ CÔNG NGHIỆP](#phần-1-tổng-quan-lộ-trình--đặc-tả-công-nghiệp)
2. [PHẦN 2: LỘ TRÌNH CHI TIẾT 6 TUẦN TRIỂN KHAI (15/02/2027 - 05/04/2027)](#phần-2-lộ-trình-chi-tiết-6-tuần-triển-khai)
   - Tuần 1: Thiết lập hạ tầng Docker Mosquitto Broker, MQTTS & ACL (15/02 - 21/02/2027)
   - Tuần 2: Nối dây I2C, Driver Cảm biến BME280 & Màn hình OLED (22/02 - 28/02/2027)
   - Tuần 3: Xây dựng MQTT Client ESP32 với QoS 1 & Last Will LWT (01/03 - 07/03/2027)
   - Tuần 4: Thiết kế cơ chế lưu đệm Store-and-Forward trên Flash LittleFS (08/03 - 14/03/2027)
   - Tuần 5: Thuật toán Drainer tự động, Rate Limiting & Dashboard Grafana (15/03 - 21/03/2027)
   - Tuần 6: Nâng cấp Firmware từ xa FOTA Dual Partition & Nghiệm thu (22/03 - 05/04/2027)
3. [PHẦN 3: BẢNG BẪY KỸ THUẬT CÔNG NGHIỆP & CÁCH PHÒNG TRÁNH](#phần-3-bảng-bẫy-kỹ-thuật-công-nghiệp--cách-phòng-tránh)
4. [PHẦN 4: BẢNG ĐẶC TẢ DANH MỤC TOPIC MQTT CHUẨN DOANH NGHIỆP](#phần-4-bảng-đặc-tả-danh-mục-topic-mqtt-chuẩn-doanh-nghiệp)
5. [PHẦN 5: BỘ CÂU HỎI PHỎNG VẤN KỸ THUẬT DÀNH CHO DỰ ÁN 2](#phần-5-bộ-câu-hỏi-phỏng-vấn-kỹ-thuật)

---

# PHẦN 1: TỔNG QUAN LỘ TRÌNH & ĐẶC TẢ CÔNG NGHIỆP

Dự án này tập trung vào bài toán cốt lõi của Internet vạn vật công nghiệp (IIoT): **"Dữ liệu không bao giờ được phép mất mát, kể cả khi hệ thống mạng vô tuyến bị sập nhiều giờ"**.

* **Bộ thông số công nghiệp:** Đo lường đồng thời 3 thông số môi trường nhà máy: Nhiệt độ ($0 \sim 65^\circ\text{C}$), Độ ẩm ($20 \sim 90\%\text{ RH}$), Áp suất khí quyển ($300 \sim 1100\text{ hPa}$).
* **Tiêu chuẩn giao thức:** MQTT 3.1.1 / MQTT 5.0 chạy trên nền TLS/SSL; mức đảm bảo dịch vụ QoS 1 kèm cơ chế xử lý bản ghi trùng lặp (Idempotency Key).
* **Khả năng chịu lỗi:** Lưu đệm tới 50.000 bản ghi đo đạc trên bộ nhớ Flash nội vi; tự động xả đệm bù khi mạng phục hồi mà không làm nghẽn máy chủ.
* **Bảo trì không chạm:** Nâng cấp phần mềm từ xa không dây qua sóng Wi-Fi/4G (FOTA) với tính năng hoàn tác tự động chống biến thiết bị thành cục gạch (Anti-Bricking Protection).

---

# PHẦN 2: LỘ TRÌNH CHI TIẾT 6 TUẦN TRIỂN KHAI

### Tuần 1: Thiết lập hạ tầng Docker Mosquitto Broker, MQTTS & ACL (15/02 - 21/02/2027)
* **Mục tiêu:** Xây dựng cụm máy chủ MQTT Broker cấp công nghiệp sử dụng Docker Compose, cấu hình cơ chế bảo mật và danh sách kiểm soát quyền truy cập (ACL).
* **Công việc chi tiết:**
  * Viết file `docker-compose.yml` khởi chạy Eclipse Mosquitto Broker trên cổng tiêu chuẩn `1883` và cổng mã hóa SSL `8883`.
  * Cấu hình file `mosquitto.conf`: Vô hiệu hóa truy cập ẩn danh (`allow_anonymous false`), tạo file mật khẩu mã hóa `password_file`.
  * Thiết lập Access Control List (`acl_file`): Giới hạn mỗi trạm ESP32 chỉ được phép Publish vào topic của chính nó (`factory/station_+/telemetry`).
  * Khởi tạo container InfluxDB (cơ sở dữ liệu chuỗi thời gian) và Grafana (bảng điều khiển đồ thị) trong cùng mạng Docker Bridge.
  * Sử dụng công cụ MQTT Explorer trên máy tính để kiểm tra gửi nhận dữ liệu và xác thực mật khẩu.
* **Nghiệm thu:** Cụm Docker khởi chạy ổn định với 1 lệnh `docker-compose up -d`; MQTT Explorer kết nối thành công với user/pass.

---

### Tuần 2: Nối dây I2C, Driver Cảm biến BME280 & Màn hình OLED (22/02 - 28/02/2027)
* **Mục tiêu:** Giao tiếp hoàn hảo với 2 thiết bị cùng mắc song song trên bus $I^2C$ của ESP32; giải mã các thanh ghi hiệu chuẩn xuất xưởng của Bosch.
* **Công việc chi tiết:**
  * Đấu nối bus $I^2C$: Cảm biến BME280 và OLED SSD1306 chung 2 chân GPIO 21 (`SDA`) và GPIO 22 (`SCL`). Mắc 2 điện trở kéo lên nguồn $3.3\text{V}$ ($4.7\text{ k}\Omega$).
  * Chạy chương trình `I2C_Scanner`: Xác minh phát hiện đúng 2 địa chỉ phần cứng `0x76` (BME280) và `0x3C` (OLED).
  * Viết driver đọc 24 thanh ghi hiệu chuẩn `calib_data` từ ROM của BME280.
  * Cài đặt các công thức số học 32-bit bù trừ nhiệt độ và áp suất theo tài liệu kỹ thuật chính thức của Bosch Sensortec.
  * Viết module hiển thị giao diện đồ họa trên màn hình OLED: Hiển thị thanh tiến trình nhiệt độ, biểu tượng độ ẩm và trạng thái kết nối mạng.
  * Kiểm tra trôi điểm 0 (Zero Drift) và độ ổn định của số liệu đo đạc liên tục trong 12 tiếng.
* **Nghiệm thu:** Màn hình OLED hiển thị rõ nét các thông số thực tế; số liệu đo nhiệt độ độ ẩm khớp chuẩn với thiết bị đo thương mại.

---

### Tuần 3: Xây dựng MQTT Client ESP32 với QoS 1 & Last Will LWT (01/03 - 07/03/2027)
* **Mục tiêu:** Lập trình ứng dụng truyền thông MQTT trên vi điều khiển với độ tin cậy cao, hỗ trợ chất lượng dịch vụ QoS 1 và cơ chế Di chúc số.
* **Công việc chi tiết:**
  * Sử dụng thư viện `PubSubClient` (hoặc `async-mqtt-client` hỗ trợ non-blocking); cấu hình kích thước bộ đệm nhận gói tin `MQTT_MAX_PACKET_SIZE = 512`.
  * Lập trình cấu trúc dữ liệu JSON đo đạc: `{ stationId, temp, hum, press, uptime, wifiRssi, timestamp }`.
  * Cấu hình bản Di chúc số (Last Will and Testament):
    - Topic: `factory/station_01/status`
    - Payload: `{"status": "OFFLINE", "reason": "abnormal_power_loss"}`
    - QoS: 1, Retain: `true`.
  * Khi khởi động kết nối thành công: Gửi bản tin trạng thái `{"status": "ONLINE"}` với cờ `Retain = true`.
  * Thiết lập chu kỳ nhịp tim Keep-Alive $30\text{ giây}$: Tự động gửi gói tin `PINGREQ` khi đường truyền rảnh.
  * Thử nghiệm rút nguồn đột ngột của ESP32: Kiểm tra trên MQTT Explorer xem Broker có tự động phát tin LWT sau 45 giây hay không.
* **Nghiệm thu:** Tin nhắn Telemetry được gửi đều đặn mỗi 5 giây với QoS 1; broker phát tin báo tử LWT chính xác khi trạm gặp sự cố mất nguồn.

---

### Tuần 4: Thiết kế cơ chế lưu đệm Store-and-Forward trên Flash LittleFS (08/03 - 14/03/2027)
* **Mục tiêu:** Giải quyết bài toán bảo toàn dữ liệu khi mạng vô tuyến bị tê liệt bằng cấu trúc hàng đợi vòng tròn Circular Ring Buffer trên bộ nhớ Flash nội.
* **Công việc chi tiết:**
  * Cấu hình phân vùng bộ nhớ Flash dành riêng $512\text{ KB}$ cho hệ thống tệp tin `LittleFS`.
  * Thiết kế cấu trúc bản ghi nhị phân nhúng (Compact Binary Struct) hoặc chuỗi JSON nén để tối ưu hóa dung lượng lưu trữ trên Flash.
  * Xây dựng máy trạng thái kết nối mạng: Khi `mqttClient.connected() == false`, chuyển ngay sang chế độ lưu đệm `BUFFERING_MODE`.
  * Lập trình cơ chế ghi tuần tự vào Flash kèm dấu thời gian thực lấy từ chip RTC ngoại hoặc thời gian SNTP đã lưu trước đó.
  * Ứng dụng kỹ thuật xoay vòng tệp tin (File Rotation): Tạo các tệm đệm kích thước $64\text{ KB}$, khi đầy tự động mở tệp mới; nếu bộ nhớ Flash chạm ngưỡng $90\%$, tự động xóa các bản ghi cũ nhất (FIFO Drops).
  * Kiểm tra tốc độ ghi Flash: Đảm bảo thời gian ghi 1 bản ghi vào LittleFS mất dưới $15\text{ ms}$, không gây giật lag luồng đo cảm biến.
* **Nghiệm thu:** Cắt sóng Wi-Fi trong 30 phút; kiểm tra thấy hàng trăm bản ghi đo đạc được lưu trữ nguyên vẹn trong bộ nhớ Flash mà không bị mất gói nào.

---

### Tuần 5: Thuật toán Drainer tự động, Rate Limiting & Dashboard Grafana (15/03 - 21/03/2027)
* **Mục tiêu:** Tự động xả bù dữ liệu từ Flash lên Broker khi mạng hồi phục; trực quan hóa toàn bộ hệ thống trên bảng điều khiển Grafana chuyên nghiệp.
* **Công việc chi tiết:**
  * Xây dựng tiến trình nền `BufferDrainerTask`: Khi kết nối Wi-Fi và MQTT phục hồi, tự động kích hoạt đọc các bản ghi từ Flash theo thứ tự FIFO (First In, First Out).
  * Thêm cờ đánh dấu vào payload gửi bù: `{ ...telemetry, is_buffered: true, original_timestamp: 1711234500 }`.
  * Cơ chế điều tiết tốc độ (Rate Limiting): Gửi từng lô 10 bản ghi, chờ phản hồi `PUBACK` từ Broker mới xóa khỏi Flash; nghỉ $50\text{ ms}$ giữa các lô để không gây nghẽn băng thông.
  * Cấu hình Telegraf subscribe topic MQTT và tự động phân tách dữ liệu ghi vào InfluxDB dựa trên `original_timestamp` (không dùng thời gian nhận của Broker).
  * Thiết kế Dashboard Grafana: Biểu đồ đường nhiệt độ/áp suất theo thời gian thực, bảng gauge đo độ ẩm, đèn báo trạng thái trạm Online/Offline.
  * Cấu hình cảnh báo Telegram Bot từ Grafana: Tự động gửi tin nhắn báo động tới điện thoại khi nhiệt độ lò vượt quá $45^\circ\text{C}$.
* **Nghiệm thu:** Dữ liệu sau khi xả đệm được điền khuyết chính xác vào trục thời gian quá khứ trên biểu đồ Grafana; đồ thị không hề bị đứt đoạn sau sự cố mất mạng.

---

### Tuần 6: Nâng cấp Firmware từ xa FOTA Dual Partition & Nghiệm thu (22/03 - 05/04/2027)
* **Mục tiêu:** Triển khai tính năng nạp phần mềm từ xa không dây (Firmware Over-The-Air - FOTA) với cơ chế Rollback tự động chống "biến thiết bị thành cục gạch" (Anti-Bricking).
* **Công việc chi tiết:**
  * Cấu hình bảng phân vùng bộ nhớ Flash `custom_partitions.csv`: Tạo 2 phân vùng ứng dụng đối xứng `ota_0` ($1.5\text{ MB}$) và `ota_1` ($1.5\text{ MB}$).
  * Xây dựng luồng nhận lệnh FOTA qua topic MQTT: `factory/station_01/ota/command`.
  * Payload lệnh chứa đường dẫn tải bản cập nhật: `{ "version": "2.1.0", "url": "http://192.168.1.100:8080/firmware_v2.bin", "sha256": "..." }`.
  * Sử dụng thư viện `Update.h` của ESP32: Tải luồng nhị phân từ HTTP server, ghi tuần tự vào phân vùng OTA đối diện, kiểm tra tính toàn vẹn chữ ký SHA-256.
  * Lập trình tính năng tự phục hồi an toàn (Rollback Mechanism):
    - Sau khi nạp firmware mới và khởi động lại, chương trình có 30 giây để xác nhận hệ thống chạy ổn định và kết nối được MQTT bằng lệnh `esp_ota_mark_app_valid_cancel_rollback()`.
    - Nếu trong vòng 30 giây chương trình bị crash loop hoặc không kết nối được mạng: Bootloader tự động hoàn tác (Rollback) trở lại phiên bản firmware cũ ngay lập tức!
  * Quay video thử nghiệm nạp firmware từ xa thành công và mô phỏng hoàn tác Rollback an toàn khi nạp file firmware lỗi. Nghiệm thu hoàn tất dự án.
* **Nghiệm thu:** Nạp firmware mới qua Wi-Fi trong 45 giây; thiết bị tự khởi động lại và chạy bản mới mượt mà; tính năng Rollback hoạt động hoàn hảo khi giả lập crash.

---

# PHẦN 3: BẢNG BẪY KỸ THUẬT CÔNG NGHIỆP & CÁCH PHÒNG TRÁNH

| Bẫy kỹ thuật thực tế | Hậu quả nghiêm trọng | Nguyên nhân kỹ thuật | Giải pháp chuẩn kỹ sư công nghiệp |
| :--- | :--- | :--- | :--- |
| **Bẫy treo bus I2C (Bus Lockup)** | Toàn bộ hệ thống bị đơ cứng, không đọc được BME280 lẫn OLED | Một thiết bị Slave bị xung nhiễu kéo giữ chân `SDA` ở mức LOW vô thời hạn, khiến Master không thể tạo xung START. | **Thực hiện chu trình I2C Bus Clear:** Trước khi gọi `Wire.begin()`, phát 9 xung Clock liên tiếp trên chân SCL để ép Slave nhả đường SDA. |
| **Bẫy mòn bộ nhớ Flash (Wear-out)** | Chip ESP32 bị hỏng Flash sau 2-3 tháng vận hành liên tục | Bộ nhớ Flash NOR chỉ có tuổi thọ khoảng $10.000 \sim 100.000$ chu kỳ ghi xóa cho mỗi sector $4\text{ KB}$. | **Sử dụng hệ thống tệp tin LittleFS** (tích hợp sẵn thuật toán Dynamic Wear Leveling trải đều việc ghi lên toàn bộ Flash) thay cho SPIFFS cũ. |
| **Bẫy lũ lụt dữ liệu (Buffer Flooding)** | Broker bị quá tải, rớt kết nối hàng loạt khi nhiều trạm xả đệm cùng lúc | Sau sự cố mất điện nhà máy, hàng chục trạm cùng có mạng và đồng loạt xả hàng ngàn gói tin lên Broker ở tốc độ tối đa. | **Áp dụng giải thuật Trì hoãn ngẫu nhiên (Jitter Delay)** và kiểm soát tốc độ (Rate Limiting: tối đa 20 gói/giây cho mỗi trạm). |
| **Bẫy kết nối "chết lâm sàng" (Half-Open Socket)** | ESP32 tưởng vẫn đang kết nối nhưng Broker đã coi thiết bị là offline | Đường truyền Wi-Fi bị đứt ngầm nhưng socket TCP không nhận được cờ ngắt FIN/RST. | **Thiết lập chu kỳ Keep-Alive ngắn ($30\text{s}$)**. Nếu quá 2 chu kỳ không có PINGRESP, chủ động đóng socket và khởi tạo lại kết nối. |
| **Bẫy biến thành cục gạch khi OTA (Bricking)** | Thiết bị mất liên lạc vĩnh viễn, bắt buộc phải tháo vỏ cắm dây nạp lại | Firmware mới tải về bị lỗi logic khiến CPU bị crash liên tục ngay trong hàm `setup()`. | **Cấu hình phân vùng Dual OTA Partition với cờ Rollback tự động** của ESP-IDF / Arduino ESP32. |

---

# PHẦN 4: BẢNG ĐẶC TẢ DANH MỤC TOPIC MQTT CHUẨN DOANH NGHIỆP

```text
+-------------------------------------------+-----+--------+----------------------------------------------------+
| Tên Topic (Topic Pattern)                 | QoS | Retain | Mô tả chức năng & Dữ liệu Payload                  |
+-------------------------------------------+-----+--------+----------------------------------------------------+
| factory/station_01/status                 |  1  |  true  | Trạng thái trạm: {"status":"ONLINE"|"OFFLINE"} (LWT|
| factory/station_01/telemetry              |  1  | false  | Số liệu đo đạc định kỳ: {temp, hum, press, uptime} |
| factory/station_01/telemetry/buffered     |  1  | false  | Dữ liệu xả đệm: {..., is_buffered:true, original_ts|
| factory/station_01/alerts                 |  1  | false  | Cảnh báo sự cố: {"type":"OVERHEAT", "val": 48.5}   |
| factory/station_01/cmd/relay              |  1  | false  | Lệnh điều khiển quạt/còi: {"state": "ON"|"OFF"}    |
| factory/station_01/ota/command            |  2  | false  | Lệnh nạp phần mềm từ xa: {version, url, sha256}    |
| factory/station_01/ota/progress           |  0  | false  | Tiến độ nạp firmware: {"percent": 45, "status":...}|
+-------------------------------------------+-----+--------+----------------------------------------------------+
```

---

# PHẦN 5: BỘ CÂU HỎI PHỎNG VẤN KỸ THUẬT DÀNH CHO DỰ ÁN 2

1. **Câu hỏi:** *Hãy phân tích sự khác nhau giữa QoS 0, QoS 1 và QoS 2 trong MQTT. Tại sao bạn lại chọn QoS 1 cho dữ liệu cảm biến công nghiệp?*  
   **Trả lời:** QoS 0 là "bắn rồi quên", không đảm bảo gói tin tới đích. QoS 2 đảm bảo đúng 1 lần duy nhất bằng bắt tay 4 bước (`PUBLISH`, `PUBREC`, `PUBREL`, `PUBCOMP`), nhưng tiêu tốn RAM gấp 3 lần và độ trễ rất cao trong mạng vô tuyến. Em chọn QoS 1 vì nó đảm bảo dữ liệu "ít nhất 1 lần" thông qua gói phản hồi `PUBACK`. Để xử lý nhược điểm gói tin có thể bị gửi trùng khi mạng chập chờn, em bổ sung mã định danh duy nhất (Idempotency Key / Timestamp) vào payload để phía Subscriber tự động loại bỏ các bản ghi trùng lặp một cách nhẹ nhàng.

2. **Câu hỏi:** *Cơ chế Last Will and Testament (LWT) hoạt động như thế nào dưới tầng giao vận TCP?*  
   **Trả lời:** Khi ESP32 thiết lập kết nối `CONNECT` tới Broker, nó gửi kèm một bản "di chúc" đăng ký trước gồm Topic, Payload, QoS và Retain Flag. Broker sẽ lưu trữ bản di chúc này trong bộ nhớ phiên của Client. Nếu Client ngắt kết nối an toàn bằng lệnh `DISCONNECT`, Broker sẽ xóa di chúc. Nhưng nếu kết nối TCP bị đứt đột ngột (mất nguồn, đứt cáp) mà không có gói tin FIN/RST, sau khoảng thời gian $1.5 \times \text{KeepAlive}$, Broker sẽ phát hiện mất nhịp tim PING, chủ động đóng socket và ngay lập tức phát tán bản di chúc LWT tới tất cả các subscriber đang theo dõi, giúp hệ thống phát hiện trạm gặp sự cố ngay lập tức.

3. **Câu hỏi:** *Bộ nhớ Flash của vi điều khiển rất dễ bị hỏng nếu ghi xóa liên tục. Bạn giải quyết vấn đề tuổi thọ Flash như thế nào trong cơ chế Store-and-Forward?*  
   **Trả lời:** Em sử dụng hệ thống tệp tin LittleFS thay vì SPIFFS. LittleFS tích hợp sẵn giải thuật cân bằng độ mòn động (Dynamic Wear Leveling), tự động phân bổ các chu kỳ ghi tuần tự trải đều lên các sector $4\text{ KB}$ khác nhau của phân vùng Flash $512\text{ KB}$, ngăn chặn việc ghi đè liên tục vào một vị trí vật lý cố định. Ngoài ra, em chỉ kích hoạt ghi Flash khi phát hiện trạng thái mất kết nối mạng (`mqtt.connected() == false`), còn khi mạng bình thường thì dữ liệu chỉ truyền thẳng trong RAM mà không chạm vào Flash.

4. **Câu hỏi:** *Trình bày quy trình nạp Firmware Over-The-Air (FOTA) an toàn và cơ chế Rollback chống biến thiết bị thành cục gạch (Anti-Bricking).*  
   **Trả lời:** Em chia Flash ESP32 thành 2 phân vùng ứng dụng đối xứng là `ota_0` và `ota_1`. Nếu vi điều khiển đang chạy trên `ota_0`, firmware mới tải về từ HTTP server sẽ được ghi tuần tự vào `ota_1`. Sau khi tải xong, ESP32 kiểm tra tính toàn vẹn mã băm SHA-256. Nếu hợp lệ, nó ghi cờ vào phân vùng `otadata` thông báo cho Bootloader chuyển quyền khởi động sang `ota_1`. Khi boot vào firmware mới, nếu trong vòng $30\text{ giây}$ mà chương trình bị crash loop hoặc không kết nối được MQTT, hàm `esp_ota_mark_app_valid_cancel_rollback()` sẽ không được gọi, Bootloader phần cứng sẽ tự động hoàn tác quyền khởi động trở lại phân vùng `ota_0` cũ an toàn.
