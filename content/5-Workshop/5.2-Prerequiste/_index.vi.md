---
title: "Điều kiện tiên quyết"
date: 2026-01-09T16:03:43+07:00
weight: 90
chapter: false
pre: " <b> 5.2. </b> "
---

# 5.2 Điều kiện tiên quyết

Trang này là một checklist. Đừng bỏ qua. Bỏ sót một mục — đặc biệt là quyền truy cập Bedrock — có thể khiến cả workshop dừng 24 giờ chờ AWS phê duyệt.

## Yêu cầu tài khoản AWS

### 1. Tài khoản AWS còn hoạt động, có quyền admin

Bạn cần một tài khoản AWS có thể:

- Tạo IAM user và role.
- Tạo bảng DynamoDB, Lambda function, AppSync API, S3 bucket, Cognito user pool, ECS cluster.
- Đăng ký truy cập model trên Bedrock.

Cho workshop này, dùng tài khoản root hoặc một IAM user có managed policy `AdministratorAccess`. **Không** dùng tài khoản read-only hay sandbox hạn chế — nhiều bước sẽ lỗi do permission boundary.

### 2. Truy cập model Bedrock — ĐĂNG KÝ TRƯỚC

> **QUAN TRỌNG — đăng ký từ trước.** Các foundation model trên Amazon Bedrock yêu cầu kích hoạt. Quyền truy cập `qwen.qwen3-vl-235b-a22b` tại `ap-southeast-2` thường được cấp trong vài phút nhưng có thể mất đến 24 giờ. Gửi yêu cầu **trước** khi bước vào phần 5.5, lý tưởng là từ ngày hôm trước.

Các bước:

1. Mở AWS Console và chuyển region sang **Asia Pacific (Sydney) — ap-southeast-2**.
2. Vào **Amazon Bedrock → Model access**.
3. Nhấn **Modify model access**.
4. Bật **Qwen 3 VL 235B A22B** (`qwen.qwen3-vl-235b-a22b`).
5. Submit và chờ trạng thái chuyển sang **Access granted**.

![Bedrock cap quyen cho Qwen3-VL](//images/workshop/workshop/bedrock-model-access.png)

IAM policy gắn vào Lambda `ai-engine` trong `backend/amplify/backend.ts` chỉ cấp `bedrock:InvokeModel` cho đúng ARN model này:

```text
arn:aws:bedrock:ap-southeast-2::foundation-model/qwen.qwen3-vl-235b-a22b
```

Nếu bạn đổi region hoặc model, bạn cũng phải sửa `backend.ts` và `ai-engine/handler.ts`.

### 3. Cảnh báo AWS Budgets

Trước khi chạy bất kỳ lệnh nào, tạo một budget để Bedrock không làm bạn bất ngờ:

- **AWS Console → Billing → Budgets → Create budget**.
- Cost budget, hằng tháng, ngưỡng **$25 USD**, cảnh báo ở 50%, 80%, 100%.

## Công cụ local

Cài các công cụ sau lên máy. Các phiên bản liệt kê là mức tối thiểu đã test với codebase.

| Công cụ                                          | Phiên bản tối thiểu | Dùng cho                                        |
| ------------------------------------------------ | ------------------- | ----------------------------------------------- |
| Node.js                                          | **20 LTS** trở lên  | Chạy `ampx` CLI, build Lambda, Expo             |
| npm                                              | **10+**             | Cài package (đi kèm Node 20+)                   |
| AWS CLI                                          | **v2**              | Cấu hình credential, thao tác CloudFormation/S3 |
| Git                                              | **2.40+**           | Clone template, push branch cho CI              |
| Docker Desktop                                   | **bản stable mới**  | Build image FastAPI cho ECS                     |
| Expo Go (điện thoại) hoặc Android Studio / Xcode | mới nhất            | Chạy mobile client                              |

### Kiểm tra phiên bản

```bash
node --version
npm --version
```

```bash
aws --version
docker --version
```

```bash
git --version
```

Cả năm lệnh phải in ra kết quả không lỗi. Node phải báo **v20.x trở lên**.

### AWS CLI profile

Cấu hình một profile để Amplify CLI và các lệnh `aws` trong phần cleanup dùng:

```bash
aws configure --profile nutritrack
aws sts get-caller-identity --profile nutritrack
```

Export `AWS_PROFILE=nutritrack` trong shell mà bạn chạy `npx ampx sandbox`, hoặc set trong shell profile.

## Google Cloud Console — OAuth Client

Cognito federate sang Google cho đăng nhập xã hội. Bạn cần một OAuth 2.0 Web client của Google.

1. Mở [Google Cloud Console](https://console.cloud.google.com/) và tạo (hoặc dùng lại) một project.
2. Vào **APIs & Services → OAuth consent screen**. Cấu hình consent screen **External** với email của bạn làm test user.
3. Vào **APIs & Services → Credentials → Create credentials → OAuth client ID**.
4. Application type: **Web application**.
5. Authorized redirect URIs: bạn sẽ thêm URL callback của Cognito Hosted UI sau khi phần 5.3 tạo user pool. Tạm thời để trống — bạn sẽ quay lại.
6. Copy lại **Client ID** và **Client secret**.

![Google OAuth Web client credentials](//images/workshop/workshop/google-oauth-client.png)

Sau này, lưu chúng thành Amplify sandbox secret (chạy trong `backend/`):

```bash
cd backend
npx ampx sandbox secret set GOOGLE_CLIENT_ID
npx ampx sandbox secret set GOOGLE_CLIENT_SECRET
```

Tên secret `GOOGLE_CLIENT_ID` và `GOOGLE_CLIENT_SECRET` được tham chiếu trong `backend/amplify/auth/resource.ts` — không đổi tên.

## Kiến thức tiên quyết

Bạn không cần là chuyên gia ở các lĩnh vực dưới đây, nhưng nên đọc và chỉnh code được mà không phải tra cú pháp cơ bản.

- **TypeScript** — toàn bộ backend và frontend là TypeScript. Bạn nên nắm type, generic cơ bản, và `async/await`.
- **Thao tác AWS Console** — biết đổi region, tìm service, đọc CloudWatch log.
- **GraphQL (nên có)** — Amplify Data sinh schema GraphQL. Bạn không cần viết resolver, nhưng đọc hiểu query sẽ thuận lợi hơn.
- **React Native (tùy chọn)** — app Expo chạy nguyên trạng. Có kinh nghiệm giúp bạn tùy chỉnh màn hình dễ hơn.

## Cảnh báo chi phí

Bedrock tính tiền theo input và output token. Chạy full workshop, bao gồm vài chục lần gọi AI để smoke test, tốn khoảng **$1 đến $5 USD** tiền Bedrock. ECS Fargate tính theo giờ lẻ khi cluster đang chạy. Những thứ còn lại (DynamoDB on-demand, AppSync, Lambda, S3) nằm gọn trong free tier cho một ngày dev nhẹ.

Nếu bạn dừng ở phần 5.10 Cleanup trong cùng ngày, tổng hóa đơn AWS thường dưới **$10 USD**. Nếu để tài nguyên chạy cả tháng, dự kiến **$50 đến $150 USD** tùy mức độ sử dụng AI.

## Sẵn sàng?

Khi mọi mục phía trên đã xong — Bedrock đã được cấp quyền, CLI đã cài, OAuth client của Google đã tạo, budget alert đã bật — tiếp tục sang [5.3 Foundation Setup](../5.3-Foundation-Setup/).
