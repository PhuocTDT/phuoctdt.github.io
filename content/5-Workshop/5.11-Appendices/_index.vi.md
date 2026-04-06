---
title: "Appendices"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.11. </b> "
---

# Phụ lục

Phần này tập hợp các tài liệu tham khảo đi kèm workshop. Nó được tách riêng khỏi các bước hướng dẫn thực hành có chủ đích: phần walkthrough cho bạn biết *bấm cái gì*, còn phần phụ lục giải thích *tại sao chọn giá trị đó*, *nó nằm ở đâu trong source*, và *xử lý ra sao khi gặp sự cố*.

Mọi phụ lục đều có thể truy ngược về một file trong `backend/amplify/`. Nếu bạn thấy trang phụ lục mâu thuẫn với code, **code là nguồn sự thật** — hãy mở issue và chúng tôi sẽ đồng bộ lại.

## Nội dung

- [5.11.1 Budget Breakdown](5.11.1-Budget-Breakdown/) — Bảng dự toán chi phí hàng tháng chi tiết cho triển khai 1.000 người dùng, chia theo từng dịch vụ AWS, có cấn trừ free tier và ghi chú khi scale. *(Nội dung được duy trì riêng — đang chờ export CSV từ worksheet pricing.)*
- [5.11.2 IAM Policies](5.11.2-IAM-Policies/) — Toàn bộ JSON policy cho từng Lambda execution role và bucket resource policy trên S3, trích nguyên văn từ `backend.ts`.
- [5.11.3 Troubleshooting](5.11.3-Troubleshooting/) — Runbook phân loại theo nhóm: triệu chứng lỗi thường gặp, nguyên nhân gốc, và chính xác lệnh hoặc file cần sửa.
- [5.11.4 Prompt Templates](5.11.4-Prompt-Templates/) — Các system prompt thực tế đang chạy trong `ai-engine/handler.ts`, mỗi prompt ứng với một action, kèm input variables và output schema.

## Cách đọc các trang này

- Mọi code block đều được gắn tag ngôn ngữ; copy trực tiếp là chạy được.
- Đường dẫn file luôn tính từ gốc repository (ví dụ: `backend/amplify/backend.ts`).
- Khi một policy hoặc prompt chứa region, model ID, hoặc table prefix, giá trị đó khớp với default của workshop — chỉ thay đổi nếu bạn hiểu rõ lý do.
- "Ollie" là tên persona AI coach. Không đổi tên này trong prompt; các string literal ở UI và test phụ thuộc vào nó.

## Các invariant dùng xuyên suốt phụ lục

| Invariant | Giá trị |
|---|---|
| Lambda runtime | Node.js 22 trên ARM64 |
| Bedrock model | `qwen.qwen3-vl-235b-a22b` |
| Bedrock region | `ap-southeast-2` |
| Lambda functions | `aiEngine`, `processNutrition`, `friendRequest`, `resizeImage` |
| Coach persona | Ollie |
| S3 prefixes | `incoming/`, `voice/`, `media/` |

Nếu bất kỳ giá trị nào ở trên bị khác đi trong fork của bạn, hãy cập nhật phụ lục tương ứng trước khi ship.
