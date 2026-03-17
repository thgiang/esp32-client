# Hướng dẫn Build cho ESP32 S3 N16R8

Tài liệu này tóm tắt cách thiết lập môi trường và thực hiện lệnh build cho module ESP32 S3 N16R8 (16MB Flash, 8MB PSRAM), trên MAC và Linux có hỗ trợ nhưng Việt Nam mình chủ yếu dùng Windows nên mình viết hướng dẫn trên Windows. 

## 1. Yêu cầu tiên quyết

*   Cài đặt **ESP-IDF v5.4** trở lên. Link tải: https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/index.html
*   Tải flash_download_tool_3.9.7.zip. Link tải: https://dl.espressif.com/public/flash_download_tool.zip
*   Sau khi cài đặt xong bạn tìm đến thư mục C:\Espressif\frameworks\esp-idf-v5.5.3-3 chạy các file `export.bat`/`export.sh`/`export.ps1` lần nữa cho chắc
*   Ấn vào Windows gõ cmd bây giờ sẽ xuất hiện 1 loại CMD riêng của ESP-IDF tên là ESP-IDF 5.5 CMD, từ bây giờ dùng command line này để chạy các lệnh liệt kê bên dưới
 
## 2. Các lệnh Build cơ bản. Nhớ đọc tới phần 3. kẻo tự cung quá sớm

Dự án này sử dụng cấu hình mặc định trong `sdkconfig.defaults.esp32s3` hỗ trợ sẵn cho N16R8. Bạn nào build cho loại khác nhớ tìm hiểu đúng loại và thay đổi tên trong câu lệnh bên dưới.

### CD vào đúng thư mục
Ví dụ mình đang có source code ở C:\xampp\htdocs\xiaozhi-esp32 thì mình gõ
```bash
cd C:\xampp\htdocs\xiaozhi-esp32
```

### Cấu hình Target
```bash
idf.py set-target esp32s3
```

### Build Firmware
```bash
idf.py build
```

### Nạp Firmware và Theo dõi Serial
```bash
idf.py flash monitor
```

## 3. Build theo Board cụ thể (Khuyên dùng)

Nếu bạn muốn build cho một loại board cụ thể đã được định nghĩa trong thư mục `main/boards` (ví dụ `bread-compact-wifi`), hãy sử dụng script hỗ trợ:

```bash
python scripts/release.py bread-compact-wifi
```

Lệnh này sẽ tự động:
1. Thiết lập target `esp32s3`.
2. Áp dụng các cấu hình `sdkconfig` cần thiết cho board.
3. Build và tạo file zip chứa firmware hoàn chỉnh trong thư mục `releases/`.

## 4. Lưu ý về N16R8
Dành cho bạn nào cần hiểu chuyên sâu, thực tế chỉ cần build ra được file trong thư mục releases là được rồi
*   **Flash Size**: Dự án đã cấu hình 16MB (`CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y`).
*   **PSRAM**: Sử dụng chế độ Octal PSRAM 80MHz (`CONFIG_SPIRAM_MODE_OCT=y`).
*   **Phân vùng**: Sử dụng bảng phân vùng v2 (xem trong `partitions/v2/`).

## 5. Nạp firmware vừa build vào phần cứng của bạn
1. Sử dụng flash_download_tool_3.9.7 chọn đúng loại mạch (Ví dụ ESP32-S3), dòng bên dưới chọn UART
2. Giải nén file vừa build ra được file merged-binary.bin, sau đó chọn file này, ô bên phải ghi 0x0 rồi nhớ tích chọn nó ở đầu dòng
3. Chọn đúng cổng COM, baud rate 115200, sau đó nhấn Start và chờ tới khi hiện Finnish (khoảng 30s) là xong. Rút điện ra cắm lại là có firmware mới.


# Cấu trúc Source Code dự án Xiaozhi ESP32

Tài liệu này tổng hợp phân tích về cấu trúc mã nguồn, các công nghệ được sử dụng, luồng khởi động và cách thiết bị ESP32 giao tiếp với server. Nó được viết nhằm giúp người mới làm quen có thể nhanh chóng nắm bắt và tham gia vào việc phát triển.

---

## 1. Tổng quan Kiến trúc và Công nghệ (Technologies & Architecture)

Dự án này là firmware dành cho các bo mạch dựa trên chip ESP32 (đặc biệt là ESP32-S3, ESP32-P4, ESP32-C3/C6). Nó đóng vai trò là "client" âm thanh thông minh, giao tiếp với máy chủ AI để thực hiện chức năng trợ lý ảo giọng nói.

- **Framework**: **ESP-IDF** (Espressif IoT Development Framework) với C/C++ (C++17/20). Hệ thống build sử dụng CMake.
- **GUI & Hiển thị**: Sử dụng thư viện **LVGL** (Light and Versatile Graphics Library) cho các màn hình LCD/OLED. Hỗ trợ hiển thị Emoji, độ ẩm/nhiệt độ, thông báo trạng thái và hiệu ứng cảm xúc (emotions) của AI.
- **Xử lý Audio**: Ghi âm qua Mic, phát qua loa (I2S). Sử dụng bộ phân tích, giải mã audio (Codec: ES8311, ES8388, v.v.), nhận diện từ khóa đánh thức (Wake word - thông qua Espressif AFE hoặc module tùy chỉnh) và VAD (Voice Activity Detection). Mã hóa âm thanh khi truyền dùng chuẩn **Opus**.
- **Network & Giao thức**: Hỗ trợ 2 giao thức chính linh hoạt qua Wi-Fi hoặc 4G:
  - **WebSocket**: Truyền dữ liệu dạng Text (JSON) và Binary (Audio) trên cùng một kết nối kết hợp.
  - **MQTT + UDP**: Dùng MQTT để nhận/gửi tín hiệu điều khiển (JSON) và mở một kênh UDP được mã hóa AES (AES-CTR) để truyền nhận stream âm thanh Audio Opus độ trễ thấp.
- **OTA (Over-The-Air)**: Tính năng tự động cập nhật Firmware và Assets (như font chữ, âm thanh thông báo) trực tiếp qua mạng.

---

## 2. Cấu trúc Thư mục chính

Dưới đây là các thành phần quan trọng bên trong thư mục `main/`:

- `main.cc`: Điểm bắt đầu (Entry point) của chương trình ESP32 (`app_main`). Khởi tạo bộ nhớ NVS flash và chạy vòng lặp chính của lớp `Application`.
- `application.cc` / `application.h`: **Trái tim của hệ thống**. Quản lý State Machine (trạng thái chờ, kết nối, nghe, nói, nâng cấp...), khởi tạo phần cứng thông qua `Board`, vận hành dịch vụ Audio và kết nối mạng.
- `audio/`: Thư mục xử lý âm thanh.
  - `audio_service.cc`: Quản lý luồng âm thanh vào/ra, kích hoạt wake word, VAD, hàng đợi gửi gói tin (send queue) và giải mã nhận (decode queue).
  - `audio_codec.cc`, `codecs/`: Khởi tạo và giao tiếp với chip xử lý âm thanh (DAC/ADC) chuyên biệt cho từng loại board.
  - `processors/`, `wake_words/`: Các module bắt từ khóa đánh thức (Wake word engine).
- `boards/`: Chứa cấu hình và mã khởi tạo phần cứng cụ thể cho hàng chục loại board khác nhau (ESP-BOX, M5Stack, LilyGo, Waveshare, v.v.). Mỗi board có thiết lập PIN (chân cấu hình), màn hình, LED và Audio riêng.
- `display/`: Xử lý giao diện người dùng. Phân nhánh hỗ trợ cho màn hình LCD (`lcd_display.cc`), OLED (`oled_display.cc`) và giao diện LVGL chuẩn hóa cho việc hiển thị cảm xúc (`emote_display.cc`).
- `protocols/`: Triển khai các tiêu chuẩn kết nối máy chủ bao gồm:
  - `mqtt_protocol.cc` / `websocket_protocol.cc`.
- `ota.cc`: Xử lý nâng cấp Firmware và tải Assets (âm thanh, hình ảnh nhúng, font).
- `mcp_server.cc`: Quản lý các tool/system commands, ví dụ như khởi động lại thiết bị hoặc thông báo trạng thái pin.
- `CMakeLists.txt`: Quản lý kịch bản build và biên dịch các tệp nguồn tùy thuộc vào `CONFIG_BOARD_TYPE` người dùng chọn qua Kconfig.

---

## 3. Luồng khởi động màn hình và chương trình (Boot Flow)

1. **`app_main` (`main.cc`)**:
   - `nvs_flash_init()`: Khởi tạo phân vùng lưu trữ cấu hình mạng/Wi-Fi.
   - Gọi `Application::GetInstance().Initialize()`.
2. **Khởi tạo hệ thống (`Application::Initialize()`)**:
   - Khởi tạo **Board**: Gọi tới thiết lập màn hình, âm thanh tương ứng với board đang nạp.
   - Khởi tạo **Display**: Hiển thị GUI trạng thái "Starting...".
   - Khởi tạo **AudioService**: Bật các task nền chờ phát hiện luồng giọng nói (VAD, wake word).
   - Khởi tạo kết nối mạng (**Network**): Tiến hành Scan và kết nối lại cấu hình Wi-Fi/Sim.
3. **Loop chính (`Application::Run()`)**:
   - Chạy một vòng lặp sự kiện dùng FreeRTOS `xEventGroupWaitBits`.
   - Bắt các sự kiện như: `MAIN_EVENT_NETWORK_CONNECTED`, `MAIN_EVENT_WAKE_WORD_DETECTED`, `MAIN_EVENT_SEND_AUDIO`, `MAIN_EVENT_STATE_CHANGED` v.v. để quyết định hành động tiếp theo.
4. **Kích hoạt OTA/Protocol (`ActivationTask`)**:
   - Khi mạng kết nối, app sẽ check phiên bản phần mềm/assets. Sau đó khởi tạo `WebsocketProtocol` hoặc `MqttProtocol` kết nối lên máy chủ. Nếu chưa có Server cấp ghép, nó sẽ in ra mã Activation Code chờ nhập từ máy chủ quản lý.

---

## 4. Giao tiếp với Server (API & Protocols & Data Flow)

Hệ thống cung cấp một API bất đồng bộ theo kiến trúc **Mạng điều khiển + Mạng truyền phát Audio**. Toàn bộ mã Payload JSON xử lý qua thư viện `cJSON`.

### 4.1. Luồng Gói dữ liệu (JSON Format)
Dữ liệu chữ đều có format JSON, ví dụ `{"type": "<lệnh>", ...}`. Các module chính khi Server trả về ESP32 bao gồm:
- `"type": "tts"`: Ra lệnh bắt đầu nói (Text-To-Speech). Gồm `state: "start"`, `"sentence_start"`, `"stop"`. ESP32 sẽ hiển thị text (subtitles) của trợ lý (assistant) lên màn hình.
- `"type": "stt"`: Phản hồi nội dung chữ người dùng vừa nói (Speech-To-Text), ESP hiển thị lên màn hình.
- `"type": "llm"`: Server trả dải trạng thái cảm xúc (VD: `"emotion": "happy"`). Màn hình ESP cập nhật trạng thái khuôn mặt AI.
- `"type": "system"`: Nhận các lệnh hệ thống (VD: `"command": "reboot"`).

### 4.2. Luồng truyền Audio (Gửi / Nhận)
#### Giao thức phổ biến nhất (MQTT + UDP):
1. ESP32 gửi chuỗi JSON Hello qua MQTT (Yêu cầu kênh `udp`).
2. Server phản hồi bằng gói JSON chứa config cho UDP (`server`, `port`, `key` mã hóa AES, `nonce`, `sample_rate`).
3. ESP32 kết nối cổng UDP theo thông tin nhận được. 
4. Âm thanh nói (Opus codec, 16kHz, 1 channel) tiếp nhận từ Mic được gói vào UDP, **mã hóa qua AES-CTR 128 bit**, và bắn trực tiếp lên Máy chủ UDP kia.
5. Server phản hồi Audio TTS, UDP ESP32 nhận, giải mã AES, nạp vào Decode Queue và xuất ra loa ngoài.

#### Giao thức thay thế (WebSocket):
- Kết nối tới URL ws:// hoặc wss://.
- **Text JSON** được gửi và nhận bằng WebSocket Data Frame định dạng Text.
- **Audio Opus** được bọc thêm Payload Header (như `version`, `type`, `timestamp`, `size`) rồi gửi xuống socket theo dạng WebSocket Binary Data Frame.

> Việc duy trì kết nối giúp thiết bị lưu giữ session trơn tru và trò chuyện với độ trễ tối thiểu (<500ms).

---

## 5. Tóm lược dành cho Người mới (Dễ tiếp cận nhất)

1. Nếu bạn muốn thêm **phần cứng màn hình/audio mới**, hay tạo một bảng cấu hình mạch khác, hãy xem bên trong thư mục `main/boards/`.
2. Nếu bạn muốn **đổi giao diện màn hình**, chỉnh sửa màu sắc, phông chữ, biểu tượng cảm xúc, hãy làm việc trong `main/display/` và các file `lvgl`.
3. Nếu bạn muốn **can thiệp xử lý đoạn hội thoại, logic trạng thái** (Listening -> Speaking -> Idle), hãy tập trung ở file `main/application.cc` và các Handler.
4. Nếu bạn mốn xem cách **ẩn mã hóa, thông điệp JSON mạng**, hãy xem `main/protocols/`.
5. Nếu bạn muốn client trỏ sang server khác thì vào file esp32-client/main/Kconfig.projbuild

Chúc các bạn có cái nhìn tổng quan dễ nhất về source code và tự do điều chỉnh thiết bị theo ý thích.
