# Technical Reports

Repo này chứa các báo cáo kỹ thuật, viết bằng Markdown để người và AI đều đọc trực tiếp được.

## Cấu trúc

- Mỗi project có 2 folder riêng (ví dụ: `CSI/`).
- Trong mỗi project có 3 thư mục:
  - **`Papers/`**: tổng hợp nghiên cứu từ các bài báo khác, tên file theo chủ đề bài báo.
  - **`Report/`**: các technical report, tên file chính là ngày viết (`YYYYMMDD.md` hoặc `YYMMDD.md`).

```text
report/
├── CSI/
│   ├── Papers/
│   │   ├── csi-bench-wifi-csi-sensing-and-hardware-brief.md
│   │   └── ...csi-sensing-signal-processing-methods-survey-section-3.md
│   ├── Report/
│   │   └── 171126.md
│   └── Template/
│       └── Mau bao cao.md
└── README.md
```

## Quy ước

- Báo cáo theo template chung 6 mục: Vấn đề và mục tiêu, Yêu cầu cần thiết trước, Nội dung đã thực hiện, Kết quả và đánh giá, Việc làm tiếp theo, Link tham khảo.
- File trong `Papers/` ghi rõ nguồn (link bài báo, phần/section đã tổng hợp) để tra cứu lại.
- Mỗi thay đổi lớn: commit riêng, message theo format `report(<ten-bao-cao>): mô tả ngắn`.
- Link nội bộ dùng đường dẫn tương đối chuẩn Markdown để đọc tốt cả trên Obsidian lẫn GitHub.
