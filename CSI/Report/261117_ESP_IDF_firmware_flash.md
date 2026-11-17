---
title: "ESP_IDF_firmware_flash"
author: "Kiên"
date: 2026-11-17
project: "CSI"
status: draft
tags: [bao-cao, esp-idf, firmware, flash]
---

# ESP_IDF_firmware_flash

**Người thực hiện:** Kiên · **Ngày thực hiện:** 17/11/2026

## 1. Vấn đề và mục tiêu giải quyết

Repo ESPectre chia làm hai phần: `src/cpp/` (firmware C++ flash lên ESP32) và `src/python/` (client desktop: script, thu thập và xử lý dữ liệu CSI). Hiện dự án CSI đang flash thiết bị bằng web flasher (`https://espectre.dev/tools/flash/`), tức là chỉ dùng được image có sẵn. Mục tiêu của báo cáo này: nắm đường build và flash firmware **Native** (ESP-IDF thuần, không dùng web flasher) để có thể tùy biến cấu hình và tự build image cho đúng chip của mình. Chỉ xét đường **Native** (`src/cpp/frontend/native/`), không xét ESPHome hay Matter.

## 2. Các yêu cầu cần thiết trước

- Repo `espectre` checkout tại `/home/ltkdt/Development/espectre` (nhánh `main`), kèm `.venv` của repo nếu dùng wrapper `./espectre`.
- ESP-IDF local bản 6.1 tại `~/.espressif/v6.1/esp-idf` (đã có sẵn trên máy, đã kiểm chứng: `source export.sh` chạy được, `idf.py --version` ra `ESP-IDF v6.1`, `esptool 5.5.0`). App yêu cầu ESP-IDF >= 5.5 (`src/cpp/frontend/native/app/CMakeLists.txt` ghi "ESP-IDF v5.5+"; `idf_component.yml` yêu cầu `idf >=5.5.3` cho ESP32-C5 LLTF), nên bản 6.1 dùng được cho đường local. CLI của repo pin bản `5.5.5` chỉ cho backend Docker; backend local tự nhận `IDF_PATH` đang active.
- Board ESP32 thuộc một trong 6 target được hỗ trợ: `esp32`, `esp32s2`, `esp32s3`, `esp32c3`, `esp32c5`, `esp32c6` (CLI alias chip: `esp32`, `s2`, `s3`, `c3`, `c5`, `c6` — xem `src/python/espectre_cli/targets.py`). Flash 4 MB.
- Cáp USB, xác định được serial port (ví dụ `/dev/cu.usbmodemXXXX` trên macOS, `/dev/ttyUSB*` trên Linux). Board ESP32-S2 dùng USB CDC phải vào sẵn download mode khi flash.
- Docker (tùy chọn, chỉ dùng làm backend build dự phòng khi không có ESP-IDF local).

## 3. Nội dung đã thực hiện

Đã inspect đường firmware Native trong `src/cpp/frontend/native/` và cách CLI `./espectre native ...` gọi `idf.py`/`esptool` (`src/python/espectre_cli/idf.py`, `esptool_runner.py`, `targets.py`).

| Module / Hạng mục | Chức năng chính | Lệnh / Cách chạy |
| ----------------- | --------------- | ---------------- |
| App ESP-IDF Native (`src/cpp/frontend/native/app/`) | Project `espectre-native`: entrypoint `main/app_main.cpp`, `partitions.csv`, `sdkconfig.defaults` + `sdkconfig.defaults.<idf_target>` | `idf.py -B build-<target> ... build` (chi tiết bên dưới) |
| CLI build (`./espectre native build`) | Gọi `idf.py -B build-<target> -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.<target>" -DIDF_TARGET=<target> -DESPECTRE_SDK_ROOT= -DSDKCONFIG=build-<target>/sdkconfig -DNATIVE_OTA_CHANNEL=<channel> build` trong app dir; bản thành công được copy (content-addressed) sang `build-flash-<target>/` | `./espectre native build --chip c3 [--ota-channel develop] [--clean\|--clean-all] [--backend auto\|local\|docker] [--json]` |
| CLI flash (`./espectre native flash`) | Đọc `build-flash-<target>/flasher_args.json`, chạy `python -m esptool --chip <target> --port <port> --baud 460800 --before default-reset --after hard-reset\|watchdog-reset write-flash ...` (baud `115200` cho `esp32` và S2 USB CDC; `--erase` thêm `--erase-all`). Không cần ESP-IDF/Docker | `./espectre native flash --chip c3 --port /dev/cu.usbmodemXXXX [--erase]` |
| Monitor | Serial log của thiết bị | `./espectre monitor --port /dev/cu.usbmodemXXXX` |
| menuconfig (tùy biến sâu) | Sửa mọi `CONFIG_ESPECTRE_*` trong sdkconfig của build dir | `idf.py -B build-<target> menuconfig` |

Các lệnh build/flash thuần ESP-IDF (tương đương những gì CLI chạy, dùng để tùy biến ngoài CLI), với ESP-IDF 6.1 local:

```bash
# 1. Kích hoạt ESP-IDF 6.1 local
source ~/.espressif/v6.1/esp-idf/export.sh

# 2. Build app Native cho một chip (ví dụ ESP32-C3)
cd /home/ltkdt/Development/espectre/src/cpp/frontend/native/app
idf.py -B build-esp32c3 \
  -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.esp32c3" \
  -DIDF_TARGET=esp32c3 \
  -DESPECTRE_SDK_ROOT= \
  -DSDKCONFIG=build-esp32c3/sdkconfig \
  -DNATIVE_OTA_CHANNEL=release \
  build
# Image: build-esp32c3/espectre-native.bin

# 3. Tùy biến cấu hình bằng menuconfig (sau đó build lại)
idf.py -B build-esp32c3 menuconfig

# 4. Flash (cách 1: idf.py) hoặc esptool với flasher_args.json (cách 2: như CLI)
idf.py -B build-esp32c3 -p /dev/cu.usbmodemXXXX flash
# hoặc trong build-esp32c3: python -m esptool --chip esp32c3 --port /dev/cu.usbmodemXXXX \
#   --baud 460800 --before default-reset --after watchdog-reset write-flash <các cặp offset file từ flasher_args.json>

# 5. Xem log
idf.py -B build-esp32c3 -p /dev/cu.usbmodemXXXX monitor
```

Các cấu hình được phép (allowed configuration) của đường Native:

- **Chip / IDF target**: 6 target ở trên; mỗi target có `sdkconfig.defaults.<target>` riêng (CPU freq: 240 MHz cho esp32/s2/s3/c5, 160 MHz cho c3/c6; console: `esp32` dùng UART, `esp32s2` dùng TinyUSB CDC, còn lại dùng USB Serial/JTAG).
- **`sdkconfig.defaults` chung**: flash 4 MB (`CONFIG_ESPTOOLPY_FLASHSIZE_4MB`), tối ưu size, log v2, FreeRTOS 1000 Hz, bật CSI (`CONFIG_ESP_WIFI_CSI_ENABLED`), bảng phân vùng custom `partitions.csv` (nvs / otadata / nvs_key / hai slot OTA `app0`/`app1` mỗi slot `0x1E0000`), console USB Serial/JTAG, mbedTLS certificate bundle full, MQTT over SSL, HTTPD WebSocket, detector mặc định `LIGHTWEIGHT`, Home Assistant discovery bật sẵn, kênh Wi-Fi `0` (auto).
- **Kconfig (menu "ESPectre Sensing", "ESPectre Native Frontend", "ESPectre Firmware", "ESPectre SDK")**: chọn band Wi-Fi (2G/5G/auto, 5G chỉ khi chip hỗ trợ), detection profile (lightweight / high_accuracy), segmentation window 1000–2000 ms, CSI target 1–500 pps, capture profile (AUTO/LLTF/HT-VHT), traffic generator (ping / dns / dns_tcp / wifi_raw / external), TX rate, chu kỳ đánh giá, bộ lọc Hampel/low-pass, priority các task; cấu hình Wi-Fi SSID/password/BSSID/channel, MQTT (host/scheme/port/credential), topic prefix, HA discovery, nút BOOT recovery (GPIO, thời gian giữ 1000–10000 ms), TinyUSB console cho S2. Runtime (NVS) ghi đè giá trị build-time.
- **Biến môi trường / file tùy biến**: `SDKCONFIG_DEFAULTS` (override toàn bộ), `ESPECTRE_IDF_SDKCONFIG` (dùng sdkconfig custom), `ESPECTRE_IDF_BUILD_DIR`, `NATIVE_OTA_CHANNEL` (`release|preview|develop` → compile definition `ESPECTRE_OTA_*_BUILD` trong `main/CMakeLists.txt`), `ESPECTRE_SDK_ROOT` (build với SDK bundle ngoài repo); nếu tạo file `sdkconfig.wifi` trong app dir thì CLI tự thêm vào `SDKCONFIG_DEFAULTS` (dùng để nhúng Wi-Fi cho ảnh lab — không hardcode credential vào production default theo `AGENTS.md`).
- **Phiên bản firmware**: lấy từ `git describe --tags --match "[0-9]*"` (`src/cpp/espectre_git_version.cmake`), không phải version tay.
- **Managed components** (`idf_component.yml`): `improv/improv 1.2.7` (provisioning Improv Serial), `espressif/mdns`, `espressif/esp_tinyusb` (chỉ S2).

Lưu ý khi chuyển từ web flasher sang bản tự build: image chính thức chỉ nhận OTA có chữ ký; personal build phải flash qua USB, và muốn quay lại official thì phải flash lại full factory image qua USB (`docs/SETUP.md` mục "Official images and personal builds").

## 4. Kết quả và đánh giá

- Đã đạt được: xác định đầy đủ đường Native (`src/cpp/frontend/native/app`), các lệnh build/flash (qua `./espectre native ...` hoặc `idf.py`/`esptool` thuần), danh sách cấu hình được phép (sdkconfig.defaults theo chip + Kconfig), và kiểm chứng môi trường ESP-IDF 6.1 local hoạt động (`idf.py --version` → `ESP-IDF v6.1`, `esptool 5.5.0`, `IDF_PATH=/home/ltkdt/.espressif/v6.1/esp-idf`).
- Chưa đạt / hạn chế: chưa chạy build và chưa flash (chờ phê duyệt); chưa kiểm chứng thực tế bản ESP-IDF 6.1 biên dịch được toàn bộ tính năng của app (repo mới pin chính thức 5.5.5 cho Docker, app yêu cầu >= 5.5); chưa có số liệu dung lượng image `espectre-native.bin` cho từng chip.

## 5. Việc làm tiếp theo / Action items

- [ ] Phê duyệt để chạy build thật với ESP-IDF 6.1 local (đề xuất chip `c3` hoặc `c5` đang dùng trong dự án CSI).
- [ ] Chạy `./espectre native build --chip <chip> --ota-channel develop` (hoặc lệnh `idf.py` thuần ở Mục 3) và ghi lại dung lượng/time build.
- [ ] Flash lên board qua `./espectre native flash --chip <chip> --port <port> [--erase]` và kiểm tra log bằng `./espectre monitor`.
- [ ] Thử tùy biến một option qua `menuconfig` (ví dụ `ESPECTRE_CSI_TARGET_PPS`, traffic mode) rồi build lại để xác nhận đường tùy biến hoạt động.
- [ ] Provision Wi-Fi bằng Improv Serial (`./espectre provision --ssid ...`) và xác nhận thiết bị lên Direct HTTP (`./espectre devices --frontend native`).

## 6. Các link, nguồn tài liệu tham khảo

- Native frontend README: `/home/ltkdt/Development/espectre/src/cpp/frontend/native/README.md`
- App ESP-IDF: `/home/ltkdt/Development/espectre/src/cpp/frontend/native/app/` (`CMakeLists.txt`, `partitions.csv`, `sdkconfig.defaults*`, `main/app_main.cpp`)
- Kconfig: `src/cpp/runtime/esp_idf/espectre_config/Kconfig.projbuild`, `src/cpp/frontend/native/espectre/Kconfig.projbuild`, `src/cpp/frontend/Kconfig.projbuild`, `src/cpp/Kconfig.projbuild`
- CLI nguồn: `src/python/espectre_cli/idf.py`, `esptool_runner.py`, `targets.py`, `build_artifacts.py`, `app.py`
- Docs: `docs/CLI.md` (mục "Local build prerequisites", "`native` and `matter`"), `docs/SETUP.md` (supported boards, official vs personal builds), `docs/API.md`, `docs/CSI.md`
- Web flasher đang dùng: https://espectre.dev/tools/flash/
- ESP-IDF Get Started (cài local): https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/index.html
- ESP-IDF local của máy: `~/.espressif/v6.1/esp-idf` (`source export.sh`)
