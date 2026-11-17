# Hiện trạng dự án RT-Wireless-Sensing

**Người thực hiện:** Kiên · **Ngày thực hiện:** 17/11/2026

## 1. Vấn đề và mục tiêu giải quyết

Cần một dự án theo dõi Wi‑Fi CSI trong thời gian thực, thu dữ liệu bằng ESP32 C5 chạy code của Espectre. Mục tiêu: quá trình lấy mẫu huấn luyện AI nhanh hơn, vì có thể kiểm tra mẫu đạt chất lượng hay không ngay trong quá trình đo.

## 2. Các yêu cầu cần thiết trước

- 1 hotspot (có thể là router) phát Wi‑Fi làm TX.
- ESP32 C5 được flash code của Espectre làm RX. Hiện đang dùng web flasher có sẵn: `https://espectre.dev/tools/flash/`.
- Laptop kết nối chung Wi‑Fi với thiết bị ESP, và có địa chỉ IP của ESP32 để làm tham số truyền.

## 3. Nội dung đã thực hiện

Dự án viết bằng Rust, gồm các module chính (tổng hợp từ README và docs của repo):


| Module                                     | Chức năng chính                                                                                                                                            | Lệnh chạy                                                                    |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Collection** (`src/collection.rs`)       | Mở `/espectre/v1/csi`, gửi UDP marker theo PPS cấu hình, ghi raw byte stream ra file `.bin`.                                                               | `cargo run --bin collect -- --ip <IP_ESP> --output capture_v8.bin --pps 100` |
| **Decoder** (`src/decoder.rs`)             | Parse tăng dần prefix HTTP 60 byte + header V8 64 byte, validate counter, trích I/Q, biên độ và phase.                                                     | `cargo run --bin decode -- capture_v8.bin`                                   |
| **Saving** (`src/saving.rs`)               | Decode file `.bin` và xuất `.npz` gồm timestamp + 64 mảng biên độ + 64 mảng phase (129 mảng cho 64 subcarrier).                                            | `cargo run --bin save -- capture_v8.bin`                                     |
| **DSP** (`src/dsp/`)                       | Xử lý tín hiệu offline trên CSI đã decode (`CsiSignal`): chuẩn hóa, resample cả bản ghi, Hampel/biquad filter, FFT/STFT. Chỉ là thư viện, không có binary. | (gọi từ code khác)                                                           |
| **Visualization** (`src/visualization.rs`) | Vẽ heatmap biên độ CSI 2D bằng HTML Canvas 2D, xuất HTML riêng hoặc serve qua HTTP.                                                                        | `cargo run --bin visualize -- capture_v8.bin`                                |
| **Streaming** (`src/streaming.rs`)         | Gom + decode CSI trực tiếp, dashboard heatmap trượt 30 s, warm-up 3 s, refresh 500 ms, click để tính occupancy của từng giây.                              | `cargo run --bin streaming -- --help`                                        |


Ngoài ra: thư mục `scripts/` (Python) để vẽ đồ thị và thống kê kiểm tra dữ liệu CSI, `web/` là dashboard HTML/JS, `samples/` chứa mẫu đã ghi, `docs_EN`/`docs_VN` là tài liệu từng module.

Pipeline tổng: ESP32 chạy firmware Espectre phát CSI qua HTTP → collection ghi `.bin` → decoder/saving cho ra `.npz` → DSP xử lý → streaming/visualization hiển thị trực tiếp. Phần AI inference (ONNX cho định vị) chưa triển khai.

Lệnh chính:

```bash
# Cài Rust nếu chưa có: https://rust-lang.org/tools/install/
cargo run --bin collect -- --ip 10.178.22.9 --output capture_v8.bin --pps 100
cargo run --bin decode -- capture_v8.bin
cargo run --bin save -- capture_v8.bin
cargo run --bin visualize -- capture_v8.bin
cargo run --bin streaming -- --help
```

## 4. Kết quả và đánh giá

- **Đã đạt được:** thu được CSI cần thiết bằng cách giải mã đúng cấu trúc frame mà firmware Espectre gửi về (prefix HTTP 60 byte + header V8 64 byte).
- **Chưa đạt / hạn chế:** tần số lấy mẫu chưa đáp ứng nhu cầu. Hiện tượng ghi nhận: Packet Per Second đặt là 100 nhưng Occupancy chỉ vào khoảng 80, tức là mất khoảng 20 mẫu mỗi giây.

## 5. Việc làm tiếp theo

- [ ] Kiểm tra xem PPS bao nhiêu là tối ưu.
- [ ] Khảo sát các lí do có thể gây ra mất mẫu.
- [ ] Kiểm tra specs phần cứng so với các bài báo / các dự án tương tự.
- [ ] Đo thêm mẫu để đánh giá.

## 6. Các link, nguồn tài liệu tham khảo

- Repo dự án: [https://github.com/ltkdt?tab=repositories](https://github.com/ltkdt?tab=repositories)
- Repo code Espectre: [https://github.com/francescopace/espectre](https://github.com/francescopace/espectre)
- Tool flash của Espectre: [https://espectre.dev/tools/flash/](https://espectre.dev/tools/flash/)
- Tải Rust nếu chạy lại dự án: [https://rust-lang.org/tools/install/](https://rust-lang.org/tools/install/)
