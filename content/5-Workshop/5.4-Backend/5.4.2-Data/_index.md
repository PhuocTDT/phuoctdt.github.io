---
title: "Data Layer (Data)"
date: 2026-01-09T16:03:43+07:00
weight: 42
chapter: false
pre: " <b> 5.4.2. </b> "
---

# 5.4.2 Data Layer (Data) — AppSync & DynamoDB

Defining schema and access rules.

## `data/resource.ts`

```typescript
import { type ClientSchema, a, defineData } from "@aws-amplify/backend";

const schema = a.schema({
  Food: a.model({
    name: a.string().required(),
    calories: a.float(),
    protein: a.float(),
    carbs: a.float(),
    fat: a.float(),
    category: a.string(),
  }).authorization((allow) => [allow.authenticated()]),

  User: a.model({
    height: a.float(),
    weight: a.float(),
    goalCalories: a.float(),
    petLevel: a.integer(),
  }).authorization((allow) => [allow.owner()]),
});

export type Schema = ClientSchema<typeof schema>;
export const data = defineData({
  schema,
  authorizationModes: {
    defaultAuthorizationMode: "userPool",
  },
});
```
![appsync-console-schema.png](/images/appsync-console-schema.png)
![appsync-queries-playground.png](/images/appsync-queries-playground.png)
![food-item-structure.png](/images/food-item-structure.png)
![dynamodb-tables-list.png](/images/dynamodb-tables-list.png)

---

[Continue to 5.4.3 Storage Layer (Storage)](../5.4.3-Storage/)
