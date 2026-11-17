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

**Changelog**

| # | Thời gian | Thay đổi |
|---|---|---|
| 1 | 22:43 17/11/26 | Bổ sung cấu trúc sdkconfig/menuconfig và cách đổi cấu hình; ghi kết quả build chip `c5` bằng ESP-IDF 6.1 local (thành công sau 2 bản vá tương thích IDF 6, tổng 8 dòng) |

## 1. Vấn đề và mục tiêu giải quyết

Repo ESPectre chia làm hai phần: `src/cpp/` (firmware C++ flash lên ESP32) và `src/python/` (client desktop: script, thu thập và xử lý dữ liệu CSI). Hiện dự án CSI đang flash thiết bị bằng web flasher (`https://espectre.dev/tools/flash/`), tức là chỉ dùng được image có sẵn. 

Mục tiêu của báo cáo này: nắm đường build và flash firmware **Native** (ESP-IDF thuần, không dùng web flasher) để có thể tùy biến cấu hình và tự build image cho đúng chip của mình. Chỉ xét đường **Native** (`src/cpp/frontend/native/`), không xét ESPHome hay Matter.

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

### Cách đổi cấu hình: cấu trúc sdkconfig / menuconfig

ESP-IDF tự gom mọi `Kconfig.projbuild` của các component trong app Native (`src/cpp/frontend/native/app/CMakeLists.txt` thêm `EXTRA_COMPONENT_DIRS` là `../espectre` và `runtime/esp_idf/espectre_config`):

- Component `espectre` = `src/cpp/frontend/native/espectre`: `Kconfig.projbuild` của nó `rsource "../../Kconfig.projbuild"` (tức `frontend/Kconfig.projbuild`) → menu **"ESPectre Native Frontend"** (Wi-Fi, MQTT, HA discovery, recovery button, loop priority, dev origins) và menu **"ESPectre Firmware"** (`ESPECTRE_TINYUSB_PRIMARY_CONSOLE`, chỉ S2).
- Component `espectre_config` = `src/cpp/runtime/esp_idf/espectre_config` → menu **"ESPectre Sensing"** (band, detection profile, window, CSI pps, capture profile, traffic mode, TX rate, filter, task priority).
- Menu **"ESPectre SDK"** ở `src/cpp/Kconfig.projbuild` chỉ xuất hiện khi build với SDK bundle bên ngoài, không có trong app Native.

Luồng file cấu hình:

```
sdkconfig.defaults  +  sdkconfig.defaults.<idf_target>  (+ sdkconfig.wifi nếu tạo)
        │  (CLI truyền -DSDKCONFIG_DEFAULTS="...;..." và -DSDKCONFIG=build-<target>/sdkconfig)
        ▼  merge lúc configure
build-<target>/sdkconfig   ← file kết quả, đây là thứ menuconfig sửa
```

- `idf.py -B build-esp32c5 menuconfig` mở giao diện menu và ghi vào `build-esp32c5/sdkconfig`; build lại (`build`) sẽ giữ nguyên các giá trị đã sửa (incremental). `idf.py -B build-esp32c5 fullclean` (hoặc `--clean-all` của CLI) xóa sạch.
- Để bền trong repo: thêm/sửa dòng `CONFIG_...=...` trong `sdkconfig.defaults` hoặc `sdkconfig.defaults.esp32c5`, hoặc tạo `sdkconfig.wifi` (đã bị gitignore, dùng cho ảnh lab — CLI tự thêm vào `SDKCONFIG_DEFAULTS`). Muốn đóng băng cả một bộ config: copy `build-esp32c5/sdkconfig` ra ngoài rồi build với `ESPECTRE_IDF_SDKCONFIG=<file>`; hoặc override cả danh sách defaults bằng biến `SDKCONFIG_DEFAULTS`.

Các recipe hay dùng (sau đó build lại bằng `./espectre native build --chip c5`):

| Muốn đổi | Chỗ sửa |
| --- | --- |
| CSI pps, detection profile, band, traffic mode, filter | menu "ESPectre Sensing" trong menuconfig, hoặc dòng `CONFIG_ESPECTRE_CSI_TARGET_PPS=...` trong `sdkconfig.defaults.esp32c5` |
| Wi-Fi SSID/password lúc build | tạo `sdkconfig.wifi` với `CONFIG_ESPECTRE_WIFI_SSID`/`CONFIG_ESPECTRE_WIFI_PASSWORD` (runtime Improv/NVS vẫn ghi đè được) |
| MQTT, HA discovery, nút BOOT recovery | menu "ESPectre Native Frontend" |
| Kênh OTA mặc định | `--ota-channel release\|preview\|develop` (biến `NATIVE_OTA_CHANNEL`) → compile definition `ESPECTRE_OTA_*_BUILD` |

Lưu ý khi dùng wrapper với ESP-IDF local: `./espectre` tự prepend `.venv/bin` vào `PATH` khi re-exec, làm `idf.py` của IDF 6.1 báo `No module named 'esp_idf_monitor'`. Cách né: gọi trực tiếp `.venv/bin/python ./espectre native build ...` sau khi `source export.sh` (bỏ qua re-exec, `PATH` giữ nguyên của IDF), hoặc dùng `idf.py` thuần như khối lệnh trên.

Các cấu hình được phép (allowed configuration) của đường Native:

- **Chip / IDF target**: 6 target ở trên; mỗi target có `sdkconfig.defaults.<target>` riêng (CPU freq: 240 MHz cho esp32/s2/s3/c5, 160 MHz cho c3/c6; console: `esp32` dùng UART, `esp32s2` dùng TinyUSB CDC, còn lại dùng USB Serial/JTAG).
- **`sdkconfig.defaults` chung**: flash 4 MB (`CONFIG_ESPTOOLPY_FLASHSIZE_4MB`), tối ưu size, log v2, FreeRTOS 1000 Hz, bật CSI (`CONFIG_ESP_WIFI_CSI_ENABLED`), bảng phân vùng custom `partitions.csv` (nvs / otadata / nvs_key / hai slot OTA `app0`/`app1` mỗi slot `0x1E0000`), console USB Serial/JTAG, mbedTLS certificate bundle full, MQTT over SSL, HTTPD WebSocket, detector mặc định `LIGHTWEIGHT`, Home Assistant discovery bật sẵn, kênh Wi-Fi `0` (auto).
- **Kconfig (menu "ESPectre Sensing", "ESPectre Native Frontend", "ESPectre Firmware", "ESPectre SDK")**: chọn band Wi-Fi (2G/5G/auto, 5G chỉ khi chip hỗ trợ), detection profile (lightweight / high_accuracy), segmentation window 1000–2000 ms, CSI target 1–500 pps, capture profile (AUTO/LLTF/HT-VHT), traffic generator (ping / dns / dns_tcp / wifi_raw / external), TX rate, chu kỳ đánh giá, bộ lọc Hampel/low-pass, priority các task; cấu hình Wi-Fi SSID/password/BSSID/channel, MQTT (host/scheme/port/credential), topic prefix, HA discovery, nút BOOT recovery (GPIO, thời gian giữ 1000–10000 ms), TinyUSB console cho S2. Runtime (NVS) ghi đè giá trị build-time.
- **Biến môi trường / file tùy biến**: `SDKCONFIG_DEFAULTS` (override toàn bộ), `ESPECTRE_IDF_SDKCONFIG` (dùng sdkconfig custom), `ESPECTRE_IDF_BUILD_DIR`, `NATIVE_OTA_CHANNEL` (`release|preview|develop` → compile definition `ESPECTRE_OTA_*_BUILD` trong `main/CMakeLists.txt`), `ESPECTRE_SDK_ROOT` (build với SDK bundle ngoài repo); nếu tạo file `sdkconfig.wifi` trong app dir thì CLI tự thêm vào `SDKCONFIG_DEFAULTS` (dùng để nhúng Wi-Fi cho ảnh lab — không hardcode credential vào production default theo `AGENTS.md`).
- **Phiên bản firmware**: lấy từ `git describe --tags --match "[0-9]*"` (`src/cpp/espectre_git_version.cmake`), không phải version tay.
- **Managed components** (`idf_component.yml`): `improv/improv 1.2.7` (provisioning Improv Serial), `espressif/mdns`, `espressif/esp_tinyusb` (chỉ S2).

Lưu ý khi chuyển từ web flasher sang bản tự build: image chính thức chỉ nhận OTA có chữ ký; personal build phải flash qua USB, và muốn quay lại official thì phải flash lại full factory image qua USB (`docs/SETUP.md` mục "Official images and personal builds").

## 4. Kết quả và đánh giá

- Đã đạt được:
  - Build thành công firmware Native cho chip **c5** bằng ESP-IDF 6.1 local: `source ~/.espressif/v6.1/esp-idf/export.sh` rồi `.venv/bin/python ./espectre native build --chip c5 --ota-channel develop` → `Project build complete`, exit 0. Image `build-esp32c5/espectre-native.bin` dung lượng **1.576.176 byte (0x180CF0)**, slot app `0x1E0000` còn 20% trống. Phiên bản firmware `3.0.0-1-g899a3fb` (từ `git describe`). CLI tự publish sang `build-flash-esp32c5/flasher_args.json` (0x2000 bootloader, 0x8000 partition table, 0xe000 ota_data, 0x20000 app; flash-mode dio, 4MB, 80m) để flash không cần IDF.
  - `sdkconfig` kết quả ở `build-esp32c5/sdkconfig` đủ các `CONFIG_ESPECTRE_*`: `CSI_TARGET_PPS=100`, `DETECTION_ALGORITHM_LIGHTWEIGHT`, `WIFI_BAND_AUTO`, `TRAFFIC_GENERATOR_MODE_PING`, `HA_DISCOVERY_ENABLED=y`, `RECOVERY_BUTTON_GPIO=28` (đúng BOOT của C5), MQTT off, Wi-Fi trống (chờ provision runtime).
  - Cần đúng **2 bản vá tương thích IDF 6.1** (tổng 8 dòng, cả hai là no-op trên bản 5.5.5 mà repo pin cho Docker): (1) thêm `espressif/mqtt` với rule `if: "idf_version >=6.0.0"` vào `src/cpp/frontend/native/espectre/idf_component.yml` (IDF 6 đã chuyển esp-mqtt ra component registry, nếu không thì báo `Failed to resolve component 'mqtt'`); (2) thêm `esp_driver_gpio` vào `REQUIRES` của `src/cpp/frontend/native/app/main/CMakeLists.txt` (IDF 6 tách `driver` thành các `esp_driver_*`, nếu không thì báo `app_main.cpp includes driver/gpio.h ... not in the requirements list`).
- Chưa đạt / hạn chế: chưa flash lên phần cứng thật (chưa có port) nên chưa kiểm chứng boot/log runtime; chưa mở menuconfig tương tác lần nào (mới đọc cấu trúc); bản personal build không nhận OTA signed nên về sau phải flash USB; các CMake warning còn lại là của chính ESP-IDF 6.1 (private include nội bộ), không phải của app.

## 5. Việc làm tiếp theo / Action items

- [x] Build firmware Native chip `c5` bằng ESP-IDF 6.1 local (kết quả ở Mục 4).
- [ ] Flash lên board: `./espectre native flash --chip c5 --port <port> [--erase]`, hoặc esptool thuần (đúng lệnh idf.py in ra): `python -m esptool --chip esp32c5 -b 460800 --before default-reset --after hard-reset write-flash --flash-mode dio --flash-size 4MB --flash-freq 80m 0x2000 build-esp32c5/bootloader/bootloader.bin 0x8000 build-esp32c5/partition_table/partition-table.bin 0xe000 build-esp32c5/ota_data_initial.bin 0x20000 build-esp32c5/espectre-native.bin`.
- [ ] Kiểm tra log boot bằng `./espectre monitor --port <port>` rồi provision Wi-Fi qua Improv Serial (`./espectre provision --ssid ...`), xác nhận thiết bị lên Direct HTTP (`./espectre devices --frontend native`).
- [ ] Thử đổi 1 option qua `idf.py -B build-esp32c5 menuconfig` (ví dụ `ESPECTRE_CSI_TARGET_PPS` hoặc traffic mode) rồi build lại incremental để xác nhận đường tùy biến hoạt động end-to-end.
- [ ] Quyết định giữ hay revert 2 bản vá tương thích IDF 6 trong `src/cpp` (hiện đang là thay đổi duy nhất trên worktree, dễ revert bằng git).

## 6. Các link, nguồn tài liệu tham khảo

- Native frontend README: `/home/ltkdt/Development/espectre/src/cpp/frontend/native/README.md`
- App ESP-IDF: `/home/ltkdt/Development/espectre/src/cpp/frontend/native/app/` (`CMakeLists.txt`, `partitions.csv`, `sdkconfig.defaults*`, `main/app_main.cpp`)
- Kconfig: `src/cpp/runtime/esp_idf/espectre_config/Kconfig.projbuild`, `src/cpp/frontend/native/espectre/Kconfig.projbuild`, `src/cpp/frontend/Kconfig.projbuild`, `src/cpp/Kconfig.projbuild`
- CLI nguồn: `src/python/espectre_cli/idf.py`, `esptool_runner.py`, `targets.py`, `build_artifacts.py`, `app.py`
- Docs: `docs/CLI.md` (mục "Local build prerequisites", "`native` and `matter`"), `docs/SETUP.md` (supported boards, official vs personal builds), `docs/API.md`, `docs/CSI.md`
- Web flasher đang dùng: https://espectre.dev/tools/flash/
- ESP-IDF Get Started (cài local): https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/index.html
- ESP-IDF local của máy: `~/.espressif/v6.1/esp-idf` (`source export.sh`)
