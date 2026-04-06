---
title: "AppSync"
date: 2026-01-09T16:03:43+07:00
weight: 100
chapter: false
pre: " <b> 5.5.1. </b> "
---

# 5.5.1 AppSync — GraphQL Schema & Custom Resolver

AppSync là tầng GraphQL managed của NutriTrack. Thay vì viết tay VTL template hay resolver pipeline, chúng ta khai báo toàn bộ trong `amplify/data/resource.ts` bằng DSL `a.schema(...)` của Amplify. Khi chạy `npx ampx sandbox` (hoặc khi push lên branch), CLI biên dịch file đó thành một AppSync API hoàn chỉnh, các bảng DynamoDB, IAM role và các wiring Lambda tương ứng.

Trang này đi qua từng khối của `data/resource.ts` thật.

## `defineData` nối dây như thế nào

Ở cuối file là:

```typescript
export const data = defineData({
  schema,
  authorizationModes: {
    defaultAuthorizationMode: 'userPool',
  },
});
```

Ba điều xảy ra:

1. `defaultAuthorizationMode: 'userPool'` yêu cầu mọi GraphQL operation phải kèm Cognito ID token (từ 5.3.2). Token được parse trên mỗi request để lấy `sub`, `email`, `cognito:groups` — `sub` trở thành trường `owner` ngầm cho mọi model dùng `allow.owner()`.
2. Đối tượng `schema` phía trên là một bản đồ lớn gồm model, custom type, query, mutation qua `a.schema({ ... })`.
3. Amplify sinh resolver CRUD + subscription cho mỗi `a.model` và nối Lambda cho mỗi `a.query`/`a.mutation` có `.handler(a.handler.function(...))`.

## Các khối trong `a.schema(...)`

### 1. `a.customType(...)` — object lồng nhau, không có bảng

Custom type tạo ra GraphQL type nhưng **không tạo bảng DynamoDB**. Chúng được nhúng trong item cha dưới dạng `Map` attribute. Ví dụ:

```typescript
Macros: a.customType({
  calories: a.float(),
  protein_g: a.float(),
  carbs_g: a.float(),
  fat_g: a.float(),
  saturated_fat_g: a.float(),
  polyunsaturated_fat_g: a.float(),
  monounsaturated_fat_g: a.float(),
  fiber_g: a.float(),
  sugar_g: a.float(),
  sodium_mg: a.float(),
  cholesterol_mg: a.float(),
  potassium_mg: a.float(),
}),
```

Rồi `Food.macros = a.ref('Macros')` sẽ nhúng cấu trúc này inline. Danh sách customType trong NutriTrack:

- **Food**: `Portions`, `Serving`, `Micronutrients`, `Macros`.
- **Log**: `LogMacros`, `LogIngredient`.
- **User**: `biometric`, `goal`, `dietary_profile`, `gamification`, `ai_preferences`.

Tổng cộng 11 custom type (cộng với 8 model), tất cả nằm trong cùng một schema.

### 2. `a.model(...)` — bảng DynamoDB

Dưới đây là model `Food` đầy đủ — catalog ~200 món Việt, guest được đọc:

```typescript
Food: a
  .model({
    food_id: a.string().required(),
    name_vi: a.string().required(),
    name_en: a.string(),
    aliases_vi: a.string().array(),
    aliases_en: a.string().array(),
    macros: a.ref('Macros'),
    micronutrients: a.ref('Micronutrients'),
    serving: a.ref('Serving'),
    verified: a.boolean(),
    source: a.string(),
  })
  .identifier(['food_id'])
  .authorization((allow) => [
    allow.guest().to(['read']),
    allow.authenticated().to(['read'])
  ]),
```

Lưu ý:

- `.identifier(['food_id'])` ghi đè partition key mặc định `id` bằng chuỗi `food_id` tự chọn.
- `.array()` trở thành `List<String>` trong DynamoDB.
- `a.ref('Macros')` nhúng custom type Macros dưới dạng map attribute.
- `allow.guest().to(['read'])` kích hoạt IAM-auth mode song song với Cognito, cho phép user chưa đăng nhập (Cognito Identity Pool guest) đọc catalog.

### 3. `a.model(...)` với owner auth — bảng user

```typescript
user: a
  .model({
    user_id: a.string().required(),
    email: a.string().required(),
    display_name: a.string(),
    avatar_url: a.string(),
    created_at: a.string(),
    updated_at: a.string(),
    last_active_at: a.string(),
    onboarding_status: a.boolean(),
    friend_code: a.string(),
    ai_context_summary: a.string(),
    biometric: a.ref('biometric'),
    goal: a.ref('goal'),
    dietary_profile: a.ref('dietary_profile'),
    gamification: a.ref('gamification'),
    ai_preferences: a.ref('ai_preferences'),
  })
  .identifier(['user_id'])
  .secondaryIndexes((index) => [
    index('friend_code'),
  ])
  .authorization((allow) => [
    allow.owner(),
  ]),
```

Điểm quan trọng:

- `allow.owner()` tự thêm trường `owner: String!` khi ghi, giá trị là Cognito `sub` của caller. Mọi `list`/`get`/`update`/`delete` sau đó được lọc theo trường này. User **không thể** thấy row của người khác.
- `.secondaryIndexes((index) => [index('friend_code')])` tạo GSI trên `friend_code`, xuất hiện trong GraphQL là `listUserByFriendCode(friendCode: ...)`. Đây là cách friend request đổi mã 6 ký tự sang user.
- Các map lồng `biometric`, `goal`, `dietary_profile`, `gamification`, `ai_preferences` đều là customType.

### 5. Model có GSI — `FoodLog`

```typescript
FoodLog: a
  .model({
    date: a.string().required(),
    timestamp: a.datetime().required(),
    food_id: a.string(),
    food_name: a.string().required(),
    meal_type: a.enum(['breakfast', 'lunch', 'dinner', 'snack']),
    portion: a.float(),
    portion_unit: a.string(),
    additions: a.string().array(),
    ingredients: a.json().array(),
    macros: a.ref('LogMacros'),
    micronutrients: a.ref('Micronutrients'),
    input_method: a.enum(['voice', 'photo', 'manual', 'barcode']),
    image_key: a.string(),
  })
  .secondaryIndexes((index) => [
    index('date'),
  ])
  .authorization((allow) => [
    allow.owner(),
  ]),
```

`index('date')` là GSI quan trọng nhất của NutriTrack — mọi query của màn hình Home đều là "liệt kê log hôm nay của tôi", tức `client.models.FoodLog.listFoodLogByDate({ date: '2026-04-05' })`. Không có GSI này thì query trở thành full-table scan trên mỗi user.

Các model có GSI:

- `user` — `friend_code`
- `FoodLog` — `date`
- `ChallengeParticipant` — `user_id`
- `Friendship` — `friend_id`

### 5. Quan hệ — `hasMany` / `belongsTo`

Challenge dùng quan hệ 1-nhiều cổ điển:

```typescript
Challenge: a
  .model({
    creator_id: a.string().required(),
    challenge_type: a.enum(['calories', 'protein', 'steps', 'streak', 'custom']),
    title: a.string().required(),
    description: a.string(),
    target_value: a.float(),
    status: a.enum(['pending', 'active', 'completed', 'expired']),
    start_date: a.string().required(),
    end_date: a.string().required(),
    participants: a.hasMany('ChallengeParticipant', 'challengeId'),
  })
  .authorization((allow) => [allow.authenticated()]),

ChallengeParticipant: a
  .model({
    challengeId: a.id().required(),
    challenge: a.belongsTo('Challenge', 'challengeId'),
    user_id: a.string().required(),
    display_name: a.string(),
    progress: a.float(),
    joined_at: a.datetime(),
  })
  .secondaryIndexes((index) => [index('user_id')])
  .authorization((allow) => [allow.authenticated()]),
```

`allow.authenticated()` nghĩa là mọi user đã đăng nhập đều đọc/ghi được — challenge là public trong app. Quan hệ sinh ra trường `Challenge.participants` mà AppSync resolve qua sub-query trên bảng `ChallengeParticipant` lọc theo `challengeId`.

### 6. Auth hỗn hợp — `UserPublicStats` cho leaderboard

```typescript
UserPublicStats: a
  .model({
    user_id: a.string().required(),
    display_name: a.string(),
    avatar_url: a.string(),
    current_streak: a.integer(),
    longest_streak: a.integer(),
    pet_score: a.integer(),
    pet_level: a.integer(),
    total_log_days: a.integer(),
    last_log_date: a.string(),
  })
  .identifier(['user_id'])
  .authorization((allow) => [
    allow.owner().to(['create', 'update', 'delete', 'read']),
    allow.authenticated().to(['read']),
  ]),
```

Hai rule xếp chồng:

- Owner có đủ CRUD trên row của mình.
- User đã đăng nhập bất kỳ đều `read` được — đây là cách leaderboard bạn bè hoạt động mà không lộ trường riêng tư từ bảng `user`.

### 7. Custom resolver — `a.query` / `a.mutation` gắn Lambda

```typescript
aiEngine: a
  .query()
  .arguments({
    action: a.string().required(),
    payload: a.string(),
  })
  .returns(a.string())
  .handler(a.handler.function(aiEngine))
  .authorization((allow) => [allow.authenticated()]),

processNutrition: a
  .query()
  .arguments({ payload: a.string().required() })
  .returns(a.string())
  .handler(a.handler.function(processNutrition))
  .authorization((allow) => [allow.authenticated()]),

friendRequest: a
  .mutation()
  .arguments({
    action: a.string().required(),
    payload: a.string().required(),
  })
  .returns(a.string())
  .handler(a.handler.function(friendRequest))
  .authorization((allow) => [allow.authenticated()]),
```

`.handler(a.handler.function(aiEngine))` gắn direct Lambda data source. AppSync bỏ qua DynamoDB hoàn toàn với ba operation này và gọi Lambda đồng bộ, truyền `event.arguments` bằng input args. Lambda trả về string (chúng ta cố tình stringify JSON để giữ schema GraphQL đơn giản — xem 5.5.2).

## Sử dụng ở frontend

Amplify JS client được type-hoá đầy đủ từ schema:

```typescript
import { generateClient } from 'aws-amplify/api';
import type { Schema } from '@/amplify/data/resource';

const client = generateClient<Schema>();

// Liệt kê log của hôm nay (dùng GSI `date`)
const { data: logs } = await client.models.FoodLog.listFoodLogByDate({
  date: '2026-04-05',
});

// Tạo một log
await client.models.FoodLog.create({
  date: '2026-04-05',
  timestamp: new Date().toISOString(),
  food_name: 'Phở bò',
  meal_type: 'lunch',
  portion: 1,
});

// Gọi Lambda ai-engine qua custom query
const result = await client.queries.aiEngine({
  action: 'analyzeFoodImage',
  payload: JSON.stringify({ s3Key: 'incoming/user-abc/img-1.jpg' }),
});
```

Type được suy đầy đủ end-to-end: đổi tên field trong `resource.ts` sẽ tạo lỗi biên dịch cho toàn bộ React Native app ở lần chạy `npx ampx sandbox` tiếp theo, nơi `amplify_outputs.json` và type client TypeScript được regen.

## Subscription real-time

Mọi model đều có `onCreate`, `onUpdate`, `onDelete` miễn phí:

```typescript
const sub = client.models.FoodLog.onCreate({
  filter: { date: { eq: '2026-04-05' } },
}).subscribe({
  next: (log) => {
    console.log('Log mới xuất hiện:', log);
  },
});
```

Đây là nền tảng cho multi-device sync — khi điện thoại log một bữa, tablet hiển thị dashboard cùng user sẽ thấy row xuất hiện trong ~200 ms. Subscription được dùng ở 5.6.2 để leaderboard challenge cập nhật live.

## Kiểm tra API sau deploy

Sau khi `npx ampx sandbox` xong, AppSync console hiển thị:

- Tab **Schema** — SDL GraphQL đã nở (lớn hơn `resource.ts` nhiều vì Amplify sinh input/filter/connection type).
- Tab **Queries** — playground. Paste query, gắn Cognito token từ Amplify Studio, chạy.
- Tab **Data sources** — một entry `AMAZON_DYNAMODB` cho mỗi model, và ba entry `AWS_LAMBDA` cho các custom resolver.

![AppSync console schema view](/images/workshop/appsync-console-schema.png)

![AppSync queries playground](/images/workshop/appsync-queries-playground.png)

## Xử lý sự cố

- **`Unauthorized` trên `list`/`get`**: owner không khớp. Trường `owner` của row không bằng `sub` của caller. Kiểm tra item trên DynamoDB và so với JWT.
- **Thay đổi schema không thấy**: phải chạy lại `npx ampx sandbox` (hoặc push branch). Amplify không hot-reload mutation schema.
- **`Validation error: field X not found`**: frontend đang dùng client sinh ra đã cũ. Xoá `amplify_outputs.json`, restart sandbox, import lại.
- **Thiếu GSI**: nếu bạn thêm `.secondaryIndexes(...)` sau lần deploy đầu, CLI sẽ chạy `UpdateTable` — với bảng lớn có thể mất vài phút. Theo dõi CloudFormation events.

## Liên kết

- Bảng và cấu trúc item: [5.5.2 DynamoDB](../5.5.2-DynamoDB/)
- Lambda resolver: [5.5.2 AIEngine](../../5.5-Processing-Setup/5.5.2-AIEngine/), [5.5.3 ProcessNutrition](../../5.5-Processing-Setup/5.5.3-ProcessNutrition/)
- Subscription thực chiến: [5.6 Automation Setup](../../5.6-Automation-Setup/)
