---
title: "DynamoDB"
date: 2026-01-09T16:03:43+07:00
weight: 95
chapter: false
pre: " <b> 5.5.2. </b> "
---

# 5.5.2 DynamoDB — Bảng, Index, Cấu trúc Item

Mỗi `a.model` trong `amplify/data/resource.ts` được biên dịch thành một bảng DynamoDB. Amplify thêm hậu tố đặc thù môi trường để bảo đảm cô lập giữa các sandbox. Mọi bảng NutriTrack có chung các thuộc tính:

- **Billing mode**: on-demand (trả theo request). Dev không cần lập kế hoạch capacity.
- **Mã hoá**: KMS key do AWS sở hữu theo mặc định.
- **Point-in-time recovery**: tắt trong sandbox, khuyến nghị bật cho `main`.
- **Stream**: bật (Amplify tự bật để subscription qua AppSync hoạt động).

## Quy ước đặt tên theo môi trường

Amplify đặt tên bảng theo dạng `{ModelName}-{apiId}-{branch}`. Tên thực tế trong dự án:

| Môi trường | Suffix API | Tên bảng ví dụ |
| --- | --- | --- |
| Sandbox (`npx ampx sandbox`) | `tynb5fej6jeppdrgxizfiv4l3m` | `Food-tynb5fej6jeppdrgxizfiv4l3m-NONE` |
| Branch `feat/phase3` | `vic4ri35gbfpvnw5nw3lkyapki` | `Food-vic4ri35gbfpvnw5nw3lkyapki-NONE` |
| Branch `main` | `2c73cq2usbfgvp7eaihsupyjwe` | `Food-2c73cq2usbfgvp7eaihsupyjwe-NONE` |

Hậu tố `-NONE` là placeholder của Amplify cho "no environment type". Sandbox dùng hậu tố này, production branch cũng vậy. Đừng bao giờ hardcode tên bảng trong Lambda — hoặc đọc từ env var inject trong `backend.ts`, hoặc discover động (xem 5.5.3 để có ví dụ thực tế).

## 8 bảng

### 1. `Food-*` — catalog món Việt dùng chung

- **Partition key**: `food_id` (String)
- **Sort key**: không có
- **GSI**: không
- **Auth**: guest đọc, authenticated đọc
- **Số item**: ~200 (seed một lần, ít update)
- **Ai truy cập**: resolver AppSync trực tiếp (đọc), Lambda `process-nutrition` (scan + đọc)

Item ví dụ:

```json
{
  "food_id": "phobo_001",
  "name_vi": "Phở bò tái",
  "name_en": "Beef noodle soup (rare)",
  "aliases_vi": ["phở tái", "phở bò"],
  "aliases_en": ["pho bo tai"],
  "macros": {
    "calories": 420,
    "protein_g": 28,
    "carbs_g": 48,
    "fat_g": 12,
    "fiber_g": 2,
    "sodium_mg": 980
  },
  "micronutrients": { "iron_mg": 3.2, "calcium_mg": 45 },
  "serving": {
    "default_g": 500,
    "unit": "bowl",
    "portions": { "small": 0.7, "medium": 1.0, "large": 1.3 }
  },
  "verified": true,
  "source": "seed"
}
```

### 2. `user-*` — profile

- **Partition key**: `user_id` (String) — bằng Cognito `sub`
- **GSI**: `user-friend_code-index` trên `friend_code` (String)
- **Auth**: owner
- **Ai truy cập**: resolver AppSync, Lambda `friend-request` (đọc qua GSI để resolve friend code)

Item nhúng 5 custom type map (`biometric`, `goal`, `dietary_profile`, `gamification`, `ai_preferences`) cộng với attribute `owner` do Cognito điền.

### 3. `FoodLog-*` — lịch sử bữa ăn

- **Partition key**: `id` (tự sinh qua `a.id()`)
- **GSI**: `FoodLog-date-index` trên `date` — là partition key của GSI
- **Auth**: owner
- **Ai truy cập**: chỉ resolver AppSync

Đây là bảng nóng nhất hệ thống. Mỗi lần mở màn hình ngày chạy `listFoodLogByDate(date: "2026-04-05")`, nó thực hiện `Query` trên GSI `date-index` và ngầm lọc theo `owner`. Chi phí tăng tuyến tính theo số log, không theo số user.

### 5. `FridgeItem-*` — tồn kho tủ lạnh

- **Partition key**: `id`
- **GSI**: không
- **Auth**: owner
- **Ai truy cập**: resolver AppSync, Lambda `ai-engine` gián tiếp (đọc item qua AppSync trước khi gọi `generateRecipe`)

### 5. `Challenge-*` — thử thách nhóm

- **Partition key**: `id`
- **GSI**: không
- **Auth**: authenticated (đọc/ghi public trong app)
- **Quan hệ**: `hasMany` tới `ChallengeParticipant`

### 6. `ChallengeParticipant-*` — bảng join

- **Partition key**: `id`
- **GSI**: `ChallengeParticipant-user_id-index` trên `user_id`
- **Auth**: authenticated
- **Query phổ biến**: "Tôi đang tham gia challenge nào?" — Query trên GSI `user_id`.

### 7. `Friendship-*` — đồ thị bạn bè hai chiều

- **Partition key**: `id`
- **GSI**: `Friendship-friend_id-index` trên `friend_id`
- **Auth**: owner
- **Ai truy cập**: resolver AppSync, Lambda `friend-request` (tạo/cập nhật cặp row nguyên tử qua `TransactWriteItems`)

Một tình bạn được lưu thành **hai** row — một do user A sở hữu với `direction: 'sent'`, một do user B sở hữu với `direction: 'received'` — vì `allow.owner()` sẽ chặn B thấy row của A. `linked_id` liên kết cặp này.

### 8. `UserPublicStats-*` — view leaderboard

- **Partition key**: `user_id`
- **GSI**: không
- **Auth**: owner ghi, authenticated đọc
- **Ai truy cập**: chỉ resolver AppSync

Bản tóm tắt denormalized được owner ghi mỗi lần log. Các user authenticated khác có thể đọc để đổ leaderboard mà không xâm phạm bảng `user` riêng tư.

## Kiểm tra bằng CLI

```bash
# Liệt kê tất cả bảng của dự án
aws dynamodb list-tables --region ap-southeast-2 \
  | jq -r '.TableNames[] | select(startswith("Food-"))'

# Xem vài row
aws dynamodb scan \
  --table-name Food-tynb5fej6jeppdrgxizfiv4l3m-NONE \
  --max-items 5 \
  --region ap-southeast-2

# Mô tả GSI trên FoodLog
aws dynamodb describe-table \
  --table-name FoodLog-tynb5fej6jeppdrgxizfiv4l3m-NONE \
  --region ap-southeast-2 \
  --query 'Table.GlobalSecondaryIndexes'
```

![DynamoDB tables list](/images/workshop/dynamodb-tables-list.png)

![Food item structure](/images/workshop/food-item-structure.png)

## Mô hình chi phí

Mặc định mọi bảng đều on-demand. Với workload NutriTrack:

- **Food**: gần như miễn phí. ~200 item, traffic read-only, vừa vặn cho DAX hoặc cache client.
- **user / UserPublicStats**: một row/user, một lần write mỗi khi profile đổi đáng kể. Chi phí không đáng kể.
- **FoodLog**: chiếm chi phí chính. Ước lượng ~5 write và ~20 read mỗi user hoạt động/ngày. Với 1.000 DAU thì ~5k write và ~20k read/ngày — vẫn dưới free tier.
- **Friendship / ChallengeParticipant**: burst nhỏ, khối lượng nhỏ.

Với production trên branch `main`, cân nhắc chuyển bảng `FoodLog` sang provisioned capacity + autoscaling khi traffic ổn định — on-demand đắt gấp ~7 lần provisioned được tận dụng đủ.

## Seed catalog Food

Bảng Food được seed một lần với ~200 món Việt đã curate. Script seed dùng `BatchWriteItem`:

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, BatchWriteCommand } from '@aws-sdk/lib-dynamodb';
import seed from './seed/vietnamese-foods.json';

const client = DynamoDBDocumentClient.from(
  new DynamoDBClient({ region: 'ap-southeast-2' })
);
const TABLE = process.env.FOOD_TABLE_NAME!;

for (let i = 0; i < seed.length; i += 25) {
  const batch = seed.slice(i, i + 25).map((item) => ({
    PutRequest: { Item: { ...item, owner: 'SYSTEM' } },
  }));
  await client.send(new BatchWriteCommand({
    RequestItems: { [TABLE]: batch },
  }));
  console.log(`Đã seed ${i + batch.length}/${seed.length}`);
}
```

Chạy một lần mỗi môi trường sau khi `npx ampx sandbox` provision xong bảng. `BatchWriteItem` giới hạn 25 item/request, nên vòng lặp chia chunk đúng mức.

## Xử lý sự cố

- **"ValidationException: One or more parameter values were invalid"** khi scan — có thể bạn đụng reserved word (`date`, `status`, `name`). Dùng `ExpressionAttributeNames`.
- **Cảnh báo hot partition trên CloudWatch** — khó xảy ra trong dev, nhưng nếu `FoodLog` lệch về một ngày có hàng triệu row, shard partition key.
- **GSI read eventually consistent** — GSI `date-index` trên FoodLog là async. Ngay sau create, row có thể chưa xuất hiện ở lần list kế tiếp. Frontend nên insert optimistic local.

## Liên kết

- Schema sinh ra các bảng: [5.5.1 AppSync](../5.5.1-AppSync/)
- Lambda truy cập bảng trực tiếp: [5.5.3 ProcessNutrition](../../5.5-Processing-Setup/5.5.3-ProcessNutrition/)
