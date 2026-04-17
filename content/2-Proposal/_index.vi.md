---
menuTitle: "Proposal"
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
- **AWS Lambda** với 5 hàm chuyên biệt xử lý mọi logic backend — từ điều phối AI, xử lý dinh dưỡng đến quản lý bạn bè và tối ưu ảnh.
- **Amazon DynamoDB** lưu trữ 6 mô hình dữ liệu cốt lõi với khả năng tự co giãn theo tải.
- **Amazon Bedrock (Qwen3-VL 235B)** phân tích ảnh đồ ăn, gợi ý công thức, và sinh dữ liệu dinh dưỡng thông minh.
- **AWS Amplify Gen 2** kết hợp React Native/Expo tạo ra giao diện song ngữ Việt–Anh mượt mà.

**Tính năng nổi bật:**

| Tính năng | Mô tả |
|-----------|-------|
| 📸 Phân tích ảnh đồ ăn | Chụp một tấm — nhận ngay bảng dinh dưỡng chi tiết qua Vision AI |
| 🎙️ Ghi nhật ký bằng giọng nói | AWS Transcribe chuyển lời nói thành thực phẩm cụ thể |
| 🍳 Tủ lạnh thông minh | Theo dõi thực phẩm, gợi ý thực đơn từ nguyên liệu sắp hết hạn |
| 🤖 AI Coach "Ollie" | Tư vấn dinh dưỡng cá nhân hóa (tên tiếng Việt: Bảo) |
| 🎮 Gamification | Hành trình tiến hóa Rồng 180 ngày, thú cưng, thử thách |
| 👥 Tính năng xã hội | Kết bạn, bảng xếp hạng Streak và Pet Score công khai |

#### Hiệu quả chi phí

Chi phí vận hành thực tế được tối ưu hóa ở mức **$60.87/tháng** cho 1.000 người dùng hoạt động. Điều này đạt được nhờ chiến lược đưa các hàm Lambda xử lý AI ra ngoài VPC để loại bỏ chi phí NAT Gateway và giảm độ trễ tối đa.

---

### 3. Kiến trúc Giải pháp

Dữ liệu di chuyển từ ứng dụng React Native → CloudFront/WAF (bảo mật biên) → AppSync GraphQL API → các Lambda handler → DynamoDB. Các tác vụ nặng về xử lý ảnh được ủy quyền cho ECS Fargate, trong khi trí tuệ nhân tạo được đảm nhận bởi Amazon Bedrock.

#### Các dịch vụ AWS

| Dịch vụ | Vai trò |
|---------|---------|
| **AWS Amplify Gen 2** | Điều phối hạ tầng bằng TypeScript CDK, quản lý pipeline CI/CD và 3 môi trường phân tách |
| **AWS AppSync** | GraphQL API với real-time subscriptions và phân quyền owner-based |
| **AWS Lambda** | 5 hàm logic: `ai-engine`, `scan-image`, `process-nutrition`, `friend-request`, `resize-image` |
| **Amazon DynamoDB** | 6 bảng NoSQL: `user`, `Food`, `FoodLog`, `FridgeItem`, `Friendship`, `UserPublicStats` |
| **Amazon Bedrock** | Qwen3-VL 235B (ap-southeast-2) xử lý 9 tác vụ AI (Vision, NLP, Coaching) |
| **AWS Transcribe** | Chuyển đổi voice recordings (.m4a) từ S3 thành văn bản xử lý món ăn |
| **Amazon S3** | Lưu trữ media phân vùng: `incoming/` (temp), `voice/`, `avatar/`, `media/` (permanent) |
| **Amazon Cognito** | Xác thực tập trung: Email/OTP và Google OAuth2 federation |
| **Amazon ECS Fargate** | Chạy FastAPI backend trong VPC để xử lý Vision AI cường độ cao |
| **AWS CloudWatch** | Giám sát tập trung với Custom Metrics và Budget Alerts |
| **AWS Secrets Manager** | Bảo mật API Keys cho ECS và Bedrock Model IDs |

#### Thiết kế kỹ thuật đặc thù

**Xử lý AI lai ghép (Hybrid)** — Hệ thống ưu tiên truy vấn fuzzy match từ Database cục bộ trước khi Fallback sang Bedrock AI để đảm bảo tốc độ và tiết kiệm chi phí.

**Tối ưu hóa VPC** — Hàm `scan-image` Lambda được đặt ngoài VPC để kết nối trực tiếp với S3 và Bedrock, giúp giảm 90% độ trễ và loại bỏ hoàn toàn chi phí NAT Gateway cho các tác vụ AI.

**Hệ thống AI Engine (9 tác vụ)** — Bộ não trung tâm điều phối:
- `analyzeFoodImage`: Phân tích ảnh món ăn trực tiếp.
- `voiceToFood`: Chuyển giọng nói sang dữ liệu dinh dưỡng.
- `generateCoachResponse`: Phản hồi hội thoại của Coach Ollie.
- `generateFood`: Sinh dữ liệu cho món không có trong DB.
- `calculateMacros`: Tính toán mục tiêu calo theo chỉ số cơ thể.
- Và các tác vụ khác như: `fixFood`, `ollieCoachTip`, `generateRecipe`, `weeklyInsight`.

---

### 4. Triển khai Kỹ thuật

#### Công nghệ sử dụng

- **Frontend:** React Native, Expo Router, TypeScript, Zustand (State), i18n.
- **Backend:** Amplify Gen 2, AppSync, DynamoDB, Lambda (Node.js 22).
- **AI/ML:** Bedrock (Qwen3-VL 235B), Transcribe, Vision AI trên ECS Fargate.
- **DevOps:** CI/CD tự động, CDK Escape Hatches để giải quyết các lỗi Table Discovery.

#### Lộ trình phát triển

**Giai đoạn 1 — Hạ tầng & Cốt lõi**
Thiết kế schema 6 bảng, cấu hình Cognito Auth và xây dựng các UI cơ bản cho việc nhập liệu.

**Giai đoạn 2 — Tích hợp AI & Voice**
Triển khai `aiEngine` phối hợp với Transcribe và Bedrock. Tối ưu hóa luồng resize ảnh tự động qua S3 triggers.

**Giai đoạn 3 — Xã hội & Gamification**
Xây dựng hệ thống bạn bè, bảng xếp hạng và logic tiến hóa thú cưng (Minh Long Dragon) theo chuỗi vận động thực tế.

**Giai đoạn 4 — Ổn định & Mở rộng**
Xử lý lỗi Table Discovery bằng CDK overrides, triển khai production trên region Sydney (ap-southeast-2).

---

### 5. Lịch trình & Cột mốc (Dragon Journey)

Hệ thống gamification được thiết kế với hành trình 180 ngày:
- **Trứng (Ngày 0-35)**: Trạng thái bắt đầu.
- **Sơ sinh (Ngày 36-71)**: Giai đoạn nở.
- **Thiếu niên (Ngày 72-107)**: Phát triển kích thước.
- **Trưởng thành (Ngày 108-143)**: Đạt đỉnh năng lượng.
- **Huyền thoại (Ngày 144+)**: Trạng thái tối thượng với các đặc quyền đặc biệt.

---

### 6. Đánh giá Rủi ro & Xử lý

| Rủi ro | Cách xử lý |
|--------|----------|
| **Lỗi Table Discovery** | Inject chính xác tên bảng qua CDK Escape Hatches (`addPropertyOverride`). |
| **Chi phí AI leo thang** | Cài đặt Budget Alert tại $80 và cache kết quả tìm kiếm trong DynamoDB. |
| **Hallucination (AI sai)** | Sử dụng Prompt Engineering với JSON Schema bắt buộc và cho phép sửa tay. |
| **Độ trễ VPC** | Đưa Lambda xử lý AI ra ngoài VPC (Public) để tối ưu đường truyền. |

---

### 7. Kết quả Kỳ vọng

- **Hiệu suất**: Thời gian đăng món ăn giảm từ 3 phút xuống còn 10 giây nhờ AI.
- **Độ chính xác**: Đạt 95% độ chính xác cho các món ăn Việt Nam nhờ mô hình Vision AI tiên tiến.
- **Khả năng mở rộng**: Sẵn sàng chịu tải 10.000+ người dùng với chi phí duy trì gần như bằng 0 khi không có người dùng (Idle).

---

### 8. Bước Tiếp Theo

- **Mở rộng DB món Việt**: Mục tiêu đạt 1.000+ món đã xác minh vào cuối năm.
- **EAS Build cho iOS**: Hoàn thiện pipeline đóng gói ứng dụng cho App Store.
- **Gói Premium**: Ra mắt các tính năng phân tích chuyên sâu (Weekly Insights) để tạo nguồn thu bền vững.
sự cố: fallback về nhập liệu thủ công hoặc fuzzy match trên ~200 món Việt đã nạp sẵn.
- Khi deploy backend lỗi: rollback qua CloudFormation/Amplify trên 3 môi trường.
- Khi chi phí Qwen3-VL leo thang: chuyển sang Claude Haiku hoặc Llama trên Bedrock.

---

### 9. Kết quả Kỳ vọng

#### Cải tiến kỹ thuật

- Thời gian ghi nhật ký giảm từ ~3 phút (nhập tay) xuống ~10 giây (AI qua ảnh/giọng nói).
- Hạ tầng serverless sẵn sàng chịu tải 10.000+ người dùng đồng thời với chi phí idle gần bằng 0 ($0.47/tháng DynamoDB cơ bản).
- Chiến lược AI lai ghép (DynamoDB fuzzy match + Bedrock AI dự phòng) cân bằng hiệu quả chi phí và độ chính xác dữ liệu.

#### Giá trị dài hạn

- Xây dựng cơ sở dữ liệu món Việt Nam được xác minh, mở rộng tự nhiên qua đóng góp của người dùng và AI — hiện đã có ~200 món và đang phát triển.
- Các mẫu kiến trúc AI tái sử dụng được (prompt templates, Lambda orchestration, Bedrock integration) có thể áp dụng cho các tính năng sức khỏe tương lai.
- Nền tảng xã hội với gamification (thú cưng tiến hóa, streak ngày, thử thách) thúc đẩy thói quen sử dụng hàng ngày và giữ chân người dùng lâu dài.

---

### 10. Bước Tiếp Theo

**iOS Pipeline** — Chuyển từ Android-first MVP sang iOS qua EAS Build cloud runners; kích hoạt macOS CI runner khi lượng người dùng đủ bù chi phí.

**Mở rộng cơ sở dữ liệu món Việt** — Từ ~200 item seed ban đầu, thu thập các entry do AI tạo (`source: "AI Generated"` → `verified: false`) để đội ngũ review; mục tiêu 1.000+ món đã xác minh sau năm đầu.

**Observability** — Kết nối 4 CloudWatch custom metrics vào dashboard tập trung + cài đặt Budget alarm tại ngưỡng $80/tháng.

**Fallback Bedrock cross-region** — Nếu capacity Qwen3-VL tại `ap-southeast-2` bị giới hạn, bổ sung route dự phòng đến Claude Haiku `us-east-1` để đảm bảo SLA.