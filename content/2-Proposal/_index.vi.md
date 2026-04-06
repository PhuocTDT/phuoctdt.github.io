---
title: "Bản đề xuất"
date: 2026-01-09T15:00:11+07:00
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Nền tảng NutriTrack

## Giải pháp AWS Serverless tích hợp AI cho Theo dõi Dinh dưỡng

---

### 1. Tóm tắt Dự án

NutriTrack được xây dựng cho người dùng Gen Z và Millennials tại Việt Nam — những người muốn kiểm soát sức khỏe cá nhân một cách thực chất, không phức tạp. Nền tảng nhắm đến quy mô 1.000+ người dùng hoạt động, với giao diện di động đa nền tảng phát triển bằng React Native (Expo), cho phép ghi nhận dữ liệu bữa ăn qua ảnh chụp, giọng nói hoặc nhập tay.

Toàn bộ hạ tầng tuân thủ AWS Well-Architected Framework, khai thác hệ sinh thái serverless (AWS Amplify Gen 2, AppSync, Lambda) và năng lực Generative AI từ Amazon Bedrock (model Qwen3-VL 235B). Kết quả là một hệ thống theo dõi dinh dưỡng thời gian thực, có AI dự đoán, cơ chế gamification phong phú, và chi phí vận hành tối ưu — tất cả được bảo vệ bởi Amazon Cognito kết hợp Google OAuth2.

---

### 2. Bối cảnh và Giải pháp

#### Vấn đề hiện tại

Hầu hết ứng dụng theo dõi dinh dưỡng hiện nay đều yêu cầu người dùng nhập liệu thủ công — tẻ nhạt và dễ bỏ cuộc. Cơ sở dữ liệu món ăn thường nghèo nàn về ẩm thực Việt Nam, thiếu hẳn những món quen thuộc như phở, bánh mì, hay bún bò Huế. Chưa kể, các API dinh dưỡng của bên thứ ba (Nutritionix, FatSecret) tốn kém khi mở rộng và bị giới hạn về mô hình dữ liệu.

Quản lý tủ lạnh, lên thực đơn hàng ngày hay tận dụng nguyên liệu sẵn có cũng là bài toán chưa được giải quyết tốt — dẫn đến lãng phí thực phẩm và thói quen ăn uống thiếu nhất quán.

#### NutriTrack giải quyết thế nào?

NutriTrack tiếp cận bằng kiến trúc AWS-native hoàn toàn serverless:

- **AWS AppSync (GraphQL)** tiếp nhận dữ liệu bữa ăn qua mutations và real-time subscriptions.
- **AWS Lambda** với 4 hàm chuyên biệt xử lý mọi logic backend — từ điều phối AI đến quản lý kết bạn đến tối ưu ảnh.
- **Amazon DynamoDB** lưu trữ 8 mô hình dữ liệu với khả năng tự co giãn theo tải.
- **Amazon Bedrock (Qwen3-VL 235B)** phân tích ảnh đồ ăn, quét mã vạch/nhãn dinh dưỡng, và sinh dữ liệu dinh dưỡng cho các món chưa có trong cơ sở dữ liệu.
- **AWS Amplify Gen 2** kết hợp React Native/Expo tạo ra giao diện song ngữ Việt–Anh mượt mà.

**Tính năng nổi bật:**

| Tính năng | Mô tả |
|-----------|-------|
| 📸 Phân tích ảnh đồ ăn | Chụp một tấm — nhận ngay bảng dinh dưỡng chi tiết |
| 🎙️ Ghi nhật ký bằng giọng nói | AWS Transcribe chuyển lời nói thành dữ liệu bữa ăn |
| 🍳 Tủ lạnh thông minh | Theo dõi thực phẩm dự trữ, gợi ý công thức từ nguyên liệu có sẵn |
| 🤖 AI Coach "Ollie" | Tư vấn dinh dưỡng cá nhân hóa và phân tích hàng tuần |
| 🎮 Gamification | Chuỗi ngày liên tục, thú cưng ảo tiến hóa, thử thách cộng đồng |
| 👥 Tính năng xã hội | Hệ thống kết bạn, bảng xếp hạng, thử thách đối kháng |

#### Hiệu quả chi phí

Chi phí vận hành ước tính **~$60.87/tháng** cho 1.000 người dùng hoạt động — nhờ tận dụng AWS Free Tier và mô hình thanh toán theo mức dùng của serverless. Mốc hòa vốn phụ thuộc vào tỉ lệ chuyển đổi sang gói premium (phân tích ảnh không giới hạn, báo cáo AI chuyên sâu, coach ưu tiên) và sẽ được xác nhận sau khi ra mắt.

---

### 3. Kiến trúc Giải pháp

Dữ liệu di chuyển từ ứng dụng React Native → CloudFront/WAF (bảo mật biên) → AppSync GraphQL API → các Lambda handler → DynamoDB, được làm giàu thêm bởi AI Bedrock. Toàn bộ hạ tầng được điều phối qua AWS Amplify Gen 2 với CI/CD tự động trên 3 môi trường triển khai.

#### Các dịch vụ AWS

| Dịch vụ | Vai trò |
|---------|---------|
| **AWS Amplify Gen 2** | Điều phối hạ tầng bằng TypeScript CDK, quản lý pipeline CI/CD và 3 môi trường (Sandbox, feat/phase3, main) |
| **AWS AppSync** | GraphQL API bảo mật với real-time subscriptions giữa mobile app và Lambda resolvers |
| **AWS Lambda** | 4 hàm xử lý logic: `ai-engine`, `process-nutrition`, `friend-request`, `resize-image` |
| **Amazon DynamoDB** | 8 bảng NoSQL: `user`, `Food` (~200 món Việt), `FoodLog`, `FridgeItem`, `Challenge`, `ChallengeParticipant`, `Friendship`, `UserPublicStats` |
| **Amazon Bedrock** | Qwen3-VL 235B (ap-southeast-2) cho phân tích ảnh, quét mã vạch, đọc nhãn, gợi ý công thức, AI coaching |
| **AWS Transcribe** | Chuyển bản ghi âm (S3 prefix `/voice`) thành văn bản cho tính năng ghi nhật ký bằng giọng nói |
| **Amazon S3** | Lưu media theo 4 prefix: `incoming/` (xóa sau 1 ngày), `voice/`, `avatar/`, `media/` |
| **Amazon Cognito** | Xác thực email/OTP và Google OAuth2, quản lý JWT tokens và user pools |
| **Amazon CloudFront** | CDN tăng tốc phân phối nội dung toàn cầu |
| **AWS WAF** | Tường lửa bảo vệ API khỏi tấn công phổ biến và DDoS |
| **Route 53** | Phân giải DNS cho domain ứng dụng |
| **Amazon ECR** | Kho chứa Docker images cho ECS Fargate |
| **Amazon ECS Fargate** | Container serverless cho backend FastAPI trong VPC (private subnets, 2 AZ, ALB, Auto Scaling) |
| **AWS CloudWatch** | Giám sát với 4 metrics tùy chỉnh: `Bedrock_AI_Error_Rate`, `Image_Processing_Time`, `User_Daily_Active`, `Food_Log_Count` |
| **AWS CloudTrail** | Ghi log API cho kiểm toán và tuân thủ |
| **AWS KMS** | Mã hóa dữ liệu nhạy cảm khi lưu trữ |
| **AWS Secrets Manager** | Bảo mật API keys, JWT master key và cấu hình ứng dụng |
| **AWS Textract** | OCR trích xuất văn bản từ nhãn dinh dưỡng |

#### Thiết kế theo tầng

**Client** — Ứng dụng React Native/Expo thu thập đầu vào qua Camera, Microphone và form nhập tay. Dữ liệu được trực quan hóa bằng biểu đồ tương tác và giao diện gamification với thú cưng ảo tiến hóa theo chuỗi ngày liên tục.

**Bảo mật biên** — Route 53 → WAF → CloudFront xử lý phân giải DNS, chặn DDoS, và tăng tốc nội dung trước khi request chạm đến backend.

**Xác thực** — Cognito quản lý đăng ký (email/OTP), đăng nhập Google OAuth2, và JWT lifecycle. Ứng dụng bổ sung lớp bảo vệ sinh trắc học cục bộ (FaceID/TouchID).

**Xử lý** — 4 Lambda handler đảm nhận toàn bộ logic:

- `aiEngine` — Điều phối 10 tác vụ Bedrock: `analyzeFoodImage`, `voiceToFood`, `generateRecipe`, `generateCoachResponse`, `searchFoodNutrition`, `fixFood`, `ollieCoachTip`, `calculateMacros`, `challengeSummary`, `weeklyInsight`
- `processNutrition` — Tra cứu lai ghép: fuzzy match trên DynamoDB (~200 món Việt) trước, Bedrock AI làm dự phòng nếu không tìm thấy
- `friendRequest` — Xử lý xã hội (gửi/chấp nhận/từ chối/chặn) qua DynamoDB `TransactWriteItems`
- `resizeImage` — S3 event trigger trên `incoming/`, dùng `sharp` để scale cạnh dài nhất về 1280px (giữ tỉ lệ, tự xoay theo EXIF), encode thành progressive JPEG chất lượng 85, ghi vào `media/{entity_id}/`

**Dữ liệu** — 8 bảng DynamoDB với phân quyền theo chủ sở hữu (owner-scoped) và GSI được tối ưu cho các pattern truy vấn phổ biến.

**AI/ML** — Bedrock (Qwen3-VL 235B) và Transcribe tạo thành lớp trí tuệ cốt lõi, tất cả đều được định tuyến qua `aiEngine` với prompt templates có cấu trúc cố định.

**Container** — ECS Fargate chạy backend FastAPI trong VPC (2 AZ tại ap-southeast-2a), cân bằng tải qua ALB, kết nối dịch vụ AWS qua VPC Endpoints.

---

### 4. Triển khai Kỹ thuật

#### Công nghệ sử dụng

- **Frontend:** React Native, Expo, TypeScript, Zustand, react-native-reanimated, expo-router, i18n (Việt/Anh)
- **Backend:** Amplify Gen 2 (TypeScript CDK), Lambda (Node.js 22), AppSync, DynamoDB (8 bảng + GSI), Cognito (User Pools + Identity Pools + Google federation), S3 (4 prefix + lifecycle rule)
- **AI:** Bedrock API (Qwen3-VL 235B, ap-southeast-2), Transcribe (async jobs), prompt engineering với JSON schema enforcement
- **Hạ tầng:** Docker/ECS Fargate, VPC (public/private subnets), ALB, Auto Scaling, ECR

#### Lộ trình phát triển

**Tháng 0 — Nghiên cứu & Kiến trúc**
Tìm hiểu React Native/Expo với Amplify Gen 2, thiết kế kiến trúc serverless-AI, phác thảo sơ đồ giải pháp và định nghĩa mô hình dữ liệu.

**Tháng 1 — Khung hạ tầng**
Ước tính chi phí bằng AWS Pricing Calculator (xác nhận dưới $65/tháng cho 1.000 user). Khởi tạo Amplify Gen 2 sandbox, Cognito + Google federation, schema GraphQL 8 model.

**Tháng 2 — Logic cốt lõi**
Triển khai 4 Lambda handler TypeScript, kết nối AppSync resolver, xây dựng UI React Native cho 6 tab chính, tích hợp pipeline upload và resize ảnh qua S3.

**Tháng 3 — Tích hợp & Ra mắt**
Tích hợp Bedrock (Qwen3-VL) cho 10 tác vụ AI, kiểm thử E2E trên 3 môi trường, xử lý các lỗi biên (JWT federation, `discoverTables()`, `NoValidAuthTokens`), triển khai production.

**Hậu ra mắt — Tối ưu liên tục**
Cải thiện UX từ phản hồi người dùng, nâng cấp gamification, tinh chỉnh prompt engineering, phát triển gói premium.

---

### 5. Lịch trình & Cột mốc

| Giai đoạn | Thời lượng | Nội dung chính |
|-----------|------------|----------------|
| **Tháng 0 — Tiền kỳ** | 1 tháng | Lập kế hoạch UI/UX trên Figma, chuyển đổi từ Flutter sang React Native, thiết kế kiến trúc trên draw.io, đánh giá dịch vụ AWS |
| **Tháng 1 — Hạ tầng** | 1 tháng | Khởi tạo Amplify Gen 2, cấu hình Cognito + Google OAuth, định nghĩa 8 model DynamoDB, xây dựng UI cốt lõi (Home, Add Food, Kitchen) |
| **Tháng 2 — Phát triển** | 1 tháng | Kết nối AppSync GraphQL, triển khai `processNutrition` và `friendRequest`, xây dựng giao diện Tủ lạnh, tích hợp S3 + `resizeImage` |
| **Tháng 3 — Ra mắt** | 1 tháng | Triển khai `aiEngine` (10 tác vụ Bedrock), gamification, E2E testing 3 môi trường, sửa lỗi JWT + `discoverTables()`, production release |
| **Hậu ra mắt** | Liên tục | Tối ưu hiệu suất, phản hồi người dùng, iOS EAS Build pipeline, mở rộng quy mô năm đầu |

---

### 6. Ước tính Ngân sách

Tính cho **1.000 người dùng hoạt động × 3 phiên/ngày × 30 ngày** = 90.000 tương tác AI/tháng.

#### Chi phí hạ tầng hàng tháng

| Thành phần | Chi phí | Ghi chú |
|------------|---------|---------|
| Amazon S3 | $2.03 | 3GB lưu trữ + 5GB transfer out + PUT/GET requests |
| AWS Lambda | $0.26 | 270K requests, trung bình 0.2s, 512MB ARM64 |
| AppSync GraphQL | $0.34 | 270K operations @ $4/triệu |
| Amazon DynamoDB | $0.47 | 2GB lưu trữ + 810K reads + 180K writes |
| Amazon Cognito | $5.50 | 1.000 MAU @ $0.0055/MAU (gói Lite) |
| CloudWatch | $1.00 | 5 custom metrics + API logging |
| CloudTrail | $0.05 | 50.000 management events |
| Secrets Manager | $1.20 | 3 secrets (API keys, JWT key, DB config) |
| AWS KMS | $1.00 | 1 customer-managed key |
| AWS Textract | $3.60 | 2.400 lần quét nhãn @ $0.0015/trang |
| **Amazon Bedrock (AI)** | **$45.42** | 90K lệnh gọi: input tokens ($16.70) + output tokens ($28.73) |
| **Tổng** | **$60.87/tháng** | **$730.44/năm** |

#### So sánh giá Bedrock (Qwen3-VL 235B) theo region

| Region | Input | Output | Chênh lệch |
|--------|-------|--------|------------|
| US East (Virginia) | $0.00053/1K tokens | $0.00266/1K tokens | Baseline |
| Asia Pacific (Tokyo) | $0.00064/1K tokens | $0.00322/1K tokens | +21% |
| Asia Pacific (Mumbai) | $0.00062/1K tokens | $0.00313/1K tokens | +18% |
| Asia Pacific (Sydney) | *(dùng cho production)* | *(dùng cho production)* | ap-southeast-2 |

**Chi phí phần mềm:** $0 — toàn bộ công cụ phát triển là mã nguồn mở. Cần tài khoản Apple Developer ($99/năm) nếu phân phối trên iOS.

---

### 7. Đánh giá Rủi ro

| Rủi ro | Tác động | Xác suất | Cách xử lý |
|--------|----------|----------|------------|
| **Chi phí Bedrock vượt ngưỡng** | Trung bình | Trung bình | Cache kết quả AI trong DynamoDB, giới hạn tốc độ request, đặt AWS Budget alert tại $80/tháng |
| **Lỗi xác thực Cognito/JWT** | Cao | Thấp | E2E testing đầy đủ cho luồng federation, dự phòng AsyncStorage session cục bộ, xử lý `UserNotConfirmedException` và `NotAuthorizedException` |
| **AI sinh dữ liệu sai (hallucination)** | Trung bình | Trung bình | Prompt templates với JSON schema bắt buộc, cho phép người dùng xác minh và chỉnh sửa, duy trì ~200 món Việt đã xác minh sẵn trong DynamoDB |
| **Rào cản build iOS** | Thấp | Cao | Ưu tiên Android cho MVP, dùng EAS Build cloud cho iOS, lên kế hoạch macOS CI runners ở giai đoạn sau |
| **Table discovery DynamoDB không chính xác** | Trung bình | Thấp | **Nguyên nhân:** `friendRequest` Lambda dùng `discoverTables()` qua `ListTables`, trả sai tên bảng khi nhiều môi trường cùng tồn tại. **Xử lý:** inject tên bảng chính xác qua CDK escape hatch — `cfnFriendRequestFn.addPropertyOverride('Environment.Variables.USER_TABLE_NAME', backend.data.resources.tables['user'].tableName)`, CDK tự resolve đúng ARN suffix khi deploy |
| **Giới hạn capacity Bedrock** | Thấp | Thấp | Hiện dùng ap-southeast-2; dự phòng us-east-1 nếu cần |

#### Kế hoạch dự phòng

- Khi AI/Bedrock gặp sự cố: fallback về nhập liệu thủ công hoặc fuzzy match trên ~200 món Việt đã nạp sẵn.
- Khi deploy backend lỗi: rollback qua CloudFormation/Amplify trên 3 môi trường.
- Khi chi phí Qwen3-VL leo thang: chuyển sang Claude Haiku hoặc Llama trên Bedrock.

---

### 8. Kết quả Kỳ vọng

#### Cải tiến kỹ thuật

- Thời gian ghi nhật ký giảm từ ~3 phút (nhập tay) xuống ~10 giây (AI qua ảnh/giọng nói).
- Hạ tầng serverless sẵn sàng chịu tải 10.000+ người dùng đồng thời với chi phí idle gần bằng 0 ($0.47/tháng DynamoDB cơ bản).
- Chiến lược AI lai ghép (DynamoDB fuzzy match + Bedrock AI dự phòng) cân bằng hiệu quả chi phí và độ chính xác dữ liệu.

#### Giá trị dài hạn

- Xây dựng cơ sở dữ liệu món Việt Nam được xác minh, mở rộng tự nhiên qua đóng góp của người dùng và AI — hiện đã có ~200 món và đang phát triển.
- Các mẫu kiến trúc AI tái sử dụng được (prompt templates, Lambda orchestration, Bedrock integration) có thể áp dụng cho các tính năng sức khỏe tương lai.
- Nền tảng xã hội với gamification (thú cưng tiến hóa, streak ngày, thử thách) thúc đẩy thói quen sử dụng hàng ngày và giữ chân người dùng lâu dài.

---

### 9. Bước Tiếp Theo

**iOS Pipeline** — Chuyển từ Android-first MVP sang iOS qua EAS Build cloud runners; kích hoạt macOS CI runner khi lượng người dùng đủ bù chi phí.

**Mở rộng cơ sở dữ liệu món Việt** — Từ ~200 item seed ban đầu, thu thập các entry do AI tạo (`source: "AI Generated"` → `verified: false`) để đội ngũ review; mục tiêu 1.000+ món đã xác minh sau năm đầu.

**Gói Premium** — Xác thực mô hình giá (tỉ lệ chuyển đổi đủ bù $60.87/tháng baseline), phát hành phân tích ảnh không giới hạn, báo cáo AI coach tuần, và biểu đồ macro nâng cao.

**Observability** — Kết nối 4 CloudWatch custom metrics vào dashboard tập trung + cài đặt Budget alarm tại ngưỡng $80/tháng.

**Fallback Bedrock cross-region** — Nếu capacity Qwen3-VL tại `ap-southeast-2` bị giới hạn, bổ sung route dự phòng đến Claude Haiku `us-east-1` để đảm bảo SLA.