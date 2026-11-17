# Template báo cáo kỹ thuật

> Cách dùng: lưu file này vào `Report/templates/` trong vault. Khi viết báo cáo, copy file, đổi tên theo quy ước `YYYY-MM-DD_ten-bao-cao.md`, điền các mục và xóa các hướng dẫn trong khối `>`.

```yaml
---
title: "Tên báo cáo"
author: "Họ tên"
date: YYYY-MM-DD
project: "Tên dự án"
status: draft   # draft | review | final
tags: [bao-cao]
---
```

---

# {{title}}

**Người thực hiện:** {{author}} · **Ngày thực hiện:** {{date}}

## 1. Vấn đề và mục tiêu giải quyết

> Vấn đề là gì, tại sao cần giải quyết, mục tiêu đạt được là gì.

## 2. Các yêu cầu cần thiết trước

> Điều kiện cần có trước khi thực hiện: mã nguồn repo, phần cứng, công cụ, môi trường...

- ...

## 3. Nội dung đã thực hiện

> Tổng hợp các module / hạng mục đã làm và nội dung chính của từng mục. Nên nêu các lệnh chính để build và chạy. Có thể dùng bảng hoặc sơ đồ Mermaid.


| Module / Hạng mục | Chức năng chính | Lệnh / Cách chạy |
| ----------------- | --------------- | ---------------- |
|                   |                 |                  |


## 4. Kết quả và đánh giá

- Đã đạt được:
- Chưa đạt / hạn chế (kèm hiện tượng, số liệu ghi nhận):

## 5. Việc làm tiếp theo / Action items

- [ ] ...

## 6. Các link, nguồn tài liệu tham khảo

- ...