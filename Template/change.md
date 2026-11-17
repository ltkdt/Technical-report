Here's a summary of the changes you asked for, for your local agent skill:

**1. File naming convention**
- Report files in `Report/` follow `YYMMDD_title.md` (e.g. `171126_rt_wireless_sensing_0.1.0.md`), one folder per project (`Papers/`, `Report/`, `Template/` inside each project folder).

**2. Editing an existing report (user names the file, two modes; ask if unspecified)**

- **Edit**: modify the report directly per the request, keep everything else unchanged. No changelog entry.
- **Update**: apply new content, and update the Changelog table placed right below the title (after the author/date line) and before Section 1:

```markdown
**Changelog**

| # | Thời gian | Thay đổi |
|---|---|---|
| 1 | HH:MM DD/MM/YY | Mô tả thay đổi làm thay đổi kết quả báo cáo |
```

- Each subsequent update appends one row with `#` increasing (2, 3, ...) in chronological order. One changelog table per report, never a duplicate block.

**3. Report format (unchanged core structure)**
- Title line: `# {{Tiêu đề}}` then `**Người thực hiện:** {{name}} · **Ngày thực hiện:** {{DD/MM/YYYY}}`, then Changelog (if any), then the 6 fixed sections: Vấn đề và mục tiêu, Yêu cầu cần thiết trước, Nội dung đã thực hiện, Kết quả và đánh giá, Việc làm tiếp theo / Action items, Link tham khảo.
