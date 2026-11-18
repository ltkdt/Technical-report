---
title: "SDK build"
author: "Kiên"
date: 2026-11-18
project: "AICAM"
status: draft
tags: [sdk, build, ax620q, camera]
---

# SDK build

**Người thực hiện:** Kiên · **Ngày thực hiện:** 18/11/2026

## 1. Vấn đề và mục tiêu giải quyết

Nhà sản xuất (LingYanGe, viết tắt LYG) đã cung cấp bộ SDK cho board camera AI, đóng gói thành `LYG_AI_Camera_sdk_20261110_131300`. Hiện tại đây là **toàn bộ** những gì nhà sản xuất cung cấp (đã được gom về một thư mục duy nhất). Cần làm rõ:

- SDK chứa gì: cấu trúc thư mục, các thành phần (toolchain, tài liệu, mã nguồn MSP/app, kernel, thông số phần cứng board).
- Cách build SDK theo đúng hướng dẫn của nhà sản xuất (README của SDK).
- Trạng thái build hiện tại: phần nào build được, phần nào bị chặn và nguyên nhân.

Board mục tiêu là **FLCAM2611N0P-1X** (xem `Board_specs/`), dùng SoC AX620QF — xem Mục 3 để biết thông số phần cứng liên quan đến SDK.

## 2. Các yêu cầu cần thiết trước

- SDK root tại `/home/ltkdt/Development/LYG_AI_Camera_sdk_20261110_131300` (gốc repo hiện tại; bên trong là nội dung giải nén của `04-SDK` cùng các thư mục `00-ROM`, `01-Tools`, `02-Software Doc`, `Board_specs` của gói nhà sản xuất).
- Máy build host: x86_64, Debian 13 (trixie); có sẵn `make`, `gcc` (host), `python3`.
- Cross toolchain `arm-AX620E-linux-uclibcgnueabihf-` (GCC 8.3.0) nằm trong SDK tại `01-Tools/arm-AX620E-linux-uclibcgnueabihf/bin` — **không có sẵn trên PATH**, cần export (đã kiểm chứng chạy được: `gcc (GCC) 8.3.0`):

  ```bash
  cd /home/ltkdt/Development/LYG_AI_Camera_sdk_20261110_131300
  export PATH="$PWD/01-Tools/arm-AX620E-linux-uclibcgnueabihf/bin:$PATH"
  ```

- Target platform: `p=AX620Q_nand_arm32_k419` (AX620Q/AX620E, arm32, uclibc, kernel 4.19) — có trong danh sách hỗ trợ của SDK (`build/project.mak`, dòng `AX_PRJ_LIST += AX620Q_nand_arm32_k419`). Build system chỉ dùng prefix `CROSS := arm-AX620E-linux-uclibcgnueabihf-` trong `build/cross_arm_uclibc.mak` và tìm tool qua PATH.
- `third-party/` (bundle prebuilt: drm, tinyalsa, faac, fdk, opus, libsamplerate, mp4, live, opencv, openssl, optee) — **thiếu**, không có trong gói SDK (`04-SDK/LYG_AI_Camera_sdk_20261110_131300.tgz` bản gốc chứa 0 entry `third-party`, đã kiểm lại ngày 18/11/2026). Phải xin từ nhà sản xuất.
- Board spec để tham chiếu phần cứng: `Board_specs/AI Camera_Board_FLCAM2611N0P-1X_Specification EN.pdf` và `Board_specs/Isolated PoE Module Specification_EN.pdf`.

## 3. Nội dung đã thực hiện

### 3.1 Cấu trúc của SDK (toàn bộ những gì nhà sản xuất cung cấp tính đến nay)

| Thư mục / File | Nội dung chính | Ghi chú |
| --- | --- | --- |
| `00-ROM/` | Ảnh ROM cho board | Hiện đang **rỗng**, chưa được cung cấp |
| `01-Tools/` | Cross toolchain `arm-AX620E-linux-uclibcgnueabihf` (GCC 8.3.0, đã giải nén) + file nén gốc `.tgz`; `AXDL_V1.24.13.1.7z` (công cụ AXDL); `Driver_V1.20.46.1.7z`; `ISPTuning_V1.24.47.1_P03.7z` (tool tuning ISP); `tftpd32.464.rar` (TFTP server để nạp ảnh qua mạng) | Toolchain dùng để build toàn bộ SDK |
| `02-Software Doc/board/` | 26 file PDF tài liệu API/reference cho board: VIN, MIPI, ISP, 3A, IVPS, VENC, VDEC, VO, AUDIO, SYS, IVE, IVES, ENGINE, CIPHER, GZIPD, OPAL, SKEL, NT, error codes... | Ví dụ: `10 - AX VIN Reference Manual.pdf`, `15 - AX ISP API Document.pdf`, `30 - AX APP Demo User Manual.pdf` |
| `02-Software Doc/pc/` | `00 - AXDL Tool User Manual.pdf`, `01 - AXIQ Tool User Manual.pdf` | Tài liệu tool trên PC |
| `Board_specs/` | Thông số phần cứng board `FLCAM2611N0P-1X` và module PoE cách ly | Xem 3.2 |
| `msp/` | **MSP (Media Service Platform)** — lớp giữa: `component/isp_proton/sensor` (driver sensor), `sample/` (27 sample: vin, ivps, venc, vdec, vo, audio, rtsp, skel, npu, optee...), `out/` (output build: `out/arm_uclibc/{bin,lib,include,...}`) | Build từ đây ra `msp/out/arm_uclibc/` |
| `app/` | Ứng dụng đầu cuối: `component/` (24 component: mp4enc, rtsp, webserver, detector, osd, encoder...), `demo/` (`FRTDemo`, `QSDemo`), `opal/` | FRTDemo là demo build chính của README |
| `build/` | Hệ build dùng chung: `project.mak` (danh sách project hỗ trợ), `cross_arm_uclibc.mak` (prefix toolchain), `projects/AX620Q_nand_arm32_k419/` (cấu hình riêng cho target này) | Không tham chiếu đường dẫn tuyệt đối nào đến `01-Tools` |
| `kernel/linux/linux-4.19.125/` | Source kernel Linux 4.19.125 cho board | |
| `README.md` | Hướng dẫn build 3 bước của nhà sản xuất (xem 3.3) | |
| `status.md` | Báo cáo trạng thái build tại chỗ (đã cập nhật ngày 18/11/2026 theo layout mới) | |

Lưu ý: thư mục gốc của repo chính là nội dung giải nén của `04-SDK/LYG_AI_Camera_sdk_20261110_131300` trong gói gốc của nhà sản xuất (bản `.tgz` vẫn còn ở vị trí cũ, không nằm trong repo này).

### 3.2 Thông số board liên quan (từ `Board_specs/`)

Board **FLCAM2611N0P-1X** (mainboard camera edge AI AX620QF V10 của LingYanGe):

- Main processor: Dual-core Cortex-A53 lên đến 1 GHz, có NEON & FPU; NPU **2.4 TOPS @INT8**.
- ISP: AI-ISP 2DOL hỗ trợ cảm biến MIPI CSI đến 5MP, 5M@30fps; hỗ trợ 3A, AI 3DNR, 2-frame HDR, khử sương, fisheye, DIS/EIS.
- Bộ nhớ: 2 Gb (256 MB) LPDDR4X; lưu trữ: 128 MB SPI NAND FLASH (đúng với target `nand` trong `AX620Q_nand_arm32_k419`).
- Video codec H.264/H.265, hỗ trợ fast boot; HĐH **Linux** (kernel 4.19 trong SDK).
- Giao tiếp: 1× MIPI CSI, 1× Ethernet 10/100M **PoE**, 1× Type-C, 1× UART, IR-CUT, loa, mic; nguồn 12V; công suất < 3 W; −20 °C đến 60 °C; kích thước 38×38 mm.
- Đi kèm spec của **Isolated PoE Module** (`Isolated PoE Module Specification_EN.pdf`).

### 3.3 Các lệnh build theo README của SDK

Tất cả lệnh chạy từ SDK root với `p=AX620Q_nand_arm32_k419`:

```bash
cd /home/ltkdt/Development/LYG_AI_Camera_sdk_20261110_131300
export PATH="$PWD/01-Tools/arm-AX620E-linux-uclibcgnueabihf/bin:$PATH"

# Bước 1: build thành phần sensor
cd msp/component/isp_proton/sensor
make p=AX620Q_nand_arm32_k419 -j8

# Bước 2: build các sample
cd ../../sample
make p=AX620Q_nand_arm32_k419 -j8

# Bước 3: build app demo
cd ../../app/demo/FRTDemo
make p=AX620Q_nand_arm32_k419 -j8
```

Kết quả build theo README nằm ở `msp/out/<arch>/bin/` (với target này là `msp/out/arm_uclibc/bin/`).

### 3.4 Kết quả từng bước build

| # | Bước build | Lệnh | Kết quả |
| --- | --- | --- | --- |
| 1 | Sensor components (`msp/component/isp_proton/sensor`) | `make p=AX620Q_nand_arm32_k419 -j8` | **Thành công.** Cài `libsns_*.so/.a` (c4395, cv2005, cv4007, gc4653, dummy...) vào `msp/out/arm_uclibc/lib/` |
| 2 | Samples (`msp/sample`) | `make p=AX620Q_nand_arm32_k419 -j8` | **Một phần (exit 2).** Build được 8 sample (bản động và tĩnh `_s`, tổng 16 file trong `msp/out/arm_uclibc/bin/`): `sample_cmm`, `sample_isp_3a`, `sample_ivps`, `sample_ivps_venc`, `sample_pool`, `sample_rtc`, `sample_vdec`, `sample_vin`. Lỗi: `audio` (thiếu `tinyalsa/pcm.h`), `vo` (thiếu `third-party/drm/lib/arm/uclibc/libdrm.a`) |
| 3 | App demo (`app/demo/FRTDemo`) | `make p=AX620Q_nand_arm32_k419 -j8` | **Lỗi.** `app/component/mp4enc/Mpeg4Encoder.h:17: fatal error: mp4_api.h: No such file or directory` — cần `third-party/mp4/include` + `third-party/mp4/lib/arm/uclibc/libmp4.a` |

## 4. Kết quả và đánh giá

- Đã đạt được:
  - Nắm được cấu trúc đầy đủ của SDK và vai trò từng thư mục (Mục 3.1); toàn bộ gói nhà sản xuất hiện đã được gom về một root duy nhất (`00-ROM`, `01-Tools`, `02-Software Doc`, `Board_specs` bên cạnh `app/`, `build/`, `kernel/`, `msp/`).
  - Việc gom thư mục không làm hỏng build: không file build nào tham chiếu đường dẫn tuyệt đối đến các thư mục này; toolchain chỉ được tìm qua PATH bằng prefix `CROSS`.
  - Build thành công bước 1 (sensor) và phần lớn bước 2: 8 sample trong `msp/out/arm_uclibc/bin/`. Đã kiểm chứng đúng định dạng đích: `sample_vin` là `ELF 32-bit LSB executable, ARM, EABI5, interpreter /lib/ld-uClibc.so.0` — khớp ARM32 uclibc mà target `AX620Q_nand_arm32_k419` yêu cầu, đúng vị trí output theo README.
- Chưa đạt / hạn chế:
  - `third-party/` không có trong gói SDK (file `.tgz` gốc trong `04-SDK` chứa 0 entry `third-party`, đã kiểm lại 18/11/2026) → sample `audio`, `vo` và toàn bộ `app/demo/FRTDemo` (cần `mp4_api.h`/`libmp4.a`) chưa build được. Đây là blocker duy nhất còn lại, cần xin bundle prebuilt từ nhà sản xuất.
  - `00-ROM/` rỗng — chưa có ảnh ROM; chưa build được kernel/image hoàn chỉnh của board (ngoài source kernel 4.19.125 có sẵn trong `kernel/`).
  - Toolchain chưa được đưa vĩnh viễn vào PATH (mới export tay theo phiên làm việc).

## 5. Việc làm tiếp theo / Action items

- [ ] Đưa toolchain vào PATH vĩnh viễn: thêm `01-Tools/arm-AX620E-linux-uclibcgnueabihf/bin` (tương đối từ SDK root) vào shell profile hoặc symlink vào `/usr/local/bin`.
- [ ] Yêu cầu nhà sản xuất bundle `third-party/` prebuilt và giải nén vào SDK root.
- [ ] Sau khi có `third-party/`, build lại bước 2 (`msp/sample`) và bước 3 (`app/demo/FRTDemo`) để hoàn thành đủ build theo README.
- [ ] Kiểm tra lại toàn bộ executable nằm trong `msp/out/arm_uclibc/bin/` và đúng chuẩn ARM32 uclibc ELF theo mục Build Output của README.
- [ ] Khi có `00-ROM`, đối chiếu quy trình nạp ảnh (TFTP/tftpd32 trong `01-Tools`) với tài liệu board.

## 6. Các link, nguồn tài liệu tham khảo

- SDK root: [../../../LYG_AI_Camera_sdk_20261110_131300/](../../../LYG_AI_Camera_sdk_20261110_131300/)
- Hướng dẫn build của nhà sản xuất: [../../../LYG_AI_Camera_sdk_20261110_131300/README.md](../../../LYG_AI_Camera_sdk_20261110_131300/README.md)
- Báo cáo trạng thái build tại chỗ: [../../../LYG_AI_Camera_sdk_20261110_131300/status.md](../../../LYG_AI_Camera_sdk_20261110_131300/status.md)
- Thông số board: [../../../LYG_AI_Camera_sdk_20261110_131300/Board_specs/AI Camera_Board_FLCAM2611N0P-1X_Specification EN.pdf](../../../LYG_AI_Camera_sdk_20261110_131300/Board_specs/AI Camera_Board_FLCAM2611N0P-1X_Specification EN.pdf), [../../../LYG_AI_Camera_sdk_20261110_131300/Board_specs/Isolated PoE Module Specification_EN.pdf](../../../LYG_AI_Camera_sdk_20261110_131300/Board_specs/Isolated PoE Module Specification_EN.pdf)
- Tài liệu API board (VIN/ISP/IVPS/VENC/VDEC/VO/AUDIO...): [../../../LYG_AI_Camera_sdk_20261110_131300/02-Software Doc/board/](../../../LYG_AI_Camera_sdk_20261110_131300/02-Software Doc/board/)
- Tài liệu tool PC (AXDL/AXIQ): [../../../LYG_AI_Camera_sdk_20261110_131300/02-Software Doc/pc/](../../../LYG_AI_Camera_sdk_20261110_131300/02-Software Doc/pc/)
- Toolchain trong SDK: [../../../LYG_AI_Camera_sdk_20261110_131300/01-Tools/arm-AX620E-linux-uclibcgnueabihf/](../../../LYG_AI_Camera_sdk_20261110_131300/01-Tools/arm-AX620E-linux-uclibcgnueabihf/)
- Gói SDK gốc của nhà sản xuất (bản `.tgz`, có `04-SDK` với 0 entry `third-party`): `/mnt/storage/Hust/Study material/2026.1/VXL/sdk/04-SDK/LYG_AI_Camera_sdk_20261110_131300.tgz`
