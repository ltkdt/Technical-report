# Calibration của ESPectre: khả năng tái sử dụng cho rt-wireless-sensing và ảnh hưởng đến việc lấy mẫu dữ liệu

**Người thực hiện:** Kiên · **Ngày thực hiện:** 18/11/2026

## 1. Vấn đề và mục tiêu giải quyết

Ứng dụng rt-wireless-sensing (xem báo cáo `171126_rt_wireless_sensing_0.1.0.md`) thu CSI WiFi thô từ ESP32-C5 chạy firmware ESPectre, với mục tiêu cuối cùng là xây dựng pipeline AI riêng cho indoor localization và human activity recognition. Hai câu hỏi cần câu trả lời dứt khoát:

1. Scheme calibration của ESPectre có thể tái sử dụng cho project này không, xét vì mục đích của nó là motion detection chứ không phải localization/HAR?
2. Calibration mode có ảnh hưởng đến việc lấy mẫu dữ liệu không? Cụ thể: khi collection cấu hình 100 pps nhưng capture giải mã trung bình chỉ \~80 bản ghi mỗi giây, calibration có phải là nguyên nhân?

Một câu hỏi thứ ba, hẹp hơn, cũng được trả lời song song: bằng cơ chế nào mode High Accuracy nhận threshold đã train offline, và tại sao cơ chế đó tiêu tốn tài nguyên thiết bị nhiều hơn Lightweight?

## 2. Các yêu cầu cần thiết trước

- Repo `espectre` (github.com/francescopace/espectre): docs `API.md`, `ALGORITHMS.md`, cộng với báo cáo đối chiếu trước đó `261117_calibration_mode_clarification.md` (ánh xạ docs-to-source của `calibrating`, `ready`, và threshold calibrator).
- Repo `RT_Wireless_Sensing` (github.com/ltkdt/RT\_Wireless\_Sensing): `src/collection.rs`, `src/visualization.rs`, `docs/visualization.md`, `Memory.md`, `plan.md`.
- Setup đang chạy: ESP32-C5 với firmware ESPectre (profile mặc định: Lightweight, Direct HTTP bật), desktop client chạy `cargo run --bin collect` cùng mạng.
- Hiện tượng ghi nhận: streaming dashboard báo effective PPS thấp hơn `--pps 100` đã cấu hình trong đa số các lần chạy.

## 3. Nội dung đã thực hiện

Đọc toàn diện docs ESPectre, đối chiếu với source RT\_Wireless\_Sensing, tách ba luồng dữ liệu mà ESP32 tạo ra, rồi lần theo xem calibration chạm đến luồng nào.

### 3.1. Ba luồng dữ liệu do một ESP32 RX tạo ra


| #   | Luồng                                            | Nguồn                                                                                                                                                                                   | rt-wireless-sensing có dùng không?                     |
| --- | ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| 1   | Sự kiện motion/idle                              | Output của detector, publish qua `GET /espectre/v1/events` (SSE) hoặc MQTT. Lưu ý: endpoint `sensing` mang state (`ready`, `calibrating`, `threshold`), không phải bản thân các sự kiện | Không                                                  |
| 2   | Payload CSI thô (amplitude/phase, 64 subcarrier) | Stream `GET /espectre/v1/csi`                                                                                                                                                           | Có, giải mã bởi `src/decoder.rs`, xuất bởi `saving.rs` |
| 3   | Metadata prefix HTTP + header bản ghi V8         | Cùng stream đó, parse bởi `csi_header.rs`                                                                                                                                               | Có                                                     |


### 3.2. Calibration thực sự là gì, và chạy ở đâu

Calibration chạy hoàn toàn trên ESP32, bên trong runtime C++ của firmware (`EspIdfRuntime::start_calibration_()`, `LightweightDetector`, `ThresholdCalibrator`). Desktop client chạy `collect` chỉ là consumer thuần: mở CSI stream và ghi bytes xuống đĩa; nó không bao giờ thực thi hay ảnh hưởng calibrator. Phần calibration phía client duy nhất trong repo ESPectre là bản mirror replay/debug trong Python CLI, không chạy trong lúc collection.

Startup calibration phục vụ đúng một mục đích: Lightweight không có threshold đã train, nên phải học threshold từ chính căn phòng đó. Nó thu các đánh giá của detector trong \~10 s phòng im, tùy chọn bỏ cửa sổ 5 s nhiễu nhất (kéo dài tối đa \~30 s), và kẹp kết quả trong \[0.095, 0.987\]. Motion event bị suppress trong lúc học chỉ vì readiness gate đặt `ready_to_publish=false` khi threshold chưa hợp lệ. Đó là toàn bộ cơ chế.

Cốt lõi: không có gì trong đường calibration sửa đổi luồng loại 2 và 3. Firmware vẫn capture CSI, vẫn chạy traffic generator, vẫn stream bản ghi thô; calibrator đọc các gói tin qua một interceptor trên thiết bị và chỉ suppress chính các sự kiện mà nó publish.

### 3.3. Kết luận về khả năng tái sử dụng cho localization/HAR

Scheme calibration không thể tái sử dụng, vì nó giải một vấn đề mà project này không có (threshold nhị phân cho motion). Thứ project cần thay vào đó, và "calibration" nên được hiểu là gì trong rt-wireless-sensing, gồm ba nhóm:

1. Làm sạch tín hiệu: phase unwrapping và hiệu chỉnh CFO/SFO (phase thô mỗi gói bị ô nhiễm bởi carrier và sampling frequency offset), chuẩn hóa amplitude theo AGC, làm mượt. Đây là DSP trên từng gói trước khi trích xuất feature.
2. Baseline theo môi trường: capture fingerprint từng vị trí từng phòng cho localization; fine-tuning theo môi trường hoặc domain adaptation cho HAR, vì CSI mang tính phòng-cụ thể. Đây là protocol thu dữ liệu, không phải runtime mode.
3. Vệ sinh dataset: kịch bản hoạt động cố định, ghi lại trạng thái thiết bị lúc bắt đầu session, thời điểm bắt đầu đã biết. Một lần `GET /espectre/v1/sensing` đầu session, ghi vào file sidecar (`calibrating`, `threshold`, `detector`, wall-clock start), bao phần trạng thái thiết bị với chi phí gần như bằng không.

Hệ quả với dữ liệu quá khứ: mọi capture trước đây đều chạy với profile Lightweight mặc định, nên bất kỳ capture nào bắt đầu ngay sau boot đều chứa một cửa sổ calibration 10–30 s không được đánh dấu. Điều này không làm hỏng chính các mẫu CSI; thiệt hại thực tế nằm ở nhãn dữ liệu (những giây đó là dữ liệu phòng im, bất kể kịch bản hoạt động giả định gì) và ở việc không có metadata threshold theo từng lần chạy. Capture cũ không thể sửa chữa; chỉ có thể phân đoạn heuristic trên chúng.

### 3.4. High Accuracy: threshold đã train đến thiết bị bằng cách nào, và tại sao tốn tài nguyên hơn

Threshold và model của High Accuracy được tạo offline, bằng cách train trên corpus thu từ nhiều môi trường. Quá trình training cho ra hai artifact: (a) một mạng neural nhỏ trên tám feature bất biến theo scale, và (b) một decision threshold cố định. Cả hai được compile vào firmware image lúc build: ML code và các trọng số đã train nằm trong flash cùng phần còn lại của firmware. Không có truyền tải lúc runtime: không có gì được download hay học trên thiết bị, và đây chính là lý do không cần calibration.

Chi phí tài nguyên vì thế không nằm ở khâu chuyển giao (lưu trong flash là rẻ và bị động) mà ở những gì thiết bị làm mỗi chu kỳ đánh giá (mỗi 250 ms theo production setting):

- Trạng thái feature: High Accuracy duy trì profile phức 56-bin live, tracking dịch chuyển ML L1 với độ trễ 100 ms, và các trajectory tracker, trên đường turbulence 12-subcarrier dùng chung với Lightweight.
- Suy luận: mỗi đánh giá chạy một forward pass qua mạng, vector-matrix multiply và activation trên tám feature, đối lập với Lightweight là hai feature, trọng số cố định, một phép tổng có trọng số, và một phép so sánh với threshold vô hướng.
- Bộ nhớ: trọng số mạng, cửa sổ feature lớn hơn, và các buffer profile chiếm RAM hoạt động mà Lightweight để trống.

Vậy chuỗi nhân quả là: train offline → nhúng vào firmware → chạy suy luận mỗi đánh giá → footprint CPU/RAM cao hơn. Việc không có calibration là hệ quả của việc đã được train, không phải của việc đắt đỏ; hai tính chất này liên quan nhưng độc lập.

### 3.5. Quy gán mất mẫu: calibration được miễn nhiệm

`--pps 100` cấu hình là mục tiêu cho tốc độ UDP marker, không phải cam kết số bản ghi CSI nhận được. Mỗi marker đi desktop → AP → ESP32, và mỗi frame nhận được cho tối đa một bản ghi CSI. Mất gói có thể xảy ra ở từng khâu:


| Khâu                | Cơ chế                                                                                                                                                                                                                                                           |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Generator (desktop) | Jitter timer của OS và các stall lập lịch trong send loop; power-save Wi-Fi trên NIC desktop làm buffer và dồn gói; traffic cạnh tranh từ chính radio đó, kể cả luồng tải CSI xuống                                                                              |
| AP / router         | Airtime contention: marker hai chiều cộng luồng upload CSI chia sẻ một kênh, pipeline tự tắc nghẽn ở tốc độ cao; nhiễu từ mạng lân cận; power-save buffering cho ESP32 nếu modem sleep bật; marker multicast (nếu cấu hình) không được retransmission ở tầng MAC |
| RX device (ESP32)   | MAC retransmission fail và RSSI yếu làm rớt frame trước khi trích CSI; filter của tầng capture (marker phải trỏ đúng MAC thiết bị); tải CPU thiết bị: trích CSI, serve HTTP CSI, parse UDP, và đánh giá detector 250 ms chạy đồng thời                           |


Đóng góp của calibration vào việc mất gói là biên chế: nó chỉ thêm một interceptor mỗi gói trong cửa sổ 10–30 s sau boot hoặc sau lần đổi cấu hình, rồi biến mất. Nếu thiếu hụt đều xuyên suốt capture thay vì dồn ở đầu, calibration hoàn toàn được miễn nhiệm.

Thủ tục quy gán: đếm số marker thực sự gửi trong sender loop (không phải tốc độ cấu hình), so với số bản ghi CSI giải mã mỗi giây. Nếu desktop gửi 100 và CSI ghi 80, mất gói nằm phía không dây (AP/RX). Nếu chính desktop chỉ gửi 85, generator có lỗi. Occupancy click-to-compute của dashboard đã cho số phía RX; thêm bộ đếm gói đã gửi vào `src/collection.rs` sẽ khép kín bức tranh.

## 4. Kết quả và đánh giá

- **Đã đạt được:** tách dứt khoát ba luồng dữ liệu và chứng minh calibration của ESPectre chỉ chạm luồng sự kiện motion; kết luận tái sử dụng (không tái sử dụng scheme threshold; thay bằng làm sạch tín hiệu, baseline môi trường, và protocol vệ sinh dataset); cơ chế High Accuracy nhận threshold đã train (compile vào flash lúc build, suy luận mỗi đánh giá 250 ms) và lý do nó tốn tài nguyên hơn Lightweight; và quy gán ba khâu của thâm hụt 80/100 pps, miễn nhiệm cho calibration.
- **Chưa đạt / hạn chế:** bảng quy gán mất gói được suy luận từ docs và triệu chứng effective-PPS quan sát được, chưa đo đạc: chưa có bộ đếm gói đã gửi trong `collection.rs`, nên mất mát phía generator chưa được định lượng. Trạng thái power-save Wi-Fi của firmware trên build ESP32-C5 chưa được kiểm chứng. Các capture quá khứ vẫn chưa phân đoạn và không thể sửa. Việc ghi sidecar một poll mỗi lần chạy chưa được implement.

## 5. Việc làm tiếp theo / Action items

- [ ] Thêm bộ đếm marker đã gửi vào `collection.rs`, so sánh gửi vs. giải mã mỗi giây để định lượng thâm hụt \~20 pps xuất phát từ đâu.
- [ ] Thêm một lần đọc `GET /espectre/v1/sensing` đầu session, ghi vào file sidecar kèm `calibrating`, `threshold`, `detector`, và wall-clock start, làm metadata theo từng lần chạy.
- [ ] Kiểm chứng build ESP32-C5 có chạy Wi-Fi modem sleep không; nếu có, đánh giá xem AP buffering giải thích được phần nào thâm hụt gói.
- [ ] Cho pipeline AI: implement làm sạch phase (hiệu chỉnh CFO/SFO) và chuẩn hóa amplitude làm bước "calibration" riêng của project, thay cho ý niệm tái sử dụng scheme threshold của ESPectre.
- [ ] Cho capture tương lai: giữ kịch bản hoạt động cố định với khoảng im lặng chủ ý trước hoạt động đầu tiên, để mọi nhập nhằng còn lại ở lúc boot bị chặn trong biên.

## 6. Các link, nguồn tài liệu tham khảo

- ESPectre docs: `docs/API.md` (sensing resource, events, support matrix), `docs/ALGORITHMS.md` (hai profile, processing pipeline, detector timing, temporal admission, settled-level recovery), repo `francescopace/espectre`.
- Báo cáo trước: `261117_calibration_mode_clarification.md` (ánh xạ docs-to-source của trạng thái calibration).
- RT\_Wireless\_Sensing: `src/collection.rs`, `src/visualization.rs`, `src/saving.rs`, `docs/visualization.md` (tính effective PPS từ device clock deltas), repo `ltkdt/RT_Wireless_Sensing`.