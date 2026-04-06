---
title: "IAM Policies"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.11.2. </b> "
---

# 5.11.2 IAM Policies

Phụ lục này ghi lại từng statement IAM được gắn vào các execution role của Lambda và resource policy của S3 bucket trong workshop. Toàn bộ JSON bên dưới được trích ra từ `backend/amplify/backend.ts` — đây là điểm CDK entry point duy nhất nơi Amplify Gen 2 gắn thêm quyền lên trên các role tự sinh.

Amplify Gen 2 mặc định tạo cho mỗi Lambda một managed execution role (có sẵn `AWSLambdaBasicExecutionRole` cho CloudWatch logs). Mọi thứ khác — Bedrock, Transcribe, DynamoDB, S3 — đều được thêm tường minh qua CDK escape hatch:

```typescript
backend.<fn>.resources.lambda.addToRolePolicy(new iam.PolicyStatement({ ... }))
```

Nếu một statement không nằm trong `backend.ts`, Lambda không có quyền đó. Không có cấu hình ẩn nào khác.

## Tham chiếu source

- File: `backend/amplify/backend.ts`
- CDK imports: `aws-cdk-lib/aws-iam`, `aws-cdk-lib/aws-s3`, `aws-cdk-lib/aws-s3-notifications`

## Role của aiEngine

Lambda `aiEngine` xử lý toàn bộ Bedrock calls, voice transcription và đọc ảnh từ S3. Đây là Lambda có quyền rộng nhất trong số bốn Lambda của workshop.

### Bedrock: invoke model Qwen3-VL

```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModel",
    "bedrock:InvokeModelWithResponseStream"
  ],
  "Resource": [
    "arn:aws:bedrock:ap-southeast-2::foundation-model/qwen.qwen3-vl-235b-a22b"
  ]
}
```

Ghi chú:

- Resource ARN được pin vào đúng một model ở `ap-southeast-2`. Nếu bạn thêm model thứ hai (ví dụ một Claude text-only làm fallback), hãy append ARN của nó vào mảng `Resource` thay vì đổi thành `*`.
- `InvokeModelWithResponseStream` có mặt vì handler để ngỏ đường streaming, dù hiện tại code gọi `InvokeModel` đồng bộ qua `callQwen()`.

### Transcribe: job voice-to-text bất đồng bộ

```json
{
  "Effect": "Allow",
  "Action": [
    "transcribe:StartTranscriptionJob",
    "transcribe:GetTranscriptionJob",
    "transcribe:DeleteTranscriptionJob"
  ],
  "Resource": ["*"]
}
```

Ghi chú:

- Transcribe không hỗ trợ resource-level ARN cho `StartTranscriptionJob` ở mọi region, nên `Resource` để `*`. Đây là grant rộng nhất trong toàn stack — xem callout ở cuối trang.
- `DeleteTranscriptionJob` có mặt để Lambda dọn dẹp các job đã xong; hiện handler để comment phần cleanup cho mục đích debug, nhưng quyền vẫn ở đó để khỏi cần redeploy khi bật lại.

### S3: đọc ảnh, voice file và xoá khi cleanup

Các grant này đến từ CDK helper, không phải raw policy statement:

```typescript
s3Bucket.grantRead(aiEngineLambda);
s3Bucket.grantDelete(aiEngineLambda);
```

Mở rộng ra tương đương `s3:GetObject`, `s3:GetObject*`, `s3:List*`, `s3:DeleteObject*` trên bucket ARN và `bucket/*`. Lambda đọc từ `incoming/` (ảnh) và `voice/` (audio), xoá khi cleanup bật.

### Inject environment variable

Tên S3 bucket được truyền vào qua CFN property override, không hardcode:

```typescript
const cfnAiEngineFn = aiEngineLambda.node.defaultChild as cdk.aws_lambda.CfnFunction;
cfnAiEngineFn.addPropertyOverride(
  'Environment.Variables.STORAGE_BUCKET_NAME',
  s3Bucket.bucketName
);
```

Handler đọc giá trị này qua `process.env.STORAGE_BUCKET_NAME` và fail-fast nếu rỗng.

## Role của processNutrition

Lambda `processNutrition` trả lời các truy vấn dinh dưỡng bằng cách scan bảng `Food-*` trên DynamoDB với thuật toán fuzzy match, và chỉ gọi Bedrock khi DB miss.

### DynamoDB: đọc bảng Food

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:Scan",
    "dynamodb:Query",
    "dynamodb:GetItem",
    "dynamodb:BatchGetItem",
    "dynamodb:DescribeTable"
  ],
  "Resource": ["arn:aws:dynamodb:*:*:table/Food-*"]
}
```

Wildcard `Food-*` khớp với tên bảng Amplify sinh ra (ví dụ `Food-abc123xyz-NONE`). Không cho phép write — seed data bất biến tại runtime.

### DynamoDB: ListTables (đường discovery legacy)

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:ListTables"],
  "Resource": ["*"]
}
```

Đây là phần rủi ro. Hàm `discoverTableName()` của Lambda fallback về `ListTables()` khi `FOOD_TABLE_NAME` không set, rồi lấy tên đầu tiên bắt đầu bằng `Food-`. Trong một AWS account dùng chung với nhiều sandbox, nó sẽ pick **sai** bảng.

**Khuyến nghị**: inject `FOOD_TABLE_NAME` tường minh (theo đúng pattern `friendRequest`) và xoá `ListTables` khỏi policy trước khi lên production:

```typescript
const cfnProcessNutritionFn = backend.processNutrition.resources.cfnResources.cfnFunction;
cfnProcessNutritionFn.addPropertyOverride(
  'Environment.Variables.FOOD_TABLE_NAME',
  backend.data.resources.tables['Food'].tableName
);
```

### Bedrock: đường fallback

Cùng một Bedrock statement như `aiEngine` cũng được gắn vào đây, vì nutrition lookup sẽ rơi xuống gọi Qwen3-VL khi không có row nào match trong DB. Resource ARN giống hệt.

## Role của friendRequest

Lambda `friendRequest` cài đặt friend graph: send / accept / decline / remove / block. Chỉ đụng DynamoDB.

### DynamoDB: bảng user + Friendship

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem",
    "dynamodb:PutItem",
    "dynamodb:UpdateItem",
    "dynamodb:DeleteItem",
    "dynamodb:Query",
    "dynamodb:Scan",
    "dynamodb:BatchGetItem",
    "dynamodb:BatchWriteItem",
    "dynamodb:DescribeTable",
    "dynamodb:TransactWriteItems"
  ],
  "Resource": [
    "arn:aws:dynamodb:*:*:table/user-*",
    "arn:aws:dynamodb:*:*:table/user-*/index/*",
    "arn:aws:dynamodb:*:*:table/Friendship-*",
    "arn:aws:dynamodb:*:*:table/Friendship-*/index/*"
  ]
}
```

Ghi chú:

- `TransactWriteItems` bắt buộc vì accept friend request ghi hai row (A→B và B→A) trong cùng một transaction.
- Suffix `/index/*` cho phép `Query` trên GSI `friend_id`.
- Không có Bedrock. Không có S3. Nếu tính năng tương lai cần gửi "friend suggestion" do Bedrock sinh, hãy thêm statement mới — không nới lỏng statement này.

### Inject env var (cách làm đúng)

```typescript
const cfnFriendRequestFn = friendRequestLambda.node.defaultChild as cdk.aws_lambda.CfnFunction;
cfnFriendRequestFn.addPropertyOverride(
  'Environment.Variables.USER_TABLE_NAME',
  backend.data.resources.tables['user'].tableName
);
cfnFriendRequestFn.addPropertyOverride(
  'Environment.Variables.FRIENDSHIP_TABLE_NAME',
  backend.data.resources.tables['Friendship'].tableName
);
```

CDK resolve tên bảng đúng tại thời điểm synth cho từng environment, nên sandbox và branch deploy không bao giờ đụng nhau. Đây là pattern mà `processNutrition` nên áp dụng.

## Role của resizeImage

Lambda `resizeImage` được wire như S3 event trigger trên prefix `incoming/`. Nó đọc ảnh upload, resize bằng sharp, ghi vào `media/` và xoá bản gốc.

Quyền được cấp qua CDK helper, không phải raw statement:

```typescript
s3Bucket.grantReadWrite(resizeLambda);
s3Bucket.grantDelete(resizeLambda);
```

Mở rộng thành:

- `s3:GetObject*`, `s3:GetBucket*`, `s3:List*` trên bucket
- `s3:PutObject`, `s3:PutObjectLegalHold`, `s3:PutObjectRetention`, `s3:PutObjectTagging`, `s3:PutObjectVersionTagging`, `s3:Abort*` để ghi
- `s3:DeleteObject*` để cleanup

Lambda này **không** có Bedrock, **không** DynamoDB, **không** Transcribe. Nếu build của bạn thêm service khác, nguyên tắc least-privilege nói: **tạo statement mới**, không nới statement sẵn có.

## S3 bucket resource policy

Transcribe chạy bất đồng bộ dưới service role riêng của nó, không phải role của Lambda. Role đó mặc định không có quyền S3, nên ta gắn bucket policy cho service principal Transcribe quyền đọc trên prefix `voice/*`:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "transcribe.amazonaws.com"
  },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::<bucket-name>/voice/*"
}
```

Trong CDK:

```typescript
s3Bucket.addToResourcePolicy(new iam.PolicyStatement({
  effect: iam.Effect.ALLOW,
  principals: [new iam.ServicePrincipal('transcribe.amazonaws.com')],
  actions: ['s3:GetObject'],
  resources: [`${s3Bucket.bucketArn}/voice/*`],
}));
```

Thiếu statement này, `StartTranscriptionJob` fail với `AccessDenied` dù Lambda role đã có `transcribe:*` — vì chính Transcribe, không phải Lambda, mới là entity đọc S3.

## S3 lifecycle rule (không phải IAM nhưng liên quan)

Prefix `incoming/` auto-expire sau 1 ngày để landing zone không bao giờ phình lên do upload chưa xử lý:

```typescript
cfnBucket.lifecycleConfiguration = {
  rules: [{
    id: 'CleanupIncomingLandingZone',
    status: 'Enabled',
    prefix: 'incoming/',
    expirationInDays: 1
  }]
};
```

Đề cập ở đây vì reviewer hay hỏi "sao `incoming/` không phình?" — rule này là câu trả lời, không phải IAM policy nào cả.

## Managed Cognito service role

Amplify Gen 2 tự tạo Cognito User Pool, client, identity pool và các IAM role đi kèm từ `auth/resource.ts`. **Không sửa các role này bằng tay.** Nếu cần thêm post-confirmation Lambda trigger, wire qua Amplify API (`defineAuth({ triggers: { ... } })`) — scaffolding sẽ cập nhật role cho bạn ở lần deploy tiếp theo. Sửa tay sẽ lệch CloudFormation state và lần `ampx sandbox` kế tiếp sẽ revert hoặc fail.

## Principle of least privilege — checklist trước production

Trước khi ship ra ngoài workshop sandbox, siết lại mọi `Resource: "*"` trong stack:

1. **Transcribe**: scope theo prefix tên job. Dùng resource ARN dạng `arn:aws:transcribe:ap-southeast-2:<account>:transcription-job/nutritrack-voice-*` cho `GetTranscriptionJob` và `DeleteTranscriptionJob`. `StartTranscriptionJob` có thể vẫn phải để `*` tuỳ region — verify với AWS IAM reference hiện hành.
2. **DynamoDB ListTables**: xoá hẳn khỏi `processNutrition` sau khi inject `FOOD_TABLE_NAME` qua env var (xem block phía trên).
3. **S3 bucket policy**: xác nhận `<bucket-name>` resolve về đúng bucket duy nhất của workshop. Nếu fork stack thành nhiều bucket, mỗi bucket cần statement riêng.
4. **Bedrock**: giữ ARN một model. Nếu thêm model khác, append — không bao giờ đổi sang `arn:aws:bedrock:*::foundation-model/*`.
5. **Cognito**: không sửa tay.

Mỗi dấu `*` ở trên là một finding mà reviewer sẽ flag. Fix trước, đừng để sau.

## File liên quan

- `backend/amplify/backend.ts` — file mà phụ lục này document
- `backend/amplify/process-nutrition/handler.ts` — pattern `discoverTableName()`
- `backend/amplify/friend-request/handler.ts` — pattern env-var (pattern đúng)
- `backend/amplify/ai-engine/handler.ts` — consumer của Bedrock + Transcribe + S3
- `backend/amplify/resize-image/handler.ts` — consumer của S3 event
