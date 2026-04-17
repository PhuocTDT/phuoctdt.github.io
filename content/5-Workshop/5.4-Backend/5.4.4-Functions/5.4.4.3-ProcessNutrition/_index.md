---
title: "Function process-nutrition"
menuTitle: "5.4.4.3. Function process-nutrition"
weight: 3
---

This function handles the calculation of nutritional macros and stores them directly into the **DynamoDB** table.

### 1. Install Libraries (npm install)

This function requires the DynamoDB Client SDK:

```bash
cd amplify/process-nutrition
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

### 2. Resource Configuration (`resource.ts`)

```typescript
import { defineFunction } from '@aws-amplify/backend';

export const processNutrition = defineFunction({
  name: 'process-nutrition',
  entry: './handler.ts',
  runtime: 22,
  memoryMB: 256,
  timeoutSeconds: 10,
});
```

### 3. Handler Logic (`handler.ts`)

```typescript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

export const handler = async (event: any) => {
  const { userId, foodName, calories, protein, carbs, fat } = event.arguments;

  const params = {
    TableName: process.env.NUTRITION_TABLE_NAME,
    Item: {
      userId,
      foodName,
      calories,
      protein,
      carbs,
      fat,
      createdAt: new Date().toISOString(),
    },
  };

  try {
    await docClient.send(new PutCommand(params));
    return { success: true };
  } catch (error) {
    console.error(error);
    return { success: false, error: "Database error" };
  }
};
```

---

[Back to functions list](../)
