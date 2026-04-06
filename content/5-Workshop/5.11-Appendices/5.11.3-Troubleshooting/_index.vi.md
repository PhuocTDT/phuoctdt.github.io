---
title: "Troubleshooting"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.11.3. </b> "
---

# 5.11.3 Troubleshooting

Runbook phân loại theo nhóm cho các lỗi bạn sẽ thực sự gặp khi build và chạy NutriTrack. Mỗi mục đi theo cấu trúc ba dòng:

- **Triệu chứng** — chuỗi lỗi chính xác hoặc hành vi quan sát được
- **Nguyên nhân gốc** — thứ thực sự đang sai
- **Cách fix** — lệnh cụ thể cần chạy hoặc file cần sửa

Quét theo category trước. Nếu lỗi của bạn không có trong danh sách, nhảy xuống checklist debug tổng quát ở cuối trang.

## Amplify sandbox

### `Error: UserPoolClient already exists`

- **Triệu chứng**: `npx ampx sandbox` fail khi deploy CFN với lỗi duplicate resource trên Cognito User Pool Client.
- **Nguyên nhân gốc**: lần chạy sandbox trước để lại state dở dang trong CloudFormation. Thư mục `.amplify/` local và CFN stack đã lệch nhau.
- **Fix**:

```bash
cd backend
npx ampx sandbox delete
npx ampx sandbox
```

Nếu chính `sandbox delete` cũng fail, xoá CFN stack thủ công trong AWS console (tìm `amplify-<app>-<user>-sandbox-*`), rồi retry.

### `The provided credentials are expired`

- **Triệu chứng**: sandbox hoặc `ampx generate outputs` fail với lỗi STS credentials.
- **Nguyên nhân gốc**: SSO session token hết hạn.
- **Fix**:

```bash
aws sso login --profile <profile-name>
```

Xác nhận identity hiện tại trước khi retry:

```bash
aws sts get-caller-identity --profile <profile-name>
```

### `sandbox secret not found for key GOOGLE_CLIENT_ID`

- **Triệu chứng**: sandbox deploy fail vì `auth/resource.ts` reference một secret chưa từng được set.
- **Nguyên nhân gốc**: secrets là per-sandbox và không đồng bộ từ máy developer khác.
- **Fix**:

```bash
npx ampx sandbox secret set GOOGLE_CLIENT_ID
npx ampx sandbox secret set GOOGLE_CLIENT_SECRET
npx ampx sandbox secret list
```

Paste giá trị từ Google Cloud Console OAuth 2.0 client của bạn khi được hỏi.

### `amplify_outputs.json not found` ở frontend

- **Triệu chứng**: Metro hoặc Expo fail start với lỗi missing module `amplify_outputs.json`.
- **Nguyên nhân gốc**: file này do backend generate ra và không được commit.
- **Fix**:

```bash
cd backend
npx ampx generate outputs --outputs-out-dir ../frontend
```

Chạy lại mỗi khi backend schema đổi.

## Bedrock

### `AccessDeniedException: You don't have access to the model with the specified model ID`

- **Triệu chứng**: invoke `aiEngine` trả về AccessDenied tham chiếu `qwen.qwen3-vl-235b-a22b`.
- **Nguyên nhân gốc**: model access chưa được cấp cho account của bạn ở `ap-southeast-2`. Bedrock gate mọi foundation model sau một bước opt-in tường minh.
- **Fix**: AWS Console → Bedrock → *Model access* → Request access → chọn `Qwen3-VL 235B A22B` ở `ap-southeast-2` → submit. Propagate mất vài phút đến vài giờ. Invoke lại sau khi status chuyển thành *Access granted*.

### `ThrottlingException: Rate exceeded`

- **Triệu chứng**: throttle gián đoạn khi tải cao, thường trong các burst image analysis.
- **Nguyên nhân gốc**: Bedrock on-demand có rate limit theo account / model / region.
- **Fix**: thêm exponential backoff trong `callQwen()` (retry khi `ThrottlingException`, `ModelTimeoutException`, `ServiceUnavailableException`). Với traffic bền vững, request Provisioned Throughput qua AWS Support.

### `ValidationException: Input is too long`

- **Triệu chứng**: image analysis fail khi user upload ảnh rất lớn.
- **Nguyên nhân gốc**: ảnh base64 cộng prompt vượt quá context window của model.
- **Fix**: Lambda `resizeImage` đã cap cạnh dài nhất ở 1280 px trước khi ảnh tới `aiEngine`. Kiểm tra handler đang đọc từ `media/` (sau resize) chứ không phải `incoming/` (trước resize) cho đường analysis. Nếu path đúng, giảm max dimension xuống 1024 px.

### Lỗi sai region trên Bedrock

- **Triệu chứng**: `The model ID is not supported in this region` hoặc tương tự.
- **Nguyên nhân gốc**: Bedrock client hoặc Lambda environment được cấu hình ở region không được approve cho `qwen.qwen3-vl-235b-a22b`.
- **Fix**: hardcode `REGION = "ap-southeast-2"` trong `ai-engine/handler.ts` và xác nhận IAM resource ARN khớp. Đừng dựa vào `AWS_REGION` — Lambda có thể chạy ở region bất kỳ nhưng vẫn cần gọi Bedrock ở Sydney.

## DynamoDB

### `ResourceNotFoundException: Requested resource not found` trong processNutrition

- **Triệu chứng**: nutrition lookup fail trong sandbox khi nhiều developer dùng chung AWS account.
- **Nguyên nhân gốc**: `discoverTableName()` dùng `ListTables()` và lấy cái tên `Food-*` đầu tiên nó tìm được — mà tên đó có thể thuộc sandbox của người khác.
- **Fix**: inject tên bảng tường minh trong `backend/amplify/backend.ts`:

```typescript
const cfnProcessNutritionFn = backend.processNutrition.resources.cfnResources.cfnFunction;
cfnProcessNutritionFn.addPropertyOverride(
  'Environment.Variables.FOOD_TABLE_NAME',
  backend.data.resources.tables['Food'].tableName
);
```

Handler đã tôn trọng `process.env.FOOD_TABLE_NAME` trước. Deploy lại, rồi xoá `dynamodb:ListTables` khỏi IAM statement.

### Cùng loại bug trên friendRequest

- **Triệu chứng**: Lambda friendRequest ném `USER_TABLE_NAME env var not set` hoặc ghi vào bảng Friendship sai.
- **Nguyên nhân gốc**: block inject env var thiếu hoặc lỗi thời sau khi đổi tên schema.
- **Fix**: `friendRequest` đã có pattern đúng trong `backend.ts`. Nếu bạn đổi tên model `user` hoặc `Friendship` trong `data/resource.ts`, cập nhật key trong `backend.data.resources.tables['<name>']` cho khớp. Cả `USER_TABLE_NAME` và `FRIENDSHIP_TABLE_NAME` đều phải resolve.

### `ConditionalCheckFailedException` trên friendRequest

- **Triệu chứng**: `sendRequest` hoặc `acceptRequest` fail với lỗi conditional check.
- **Nguyên nhân gốc**: row friendship đã tồn tại (request trùng, hoặc phía bên kia đã accept rồi).
- **Fix**: catch `ConditionalCheckFailedException` trong Lambda và trả về response có typed code (`{ success: false, code: 'ALREADY_FRIENDS' }`). Frontend render message thân thiện, không phải crash.

## Lambda / Node.js 22

### `Cannot find module 'sharp'` trong resizeImage

- **Triệu chứng**: `resizeImage` crash ngay lúc import.
- **Nguyên nhân gốc**: sharp ship native binary theo từng OS/CPU. Cài trên macOS hoặc Windows thì bundle Lambda không có binary Linux ARM64.
- **Fix**: install sharp với target đúng:

```bash
cd backend/amplify/resize-image
npm install --os=linux --cpu=arm64 sharp
```

Hoặc package sharp thành Lambda layer và reference từ function. Xác nhận binary tồn tại trong `node_modules/@img/sharp-linux-arm64/` trước khi deploy.

### Cold start vượt quá 2 giây

- **Triệu chứng**: invocation đầu tiên sau khi idle chậm; các call sau nhanh.
- **Nguyên nhân gốc**: Node.js 22 với nhiều SDK client có init không hề rẻ. Memory setting cũng giới hạn CPU.
- **Fix**: tăng `memoryMB` trong `resource.ts` của function (256 → 512 hoặc 1024). Với hot path user-facing (`aiEngine`), cân nhắc Provisioned Concurrency thay vì cứ tăng memory vô hạn.

### Timeout trên aiEngine khi Bedrock call lâu

- **Triệu chứng**: voice-to-food hoặc image analysis trả về Lambda timeout sau 30 giây.
- **Nguyên nhân gốc**: Bedrock có thể mất 10–60 s với prompt dài hoặc model routing cold. Timeout Lambda mặc định quá thấp.
- **Fix**: set `timeoutSeconds: 120` trong `backend/amplify/ai-engine/resource.ts`. Nếu chuỗi Transcribe + Bedrock thực sự vượt quá, redesign thành hai invocation (start job → poll qua AppSync subscription) thay vì tiếp tục tăng timeout.

## Cognito

### `UserNotFoundException` khi sign-in

- **Triệu chứng**: login fail ngay với `UserNotFoundException`.
- **Nguyên nhân gốc**: email chưa đăng ký trong User Pool này.
- **Fix**: route user sang màn signup. Không surface raw error — nó leak email nào tồn tại.

### `UserNotConfirmedException`

- **Triệu chứng**: login fail vì email OTP chưa verify.
- **Nguyên nhân gốc**: signup xong nhưng step verify bị skip.
- **Fix**: gọi `resendSignUpCode({ username })` rồi navigate sang verify-otp. User nhập code mới và confirm.

### `NotAuthorizedException: Incorrect username or password`

- **Triệu chứng**: login fail dù user tồn tại.
- **Nguyên nhân gốc**: password sai, hoặc account bị disable trong Cognito console.
- **Fix**: cung cấp flow "Forgot password". Nếu do admin (account bị disable), check status user trong Cognito console.

### Google OAuth redirect loop

- **Triệu chứng**: Google login nhảy qua nhảy lại giữa Cognito hosted UI và Google, không bao giờ về app.
- **Nguyên nhân gốc**: callback URL mismatch. Danh sách callback URL cho phép của Cognito và authorised redirect URI của Google Cloud Console phải khớp **chính xác**, kể cả scheme, host, path và dấu `/` cuối.
- **Fix**: vào Cognito → App client settings, copy danh sách callback. Paste nguyên văn sang Google Cloud Console → OAuth 2.0 Client → *Authorised redirect URIs*. Xoá các URL dev cũ không còn tồn tại.

## Expo / React Native

### Lỗi peer dependency khi `npm install`

- **Triệu chứng**: `npm install` trong `frontend/` fail với `ERESOLVE` peer conflict.
- **Nguyên nhân gốc**: hệ sinh thái native module của Expo có peer range chặt mà React Native version mới vi phạm.
- **Fix**: luôn dùng `--legacy-peer-deps`. Đã enforce trong `frontend/.npmrc`; nếu xoá file đó thì tạo lại:

```text
legacy-peer-deps=true
```

### Metro không resolve được `amplify_outputs.json`

- **Triệu chứng**: Metro bundler lỗi khi import `amplify_outputs.json`.
- **Nguyên nhân gốc**: file không commit và chưa generate local.
- **Fix**: regenerate từ backend:

```bash
cd backend
npx ampx generate outputs --outputs-out-dir ../frontend
```

### Biometric prompt không xuất hiện

- **Triệu chứng**: `expo-local-authentication` không bao giờ prompt trên Android hoặc iOS.
- **Nguyên nhân gốc**: Expo Go không ship sẵn mọi native module. Biometric cần dev build.
- **Fix**:

```bash
cd frontend
npx expo run:android      # hoặc
npx expo run:ios
# Hoặc build hosted:
eas build --profile development --platform android
```

Cài lại dev client và prompt sẽ hiện.

## ECS (phía FastAPI)

### Task kẹt ở `PROVISIONING`

- **Triệu chứng**: ECS task không bao giờ tới `RUNNING`; service event hiện pull fail lặp lại.
- **Nguyên nhân gốc**: không có network path tới ECR (thiếu NAT Gateway cho private subnet), hoặc task execution role thiếu quyền ECR.
- **Fix**: xác nhận `AmazonECSTaskExecutionRolePolicy` đã attach lên execution role (`ecr:GetAuthorizationToken`, `ecr:BatchGetImage`). Nếu dùng private subnet, đảm bảo có NAT Gateway hoặc ECR interface endpoint.

### Task stop ngay sau khi start

- **Triệu chứng**: task chuyển `PROVISIONING → RUNNING → STOPPED` trong vài giây.
- **Nguyên nhân gốc**: container crash lúc startup — thiếu env var, fail kết nối DB, syntax error.
- **Fix**: mở CloudWatch Logs của log group task đó. 10 dòng đầu sau startup thường chỉ đúng vấn đề. Thêm env var thiếu qua task definition, không phải qua image.

### ALB trả `502 Bad Gateway`

- **Triệu chứng**: ALB route traffic nhưng mọi request đều 502.
- **Nguyên nhân gốc**: health check path (mặc định `/health`) chưa được implement trong app FastAPI, nên target group mark task unhealthy.
- **Fix**: thêm handler đơn giản:

```python
@app.get("/health")
def health():
    return {"status": "ok"}
```

Rebuild, push, redeploy. Xác nhận health check path của target group trong console.

## Checklist debug tổng quát

Khi không có mục nào trong trang này khớp, đi theo list dưới đây, rẻ nhất trước:

1. **Reproduce local** bằng `npx ampx sandbox` trỏ vào account của riêng bạn. Hầu hết bug "bí ẩn" sẽ trở nên nhàm chán khi bạn làm chủ stack.
2. **Kiểm tra CloudWatch Logs** cho đúng Lambda đang lỗi. Set `DEBUG=true` làm env var trên function để bật logger `debug()` trong handler.
3. **Set CloudWatch log retention** cho mọi log group (mặc định *Never expire*, vừa tốn tiền vừa vô ích). Hai tuần là default tốt cho non-production.
4. **Tag mọi resource** với `Project=NutriTrack` và `Environment=sandbox|staging|prod`. Giúp track cost và cleanup dễ hơn.
5. **Chạy AWS CLI với `--debug`** khi một call hành xử bất thường. Full signed request và raw response giúp phân biệt lỗi policy hay lỗi config.
6. **So sánh IAM statement** với [5.11.2 IAM Policies](../5.11.2-IAM-Policies/). Nếu một statement bị thiếu, retry ở tầng code không cứu được.
7. **Check region** hai lần. NutriTrack pin Bedrock vào `ap-southeast-2` — service khác có thể theo region khác, Bedrock thì không.

Nếu làm hết checklist vẫn stuck, capture lại lỗi, lệnh chính xác, region và Lambda request ID rồi file issue.
