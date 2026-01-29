Real-Time Smart Door Lock System (ESP32 + Fingerprint + MQTT)
1. Giới thiệu
Dự án này hiện thực một hệ thống khóa cửa thông minh theo hướng real-time end-to-end, sử dụng ESP32, cảm biến vân tay, và MQTT (HiveMQ Cloud).
Khác với các hệ IoT thông thường, project tập trung vào tính dự đoán được về thời gian (predictable timing), bao gồm:
- Deadline rõ ràng cho hành động mở cửa
- Đo đạc end-to-end latency (microsecond-level)
- Phân tích p95 / p99 latency, jitter, deadline miss rate
- So sánh Baseline (blocking) và Improved (RMS-like scheduling)
Dự án được thực hiện trong khuôn khổ môn RTS20242 – Real-Time Systems.
________________________________________
2. Tính năng chính
- 🔐 Mở cửa bằng vân tay (local, real-time critical)
- 🌐 Mở cửa từ xa qua MQTT command
- ⏱️ Đo end-to-end latency (e2e_us) và actuator time (act_us)
- 📊 So sánh baseline vs improved scheduling
- 🚦 Cơ chế drop/skip khi overload (cooldown)
- 🧪 Hỗ trợ stress test (MQTT burst)
________________________________________
3. Yêu cầu phần cứng
- ESP32 Dev Module
- Cảm biến vân tay (AS608 / R305 / tương đương)
- LCD I2C 16x2 (địa chỉ 0x27)
- Keypad 4x4
- Relay module (GPIO 23)
- Nguồn 12V 
________________________________________
4. Yêu cầu phần mềm
- Arduino IDE ≥ 1.8.x
- ESP32 Board Package (Espressif)
- Install các thư viện trong arduino IDE:
- WiFi.h
- WiFiClientSecure.h
- PubSubClient
- LiquidCrystal_I2C
- Keypad
- Adafruit_Fingerprint
________________________________________
6. Cấu hình MQTT
- Broker: HiveMQ Cloud
- Protocol: MQTT over TLS (port 8883)
- Subscribe topic: home/door2/cmd
- Publish topic: home/door2/status
- Command payload:
- Open
________________________________________
7. Cách build & chạy (Reproducibility)
- 7.1 Nạp firmware
- Mở Arduino IDE
- Chọn board: ESP32 Dev Module
- Mở file .ino trong thư mục firmware/
- Cấu hình WiFi & MQTT trong code
- Compile và upload lên ESP32
- 7.2 Chạy hệ thống
- Sau khi nạp:
- Hệ thống tự kết nối WiFi & MQTT
- Có thể mở cửa bằng:
- Vân tay
- MQTT command "Open"
________________________________________
8. Thu thập log & đo KPI timing
- 8.1 Log format
- Mỗi lần mở cửa, hệ thống publish log dạng:
UnlockedByMQTT;e2e_us=253443;act_us=99842
UnlockedByFP;id=2;e2e_us=4;act_us=99756
- e2e_us: end-to-end latency (µs)
- act_us: thời gian actuator/relay (µs)
- 8.2 Thu log
- Subscribe topic: home/door2/status
- Lưu log ra file text, ví dụ:
- logs/improved_status.txt
________________________________________
9. Tính KPI (p95 / p99 / jitter / miss rate)
- Max latency
- p50 / p95 / p99
- Jitter (max − min)
- Deadline miss rate
________________________________________
10. Thí nghiệm & stress test
MQTT burst
Gửi nhiều lệnh "Open" liên tục để tạo overload:
- Quan sát latency tăng
- Quan sát log CmdDropped:Cooldown
- Kiểm tra deadline miss rate
Luồng Fingerprint vẫn giữ latency thấp và ổn định → chứng minh ưu tiên real-time.

