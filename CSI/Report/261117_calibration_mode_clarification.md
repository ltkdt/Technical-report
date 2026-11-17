---
title: "Calibration mode clarification"
author: "Kiên"
date: 2026-11-17
project: "CSI"
status: draft
tags: [bao-cao, calibration, sensing-state, direct-http]
---

# Calibration mode clarification

**Người thực hiện:** Kiên · **Ngày thực hiện:** 17/11/2026

## 1. Vấn đề và mục tiêu giải quyết

Ứng dụng rt-wireless-sensing (xem báo cáo `171126_rt_wireless_sensing_0.1.0.md`) cần biết ESP32 đang ở trạng thái **calibrating** hay đã **ready** để quyết định khi nào bắt đầu thu/đánh giá dữ liệu. Báo cáo này trả lời 3 câu hỏi, đọc tài liệu trước rồi đối chiếu mã nguồn repo `espectre`:

1. Phía C++ (firmware), ESP32 tự biết mình đang calibration bằng cách nào?
2. Phía client (laptop chạy lệnh collection), làm sao biết ESP32 vẫn đang calibration?
3. ESP32 làm gì trong lúc calibration?

## 2. Các yêu cầu cần thiết trước

- Repo `espectre` tại `/home/ltkdt/Development/espectre` (đọc docs trước: `docs/API.md`, `docs/ALGORITHMS.md`, `docs/CSI.md`, `docs/SDK.md`, `docs/TROUBLESHOOTING.md`, rồi mới đọc source `src/cpp/` và `src/python/espectre_cli/`).
- Thiết bị ESP32 chạy firmware Native với Direct HTTP (mặc định bật); laptop cùng mạng, biết IP/hostname của thiết bị.
- Để kiểm tra nhanh bằng CLI: `./espectre direct get sensing --target <ip-hoac-host>` (xem `docs/CLI.md` mục `direct`).

## 3. Nội dung đã thực hiện

Đã đọc docs và đối chiếu source. Kết quả tóm tắt theo 3 câu hỏi:

| Câu hỏi | Cơ chế chính | Nơi trong source |
| --- | --- | --- |
| ESP32 tự biết đang calibration (C++) | Cờ `snapshot_.calibrating` trong `EspIdfRuntime`, set/clear ở `start_calibration_()` / `cancel_calibration_()`; detector có `calibrating_` + hook begin/complete/abandoned; readiness gate gắn `CALIBRATING` làm `ready_to_publish=false` | `src/cpp/runtime/runtime_snapshot.h:173`, `src/cpp/runtime/esp_idf/esp_idf_runtime.cpp` (`is_calibrating()` ~:599, set ~:1368, clear ~:825/:1472), `src/cpp/runtime/sensing_readiness_gate.h`, `src/cpp/core/lightweight_detector.cpp:29`, `src/cpp/core/base_detector.h` |
| Client biết ESP32 đang calibration | Resource `sensing` có 2 trường `ready` và `calibrating`; publish qua Direct HTTP `GET /espectre/v1/sensing`, SSE event `sensing` ở `GET /espectre/v1/events`, và MQTT retained topic `espectre/v1/devices/{device_id}/sensing` | `src/cpp/runtime/esp_idf/runtime_direct_http_bridge.cpp:651-668` (`sensing_payload_()`: `ready = ready_to_publish && !collecting`, `calibrating = is_calibrating()`); docs `docs/API.md` mục `sensing` và `Events` |
| ESP32 làm gì lúc calibration | Lightweight: đọc 10 s đánh giá "phòng im" để tính ngưỡng q95 (mở rộng từng bước 5 s tối đa ~30 s, bỏ cụm nhiễu, ngưỡng kẹp trong [0.095, 0.987]); trong lúc đó vẫn thu CSI và chạy đánh giá nhưng không phát motion event; xong thì áp ngưỡng mới. High Accuracy không calibration | `docs/ALGORITHMS.md` "Startup threshold calibration"; `src/cpp/runtime/esp_idf/esp_idf_runtime.cpp` `start_calibration_()` (~:1330-1380: `threshold_calibrator_->begin()`, packet interceptor, `calibration_motion_guard_`); `docs/CSI.md:77` |

Chi tiết từng câu hỏi:

**1. Phía C++ — ESP32 tự biết mình đang calibration:**

- Trạng thái chuẩn là `RuntimeSnapshot.calibrating` (`bool`, `runtime_snapshot.h:173`) kèm `calibration_packets` / `calibration_target_packets`. `EspIdfRuntime::start_calibration_()` đặt `snapshot_.calibrating = true` và bắt đầu `threshold_calibrator_` với budget = số slot của `window_size_ms * CALIBRATION_NUM_WINDOWS`; một packet interceptor đếm từng gói CSI vào calibrator. `is_calibrating()` đọc trực tiếp cờ này; `cancel_calibration_()` và nhánh kết thúc đều clear cờ và gọi `on_calibration_finished(snapshot, success)` cho listener (`runtime_events.h:113-127`).
- Ở tầng detector, `LightweightDetector` có `calibrating_` và các hook `on_startup_calibration_begin/complete/abandoned` (`base_detector.h:209-221`); `HighAccuracyDetector` không calibration (threshold đã train).
- Readiness gate (`sensing_readiness_gate.h`): `inputs.calibrating` → `SensingReadinessReason::CALIBRATING` → `ready_to_publish` bị false (`esp_idf_runtime.cpp:321`: `ready_to_publish = ready_to_publish && !calibrating && input còn mới...`). Nói cách khác: đang calibration thì coi như chưa ready, kể cả khi đã có detector.

**2. Phía client — laptop biết ESP32 đang calibration:**

- Cách trực tiếp và đủ dùng cho ứng dụng: đọc resource `sensing`:
  - Direct HTTP: `GET http://<esp-ip>/espectre/v1/sensing` → JSON có `"ready": bool` và `"calibrating": bool` (cùng `mode`, `detector`, `threshold`, `traffic_generator_mode`, ...). CLI tương đương: `./espectre direct get sensing --target <ip>`.
  - Push theo sự kiện: `GET /espectre/v1/events` (SSE) phát event `sensing` nguyên resource mỗi khi trạng thái đổi — `calibrating` chuyển true→false là tín hiệu "calibration xong".
  - MQTT: topic retained `espectre/v1/devices/{device_id}/sensing` (cùng payload), base topic `espectre/v1/devices/{device_id}`.
- `POST /espectre/v1/sensing/calibrations` (hoặc MQTT command `recalibrate`) để yêu cầu calibration: `202` nếu bắt đầu, `409` code `busy` nếu đang chạy (`docs/API.md` "Sensing update and calibration").
- Bản thân lệnh `collect` của CLI đọc `GET sensing` để kiểm tra cấu hình (`src/python/espectre_cli/host.py:163`) và chạy detector replay cục bộ với một calibration tracker mirror đúng luồng `EspIdfRuntime::start_calibration_` (`host.py:624-631`) để gắn nhãn CALIBRATING/READY; cờ `--ready-stable-seconds` (mặc định 3.0s) chặn việc lưu dữ liệu cho tới khi metric nằm dưới threshold đủ lâu.
- Lưu ý: khi đang mở phiên raw CSI (`GET /csi`), `mode` là `csi_collection`, `ready` bị ép false và các event suy ra (motion) bị pause trên mọi transport.

**3. ESP32 làm gì trong calibration mode:**

- Vẫn thu CSI và chạy traffic generator bình thường; vẫn chạy đánh giá detector từng chu kỳ (`ESPECTRE_EVALUATION_INTERVAL_MS`, mặc định 250 ms) — nhưng vì `calibrating=true` nên `ready_to_publish=false`, motion event không được phát ra (client chỉ thấy `sensing.calibrating=true`).
- Lightweight học ngưỡng từ dữ liệu "phòng im": đọc 10 s đánh giá sẵn sàng (~40 đánh giá ở cadence mặc định), nếu bỏ cụm 5 s nhiễu nhất mà làm q95 khởi động giảm > 2.5 logit thì coi là có burst và tiếp tục từng bước 5 s, tối đa ~30 s; kết quả được kẹp trong [0.095, 0.987] và bằng q95 của phần còn lại. Mọi đánh giá trên motion ceiling (0.9933; hoặc trên ngưỡng đang dùng khi recalibrate cùng setup) sẽ restart calibration. Nếu ~30 s không có cửa sổ im thì calibration fail nhưng không fatal — giữ nguyên ngưỡng cũ (`docs/ALGORITHMS.md` "Startup threshold calibration").
- Khi xong: áp threshold mới, clear cờ `calibrating`, phát `on_calibration_finished(success=true)` + `on_threshold_changed`, và publish event `sensing` mới (`calibrating:false`, `ready` có thể thành true). Sau đó Lightweight còn có cơ chế "settled-level recovery" chỉ hạ ngưỡng, không tăng.
- Đổi detector hoặc đổi traffic mode sẽ restart calibration của Lightweight (`docs/CSI.md:77`). High Accuracy không calibration mà chỉ đợi đủ CSI và đầy cửa sổ feature.

Lệnh kiểm tra nhanh trên laptop:

```bash
# Trạng thái hiện tại của ESP32 (ready/calibrating)
./espectre direct get sensing --target 192.168.1.50
# hoặc curl: curl -s http://192.168.1.50/espectre/v1/sensing | jq '{ready, calibrating, mode, detector, threshold}'

# Theo dõi realtime qua SSE (event "sensing" đổi khi calibration xong)
curl -N http://192.168.1.50/espectre/v1/events | grep --line-buffered sensing

# Yêu cầu recalibrate bằng tay
./espectre direct post sensing/calibrations --target 192.168.1.50   # 202 = bắt đầu, 409 busy = đang chạy
```

## 4. Kết quả và đánh giá

- Đã đạt được: câu trả lời rõ ràng cho cả 3 câu hỏi, có đối chiếu docs ↔ source. Điểm mấu chốt cho ứng dụng rt-wireless-sensing: chỉ cần đọc `sensing.calibrating` (đang học) và `sensing.ready` (output an toàn để dùng) từ `GET /espectre/v1/sensing` hoặc event SSE/MQTT `sensing` — không cần suy diễn từ motion event. `ready=false` bao gồm cả trường hợp đang calibration, đang warm-up, hoặc input bị stale (xem `SensingReadinessReason`).
- Chưa đạt / hạn chế: chưa kiểm chứng thực tế trên phần cứng (chưa đọc `sensing` từ một thiết bị đang boot; mới đọc từ source/docs); `ready` là điều kiện đủ nhưng không cần thiết để thu raw CSI — khi `mode=csi_collection` thì `ready=false` nhưng vẫn thu được dữ liệu; thời gian calibration thực tế phụ thuộc chất lượng luồng CSI (thiếu gói thì lâu hơn, có thể tới ~30 s rồi fail giữ ngưỡng cũ).

## 5. Việc làm tiếp theo / Action items

- [ ] Trong rt-wireless-sensing, trước khi thu mẫu: poll `GET /espectre/v1/sensing` tới khi `calibrating=false` và `ready=true` (hoặc đăng ký SSE event `sensing` để phản ứng tức thì).
- [ ] Ghi nhận `threshold` trong `sensing` sau calibration để so sánh giữa các lần chạy.
- [ ] Kiểm chứng thực tế: boot ESP32, theo dõi `sensing` qua SSE và ghi lại thời gian `calibrating` chuyển false trong môi trường phòng im/thực.
- [ ] Nếu cần ép recalibrate từ app: `POST /espectre/v1/sensing/calibrations`, xử lý `409 busy`.

## 6. Các link, nguồn tài liệu tham khảo

- `docs/API.md`: mục `sensing` resource (fields `ready`, `calibrating`), "Sensing update and calibration" (`POST /sensing/calibrations`), "Events" (SSE `sensing`), "MQTT topics" (retained `sensing`).
- `docs/ALGORITHMS.md`: "Startup threshold calibration" (10 s, bước 5 s, tối đa 30 s, ngưỡng kẹp [0.095, 0.987], motion ceiling 0.9933, fail giữ ngưỡng cũ).
- `docs/CSI.md`: đổi traffic mode restart calibration của Lightweight; `docs/SDK.md`: gate `ready_to_publish`; `docs/TROUBLESHOOTING.md`: calibration stall do thiếu input.
- Source C++: `src/cpp/runtime/runtime_snapshot.h` (`calibrating`, `calibration_packets`), `src/cpp/runtime/sensing_readiness_gate.h` (`SensingReadinessReason::CALIBRATING`), `src/cpp/runtime/esp_idf/esp_idf_runtime.cpp` (`start_calibration_`, `cancel_calibration_`, `is_calibrating`), `src/cpp/runtime/esp_idf/runtime_direct_http_bridge.cpp:651` (`sensing_payload_`), `src/cpp/runtime/runtime_events.h` (`on_calibration_started/finished`), `src/cpp/core/lightweight_detector.*`, `src/cpp/core/base_detector.h`.
- Source client: `src/python/espectre_cli/host.py` (`control.request("get", "sensing")`, calibration tracker mirror, `--ready-stable-seconds`), `src/python/espectre_cli/app.py` (lệnh `direct`, `collect`).
- Báo cáo liên quan: `CSI/Report/171126_rt_wireless_sensing_0.1.0.md` (dự án rt-wireless-sensing).
