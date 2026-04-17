---
title: "Lớp Dữ liệu (Data)"
menuTitle: "Lớp Dữ liệu (Data)"
date: 2026-01-09T16:03:43+07:00
weight: 42
chapter: false
pre: " <b> 5.4.2. </b> "
---

# AppSync & DynamoDB

Định nghĩa lược đồ dữ liệu (Schema) và các quy tắc truy cập.

## Cấu hình `data/resource.ts`

Chúng ta định nghĩa các model chính phục vụ cho ứng dụng.

```typescript
import { type ClientSchema, a, defineData } from '@aws-amplify/backend';
import { aiEngine } from '../ai-engine/resource';
import { scanImage } from '../scan-image/resource';
import { processNutrition } from '../process-nutrition/resource';
import { friendRequest } from '../friend-request/resource';


const schema = a.schema({
  //========================================
  // Food Database
  //========================================
  Portions: a.customType({
    small: a.float(),
    medium: a.float(),
    large: a.float(),
  }),

  Serving: a.customType({
    default_g: a.float(),
    unit: a.string(),
    portions: a.ref('Portions'),
  }),

  Micronutrients: a.customType({
    calcium_mg: a.float(),
    iron_mg: a.float(),
    vitamin_a_ug: a.float(),
    vitamin_c_mg: a.float(),
  }),

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
    .secondaryIndexes((index) => [
      index('name_vi'),
      index('name_en'),
    ])
    .authorization((allow) => [
      allow.guest().to(['read']),
      allow.authenticated().to(['read'])
    ]),

  //========================================
  // User Database
  //========================================
  biometric: a.customType({
    age: a.integer(),
    gender: a.string(),
    height_cm: a.float(),
    weight_kg: a.float(),
    active_level: a.string(),
  }),

  goal: a.customType({
    daily_calories: a.float(),
    daily_carbs_g: a.float(),
    daily_protein_g: a.float(),
    daily_fat_g: a.float(),
    target_weight_kg: a.float(), 
  }),

  dietary_profile: a.customType({
    allergies: a.string().array(),
    preferences: a.string().array(),
  }),

  gamification: a.customType({
    current_streak: a.integer(),
    longest_streak: a.integer(),
    last_log_date: a.string(),
    total_points: a.integer(),
  }),

  ai_preferences: a.customType({
    coach_tone: a.string(),
  }),

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


  //========================================
  // Food Logs (Meal History)
  //========================================
  LogMacros: a.customType({
    calories: a.float(),
    protein_g: a.float(),
    carbs_g: a.float(),
    fat_g: a.float(),
    fiber_g: a.float(),
    sugar_g: a.float(),
    sodium_mg: a.float(),
  }),

  LogIngredient: a.customType({
    name: a.string(),
    weight_g: a.float(),
  }),

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

  //========================================
  // Fridge Inventory
  //========================================
  FridgeItem: a
    .model({
      name: a.string().required(),
      food_id: a.string(),
      quantity: a.float(),
      unit: a.string(),
      added_date: a.datetime(),
      expiry_date: a.string(),
      category: a.enum(['meat', 'vegetable', 'fruit', 'dairy', 'pantry', 'other']),
      emoji: a.string(),
      calories: a.float(),
      protein_g: a.float(),
      carbs_g: a.float(),
      fat_g: a.float(),
    })
    .authorization((allow) => [
      allow.owner(),
    ]),

//========================================
  // Friendships
  //========================================
  Friendship: a
    .model({
      friend_id: a.string().required(),
      friend_code: a.string(),
      friend_name: a.string(),
      friend_avatar: a.string(),
      status: a.enum(['pending', 'accepted', 'blocked']),
      direction: a.enum(['sent', 'received']),
      linked_id: a.string(),
    })
    .secondaryIndexes((index) => [
      index('friend_id'),
    ])
    .authorization((allow) => [
      allow.owner(),
    ]),

  //========================================
  // User Public Stats (readable by friends)
  //========================================
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

  //========================================
  // AI Engine (Bedrock)
  //========================================
  aiEngine: a
    .query()
    .arguments({
      action: a.string().required(),
      payload: a.string(),
    })
    .returns(a.string())
    .handler(a.handler.function(aiEngine))
    .authorization((allow) => [allow.authenticated()]),

  //========================================
  // Scan Image (proxy to ECS for photo analysis)
  //========================================
  scanImage: a
    .query()
    .arguments({
      action: a.string().required(),
      payload: a.string(),
    })
    .returns(a.string())
    .handler(a.handler.function(scanImage))
    .authorization((allow) => [allow.authenticated()]),

  //========================================
  // Process Nutrition (DB verify + AI fallback)
  //========================================
  processNutrition: a
    .query()
    .arguments({ payload: a.string().required() })
    .returns(a.string())
    .handler(a.handler.function(processNutrition))
    .authorization((allow) => [allow.authenticated()]),

  //========================================
  // Friend Request (send/accept/decline/remove/block)
  //========================================
  friendRequest: a
    .mutation()
    .arguments({
      action: a.string().required(),
      payload: a.string().required(),
    })
    .returns(a.string())
    .handler(a.handler.function(friendRequest))
    .authorization((allow) => [allow.authenticated()]),
});

export type Schema = ClientSchema<typeof schema>;

export const data = defineData({
  schema,
  authorizationModes: {
    defaultAuthorizationMode: 'userPool',
  },
});
```

![appsync-console-schema.png](/images/appsync-console-schema.png)
![appsync-queries-playground.png](/images/appsync-queries-playground.png)

![food-item-structure.png](/images/food-item-structure.png)
![dynamodb-tables-list.png](/images/dynamodb-tables-list.png)

---

[Tiếp tục đến 5.4.3 Lớp Lưu trữ (Storage)](../5.4.3-Storage/)
