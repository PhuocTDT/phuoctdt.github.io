---
title: "Function scan-image"
menuTitle: "5.4.4.2. Function scan-image"
weight: 2
---

This function acts as a proxy (intermediary) to send image analysis requests from AppSync to the **ECS Fargate** cluster via an Application Load Balancer.

### 1. Resource Configuration (`resource.ts`)

```typescript
import { defineFunction } from '@aws-amplify/backend';

export const scanImage = defineFunction({
  name: 'scan-image',
  entry: './handler.ts',
  runtime: 22,
  memoryMB: 1024,
  timeoutSeconds: 300,
});
```

### 2. Handler Logic (`handler.ts`)

```typescript
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";
import { SecretsManagerClient, GetSecretValueCommand } from "@aws-sdk/client-secrets-manager";
import { createHmac } from "crypto";

const REGION = "ap-southeast-2";
const s3Client = new S3Client({ region: REGION });
const secretsClient = new SecretsManagerClient({ region: REGION });

const STORAGE_BUCKET = process.env.STORAGE_BUCKET_NAME || "";
const ECS_BASE_URL = process.env.ECS_BASE_URL || "";
const IS_DEBUG = process.env.DEBUG === "true" || process.env.NODE_ENV === "development";

const debug = (message: string, data?: any) => {
  if (IS_DEBUG) {
    console.log(`[scan-image] ${message}`, data || "");
  }
};

// Cache secrets across warm Lambda invocations
let cachedSecrets: { apiKey: string; internalToken: string } | null = null;

async function getSecrets(): Promise<{ apiKey: string; internalToken: string }> {
  if (cachedSecrets) return cachedSecrets;

  const resp = await secretsClient.send(
    new GetSecretValueCommand({ SecretId: "nutritrack/prod/api-keys" })
  );
  const secret = JSON.parse(resp.SecretString || "{}");
  cachedSecrets = {
    apiKey: secret.NUTRITRACK_API_KEY || "",
    internalToken: secret.NUTRITRACK_INTERNAL_TOKEN || "",
  };
  return cachedSecrets;
}

// Generate HS256 JWT using Node.js built-in crypto (no external JWT library needed)
function generateJWT(secret: string): string {
  const b64url = (buf: Buffer) =>
    buf.toString("base64").replace(/\+/g, "-").replace(/\//g, "_").replace(/=/g, "");

  const header = b64url(Buffer.from(JSON.stringify({ alg: "HS256", typ: "JWT" })));
  const now = Math.floor(Date.now() / 1000);
  const payload = b64url(
    Buffer.from(JSON.stringify({ service: "backend", iat: now, exp: now + 300 }))
  );
  const data = `${header}.${payload}`;
  const sig = b64url(createHmac("sha256", secret).update(data).digest());
  return `${data}.${sig}`;
}

const ACTION_TO_ENDPOINT: Record<string, string> = {
  analyzeFoodImage: "/analyze-food?method=tools",
  analyzeFoodLabel: "/analyze-label",
  scanBarcode: "/scan-barcode",
};

// Poll /jobs/{jobId} until completed or failed
async function pollJob(
  jobId: string,
  authHeader: string,
  internalToken: string,
  signal: AbortSignal,
  stickyCookie?: string,
  pollIntervalMs = 3000,
  maxWaitMs = 270_000
): Promise<any> {
  const pollUrl = `${ECS_BASE_URL}/jobs/${jobId}`;
  const start = Date.now();

  while (true) {
    if (Date.now() - start > maxWaitMs) {
      throw new Error("Job polling timeout — analysis took too long");
    }

    // Wait before polling
    await new Promise<void>((resolve) => {
      const t = setTimeout(resolve, pollIntervalMs);
      signal.addEventListener("abort", () => { clearTimeout(t); resolve(); }, { once: true });
    });

    if (signal.aborted) throw new Error("AbortError");

    const pollHeaders: Record<string, string> = {
      Authorization: authHeader,
      "X-Nutri-Internal-Token": internalToken,
    };
    if (stickyCookie) pollHeaders["Cookie"] = stickyCookie;

    const resp = await fetch(pollUrl, {
      headers: pollHeaders,
      signal,
    });

    if (!resp.ok) {
      const body = await resp.text();
      throw new Error(`Job poll failed (${resp.status}): ${body}`);
    }

    const job = await resp.json();
    debug("Poll result", { status: job.status });

    if (job.status === "completed") {
      return job.result;
    }

    if (job.status === "failed") {
      throw new Error(job.error || "ECS job failed");
    }
    // status === "processing" → keep polling
  }
}

interface GENFOODMacros {
  calories: number;
  protein_g: number;
  carbs_g: number;
  fat_g: number;
  saturated_fat_g: number;
  polyunsaturated_fat_g: number;
  monounsaturated_fat_g: number;
  fiber_g: number;
  sugar_g: number;
  sodium_mg: number;
  cholesterol_mg: number;
  potassium_mg: number;
}

interface GENFOODMicronutrients {
  calcium_mg: number;
  iron_mg: number;
  vitamin_a_ug: number;
  vitamin_c_mg: number;
}

interface GENFOODServing {
  default_g: number;
  unit: string;
  portions?: Record<string, number>;
}

interface GENFOODIngredient {
  name_vi: string;
  name_en: string;
  weight_g: number;
}

interface GENFOOD {
  food_id: string;
  name_vi: string;
  name_en: string;
  macros: GENFOODMacros;
  micronutrients: GENFOODMicronutrients;
  serving: GENFOODServing;
  ingredients: GENFOODIngredient[];
  verified: boolean;
  source: string;
}

const EMPTY_MACROS = {
  saturated_fat_g: 0, polyunsaturated_fat_g: 0, monounsaturated_fat_g: 0,
  fiber_g: 0, sugar_g: 0, sodium_mg: 0, cholesterol_mg: 0, potassium_mg: 0,
};
const EMPTY_MICRONUTRIENTS = { calcium_mg: 0, iron_mg: 0, vitamin_a_ug: 0, vitamin_c_mg: 0 };

// Food photo: job.result.data.dishes[0]
function normalizeFoodToGENFOOD(ecsResult: any): GENFOOD {
  const dish = ecsResult?.data?.dishes?.[0];
  if (!dish) throw new Error("No food detected in image");

  const n = dish.nutritions || {};
  return {
    food_id: "custom_gen_temp",
    name_vi: dish.vi_name || dish.name || "Unknown Food",
    name_en: dish.name || "Unknown Food",
    macros: {
      calories: Number(n.calories) || 0,
      protein_g: Number(n.protein) || 0,
      carbs_g: Number(n.carbs) || 0,
      fat_g: Number(n.fat) || 0,
      ...EMPTY_MACROS,
    },
    micronutrients: EMPTY_MICRONUTRIENTS,
    serving: {
      default_g: Number(dish.weight || dish.serving_value) || 0,
      unit: dish.serving_unit || "serving",
      portions: { small: 0.7, medium: 1.0, large: 1.3 },
    },
    ingredients: (dish.ingredients || []).map((ing: any) => ({
      name_vi: ing.name || "Ingredient",
      name_en: ing.name || "Ingredient",
      weight_g: Number(ing.weight) || 0,
    })),
    verified: false,
    source: "ECS Scan",
  };
}

// Nutrition label: job.result.data.labels[0]
// label.nutrition = [{ nutrient: "Chất đạm", value: 6.9, unit: "g" }, ...]
function normalizeLabelToGENFOOD(ecsResult: any): GENFOOD {
  const label = ecsResult?.data?.labels?.[0];
  if (!label) throw new Error("No nutrition label detected in image");

  const nutrition: { nutrient: string; value: number }[] = label.nutrition || [];
  const findNutrient = (...keywords: string[]) => {
    const item = nutrition.find((it) =>
      keywords.some((k) => it.nutrient.toLowerCase().includes(k))
    );
    return Number(item?.value) || 0;
  };

  return {
    food_id: "custom_label_temp",
    name_vi: label.name || "Sản phẩm",
    name_en: label.name || "Product",
    macros: {
      calories: findNutrient("năng lượng", "energy", "calori", "kcal"),
      protein_g: findNutrient("đạm", "protein"),
      carbs_g: findNutrient("carbohydrate", "glucid", "tinh bột", "đường bột"),
      fat_g: findNutrient("béo", "fat", "lipid"),
      ...EMPTY_MACROS,
    },
    micronutrients: EMPTY_MICRONUTRIENTS,
    serving: {
      default_g: Number(label.serving_value) || 0,
      unit: label.serving_unit || "g",
      portions: { small: 0.7, medium: 1.0, large: 1.3 },
    },
    ingredients: (label.ingredients || []).map((name: string) => ({
      name_vi: name,
      name_en: name,
      weight_g: 0,
    })),
    verified: false,
    source: "ECS Label Scan",
  };
}

// Barcode: job.result.data = { found: bool, food: { product_name, nutritions: { calories, protein, fat, carbs } } }
function normalizeBarcodeToGENFOOD(ecsResult: any): GENFOOD {
  const data = ecsResult?.data;
  if (!data?.found) throw new Error("Barcode not found or product not in database");

  const food = data.food;
  if (!food) throw new Error("No product data for barcode");

  const n = food.nutritions || {};
  return {
    food_id: `barcode_${food.barcode || "unknown"}`,
    name_vi: food.product_name || food.name || "Sản phẩm",
    name_en: food.product_name || food.name || "Product",
    macros: {
      calories: Number(n.calories) || 0,
      protein_g: Number(n.protein) || 0,
      carbs_g: Number(n.carbs) || 0,
      fat_g: Number(n.fat) || 0,
      fiber_g: Number(n.fiber) || 0,
      sugar_g: Number(n.sugar) || 0,
      sodium_mg: Number(n.sodium) ? Number(n.sodium) * 1000 : 0,
      saturated_fat_g: 0, polyunsaturated_fat_g: 0, monounsaturated_fat_g: 0,
      cholesterol_mg: 0, potassium_mg: 0,
    },
    micronutrients: EMPTY_MICRONUTRIENTS,
    serving: {
      default_g: 100,
      unit: "g",
      portions: { small: 0.7, medium: 1.0, large: 1.3 },
    },
    ingredients: [],
    verified: true,
    source: `Barcode (${data.source || "database"})`,
  };
}

function normalizeEcsResult(action: string, ecsResult: any): GENFOOD {
  switch (action) {
    case "analyzeFoodLabel": return normalizeLabelToGENFOOD(ecsResult);
    case "scanBarcode":      return normalizeBarcodeToGENFOOD(ecsResult);
    default:                 return normalizeFoodToGENFOOD(ecsResult);
  }
}

export const handler = async (event: any) => {
  try {
    const { action, payload } = event.arguments;

    debug("Handler invoked", { action, hasPayload: !!payload });

    // Validate action
    if (!ACTION_TO_ENDPOINT[action]) {
      return JSON.stringify({
        success: false,
        error: `Unknown action: ${action}`,
      });
    }

    // Parse payload
    let s3Key: string;
    try {
      const parsed = JSON.parse(payload || "{}");
      s3Key = parsed.s3Key;
    } catch (e) {
      return JSON.stringify({
        success: false,
        error: "Invalid payload JSON",
      });
    }

    // Validate s3Key
    if (!s3Key || typeof s3Key !== "string") {
      return JSON.stringify({
        success: false,
        error: "Missing or invalid s3Key in payload",
      });
    }

    if (s3Key.includes("..")) {
      return JSON.stringify({
        success: false,
        error: "Invalid s3Key: path traversal not allowed",
      });
    }

    debug("Downloading image from S3", { s3Key });

    // Download image from S3
    const s3Response = await s3Client.send(
      new GetObjectCommand({
        Bucket: STORAGE_BUCKET,
        Key: s3Key,
      })
    );

    // Stream body to buffer
    const chunks: Uint8Array[] = [];
    const stream = s3Response.Body;
    if (!stream) {
      throw new Error("Failed to read S3 object");
    }

    for await (const chunk of stream as any) {
      chunks.push(chunk instanceof Buffer ? chunk : Buffer.from(chunk));
    }

    const imageBuffer = Buffer.concat(chunks);
    const contentType = s3Response.ContentType || "image/jpeg";

    debug("Image downloaded", { size: imageBuffer.length, contentType });

    // Get API key + generate JWT for ECS auth
    const secrets = await getSecrets();
    if (!secrets.apiKey) {
      return JSON.stringify({
        success: false,
        error: "ECS API key not configured",
      });
    }
    const token = generateJWT(secrets.apiKey);
    const authHeader = `Bearer ${token}`;
    const internalToken = secrets.internalToken;

    // Build FormData with Blob
    const blob = new Blob([imageBuffer], { type: contentType });
    const form = new FormData();
    form.append("file", blob, "upload.jpg");

    // Call ECS — POST to get job_id (async response, HTTP 202)
    const endpoint = ACTION_TO_ENDPOINT[action];
    const ecsUrl = `${ECS_BASE_URL}${endpoint}`;

    debug("Calling ECS", { ecsUrl });

    const controller = new AbortController();
    // 290s total: ~3s init + up to 270s polling + buffer
    const timeout = setTimeout(() => controller.abort(), 290_000);

    let jobId: string;
    let stickyCookie: string | undefined;
    try {
      const initResponse = await fetch(ecsUrl, {
        method: "POST",
        body: form,
        headers: { Authorization: authHeader, "X-Nutri-Internal-Token": internalToken },
        signal: controller.signal,
      });

      if (!initResponse.ok) {
        const errorBody = await initResponse.text();
        let errorDetail = "Analysis failed";
        try {
          const errorJson = JSON.parse(errorBody);
          errorDetail = errorJson.detail || errorDetail;
        } catch {
          // Ignore parse errors
        }
        return JSON.stringify({
          success: false,
          error: `ECS error: ${errorDetail}`,
        });
      }

      const initBody = await initResponse.json();
      jobId = initBody.job_id;

      if (!jobId) {
        return JSON.stringify({
          success: false,
          error: "ECS did not return a job_id",
        });
      }

      // Read ALB sticky cookie to ensure polls go to same ECS task
      const setCookie = initResponse.headers.get("set-cookie");
      stickyCookie = setCookie ? setCookie.split(";")[0] : undefined;

      debug("Job accepted", { jobId, status: initBody.status });
    } finally {
      // Don't clear timeout yet — still need it for polling
    }

    // Poll for result
    let ecsResult: any;
    try {
      ecsResult = await pollJob(jobId, authHeader, internalToken, controller.signal, stickyCookie);
    } finally {
      clearTimeout(timeout);
    }

    debug("Job completed", { jobId });

    // Normalize response based on action type
    const genfood = normalizeEcsResult(action, ecsResult);

    return JSON.stringify({
      success: true,
      text: JSON.stringify(genfood),
    });
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    console.error("[scan-image] Error:", errorMessage, error);

    if (errorMessage.includes("NoSuchKey")) {
      return JSON.stringify({
        success: false,
        error: "Image not found in S3",
      });
    }

    if (
      errorMessage.includes("AbortError") ||
      errorMessage.includes("timeout")
    ) {
      return JSON.stringify({
        success: false,
        error: "Image processing service unavailable (timeout)",
      });
    }

    if (errorMessage.includes("ECONNREFUSED")) {
      return JSON.stringify({
        success: false,
        error: "Image processing service unavailable (connection refused)",
      });
    }

    return JSON.stringify({
      success: false,
      error: errorMessage,
    });
  }
};
```

---

[Back to functions list](../)
