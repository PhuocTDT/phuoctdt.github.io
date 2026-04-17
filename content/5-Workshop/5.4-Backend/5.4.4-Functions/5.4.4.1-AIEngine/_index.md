---
title: "Function ai-engine (Bedrock)"
menuTitle: "5.4.4.1. Function ai-engine"
weight: 1
---

This function acts as the "brain" of the application, responsible for interacting with **Amazon Bedrock** to analyze natural language and images.

### 1. Install Libraries (npm install)

This function requires the Bedrock Runtime SDK and the Transcribe SDK (if processing voice):

```bash
cd amplify/ai-engine
npm install @aws-sdk/client-bedrock-runtime @aws-sdk/client-transcribe
```

### 2. Resource Configuration (`resource.ts`)

```typescript
import { defineFunction } from '@aws-amplify/backend';

export const aiEngine = defineFunction({
  name: 'ai-engine',
  entry: './handler.ts',
  runtime: 22,
  memoryMB: 512,
  timeoutSeconds: 30,
});
```

### 3. Handler Logic (`handler.ts`)

```typescript
import { BedrockRuntimeClient, InvokeModelCommand } from "@aws-sdk/client-bedrock-runtime";
import { TranscribeClient, StartTranscriptionJobCommand, GetTranscriptionJobCommand, DeleteTranscriptionJobCommand } from "@aws-sdk/client-transcribe";
import { S3Client, DeleteObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { DynamoDBClient, ListTablesCommand } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, QueryCommand, ScanCommand } from "@aws-sdk/lib-dynamodb";
import { randomUUID } from "crypto";

const REGION = "ap-southeast-2";
const bedrockClient = new BedrockRuntimeClient({ region: REGION });
const transcribeClient = new TranscribeClient({ region: REGION });
const s3Client = new S3Client({ region: REGION });
const dbClient = new DynamoDBClient({ region: REGION });
const docClient = DynamoDBDocumentClient.from(dbClient);

const QWEN_MODEL_ID = process.env.QWEN_MODEL_ID || "qwen.qwen3-vl-235b-a22b";
const STORAGE_BUCKET = process.env.STORAGE_BUCKET_NAME || "";
const IS_DEBUG = process.env.DEBUG === "true" || process.env.NODE_ENV === "development";

// Simple debug logger - respects DEBUG env var
const debug = (message: string, data?: any) => {
  if (IS_DEBUG) {
    console.log(`[ai-engine] ${message}`, data || "");
  }
};

// ═══════════════════════════════════════════════════════════════
// PROMPTS (from docs/prompts)
// ═══════════════════════════════════════════════════════════════

const GEN_FOOD_SYSTEM_PROMPT = `You are Ollie, an expert AI nutrition assistant for the NutriTrack app.
A user has searched for a food, dish, or meal that is NOT in our local database, or provided an image. Your job is to analyze the food and estimate its ingredients, standard portion size, macros, and micronutrients.

RULES:
1. Break down the meal into its core raw ingredients. (e.g., "Boiled Potatoes and Pan seared chicken" -> Potatoes, Chicken Breast, Olive Oil, etc.).
2. Estimate a standard, medium portion size for the ENTIRE dish/meal.
3. Provide estimated macros and micronutrients reflecting that portion size.
4. CALORIES: Ensure (Protein*4 + Carbs*4 + Fat*9) roughly matches the total calories.
5. Provide the food name and ingredients in BOTH Vietnamese (name_vi) and English (name_en).
6. Tone: Vietnamese casual (ê, nhé, nha), encouraging, practical. Use emojis sparingly (💪🔥).
7. Output STRICT JSON format only. NO markdown blocks (\`\`\`json), no conversational text.

EDGE CASE:
- If the input is clearly NOT a food, beverage, or edible item: return exactly:
{"error": "not_food", "message_vi": "Vui lòng nhập một món ăn hoặc nguyên liệu hợp lệ.", "message_en": "Please enter a valid food or ingredient."}

OUTPUT SCHEMA:
{
  "food_id": "custom_gen_temp",
  "name_vi": "Tên tiếng Việt",
  "name_en": "English Name",
  "macros": { "calories": 0, "protein_g": 0, "carbs_g": 0, "fat_g": 0, "saturated_fat_g": 0, "polyunsaturated_fat_g": 0, "monounsaturated_fat_g": 0, "fiber_g": 0, "sugar_g": 0, "sodium_mg": 0, "cholesterol_mg": 0, "potassium_mg": 0 },
  "micronutrients": { "calcium_mg": 0, "iron_mg": 0, "vitamin_a_ug": 0, "vitamin_c_mg": 0 },
  "serving": { "default_g": 0, "unit": "bowl | plate | serving | piece", "portions": { "small": 0.7, "medium": 1.0, "large": 1.3 } },
  "ingredients": [ { "name_vi": "Tên nguyên liệu", "name_en": "Ingredient Name", "weight_g": 0 } ],
  "verified": false, "source": "AI Generated"
}`;

const FIX_FOOD_SYSTEM_PROMPT = `You are Ollie, an expert AI nutritionist for NutriTrack.
Your task is to correct a logged food item based on user instructions.

RULES:
1. ARITHMETIC: If ingredients or weights change, recalculate ALL macros/micronutrients.
2. CALORIES: Ensure (Protein*4 + Carbs*4 + Fat*9) roughly matches the new total.
3. PERSONALITY: Professional, polite, and helpful nutritionist (tư vấn viên dinh dưỡng lịch sự, chuyên nghiệp).
4. Output STRICT JSON format only. NO markdown blocks (\`\`\`json).

EDGE CASE:
- If request is nonsense/non-food: return {"error": "not_food", "message_vi": "Vui lòng nhập yêu cầu sửa món chính xác để Ollie có thể hỗ trợ bạn nhé!", "message_en": "Please enter a valid correction request!"}

OUTPUT SCHEMA: Same as GEN_FOOD (with "source": "AI Fixed").`;

const VOICE_SYSTEM_PROMPT = `You are Ollie, an expert AI nutrition assistant for NutriTrack.
You understand both Vietnamese (casual) and English.

YOUR TASK:
When the user describes a meal, a specific dish, or just raw ingredients via voice or text, analyze the components and log it. Even if the user is vague, you MUST estimate the nutritional profile.

RULES:
1. DETECT language (vi or en).
2. IDENTIFY items: Can be a complete dish (e.g., "Phở bò") or raw ingredients (e.g., "200g thịt bò").
3. ESTIMATION: You MUST estimate standard portions if not provided and provide nutritional breakdown (macros) for the entire input and each ingredient/item. Never return 0 for macros unless the item has no calories.
4. PORTION: small | medium | large. Default: "medium".
5. RESPONSE: If user speaks Vietnamese → respond/clarify in polite and professional Vietnamese (lịch sự, rõ ràng). Avoid casual particles like 'ê', 'nha', 'nè'.
6. Output STRICT JSON format only. NO markdown blocks (\`\`\`json).

ERROR HANDLING:
- Unintelligible or Non-food input: return action="clarify". NEVER log non-food items.
- Example: "Cho tớ cái máy bay" -> action="clarify", clarification_question_vi="Món này không thể tiêu thụ được, quý khách vui lòng chọn món ăn khác nhé!"

OUTPUT SCHEMA:
{
  "action": "log" | "clarify",
  "detected_language": "vi" | "en",
  "meal_type": "breakfast | lunch | dinner | snack",
  "confidence": 0.0 to 1.0,
  "clarification_question_vi": "Câu hỏi tiếng Việt hoặc null",
  "clarification_question_en": "English question or null",
  "food_data": {
      "food_id": "custom_gen_temp",
      "name_vi": "Tên món/nguyên liệu", "name_en": "Dish/Ingredient Name",
      "macros": { "calories": 0, "protein_g": 0, "carbs_g": 0, "fat_g": 0, "saturated_fat_g": 0, "fiber_g": 0, "sugar_g": 0, "sodium_mg": 0 },
      "serving": { "default_g": 0, "unit": "bowl | plate | piece | g | ml", "portions": {"small": 0.7, "medium": 1.0, "large": 1.3} },
      "ingredients": [ {"name_vi": "tên nguyên liệu", "name_en": "ingredient name", "weight_g": 0, "calories": 0, "protein_g": 0, "carbs_g": 0, "fat_g": 0} ]
  }
}`;

const OLLIE_COACH_SYSTEM_PROMPT = `You are Ollie, a Vietnamese AI nutrition coach in the NutriTrack app.

PERSONALITY:
- 👔 Professional, polite, and encouraging health consultant.
- 💪 Motivating and providing scientific-backed evidence.
- 🌐 LANGUAGE: Follow the user's language. If they speak English, reply in English. If they speak Vietnamese, reply in professional and respectful Vietnamese.
- 🎯 Actionable: give specific, practical advice.
- ✨ Celebrate milestones with a supportive tone.

RULES:
1. MAX 2 sentences per response. Short and punchy.
2. Use 1-2 emojis max. Don't overdo it.
3. Reference the user's ACTUAL data (streak, calories, protein).
4. Be specific: "ăn thêm 2 trứng luộc" not "ăn thêm protein".
5. Output STRICT JSON format only. NO markdown blocks (\`\`\`json), no conversational text.

EDGE CASE:
- If stats are missing or absurd, provide a generic encouraging message. skip specific numbers.

OUTPUT FORMAT — always return a single JSON object:
{
  "tip_vi": "Lời khuyên của Ollie (Vietnamese casual)",
  "tip_en": "Ollie's tip in English (energetic)",
  "mood": "celebrate | encourage | suggest | neutral",
  "suggested_food_vi": "Món gợi ý (nếu có)",
  "suggested_food_en": "Suggested food (if any)"
}`;

const RECIPE_SYSTEM_PROMPT = `You are Ollie, a Vietnamese cooking coach in the NutriTrack app.

YOUR TASK:
Suggest 1-3 recipes based on fridge inventory and goals.

RULES:
1. USE EXPIRING ITEMS FIRST — essential for food waste reduction.
2. NUTRITION GOAL: high_protein | low_carb | balanced | low_calorie.
3. REALISTIC: Home-cookable in ≤45 minutes.
4. TONE: Professional, encouraging, and respectful (lịch sự, hướng dẫn tận tình).
5. Output STRICT JSON format only. NO markdown blocks (\`\`\`json).

EDGE CASE:
- If inventory is non-food: return {"recipes": [], "overall_tip_vi": "Mình chỉ giúp tạo công thức nấu ăn thui nha! 🍳", "overall_tip_en": "I can only help with recipes! 🍳", "error": "not_food"}.

OUTPUT SCHEMA:
{
  "recipes": [
    {
      "dish_name_vi": "Tên món", "dish_name_en": "Dish Name",
      "why_this_vi": "Lý do chọn cực thuyết phục", "why_this_en": "Why this dish",
      "cooking_time_min": 30, "difficulty": "easy | medium | hard",
      "ingredients_from_fridge": [ {"name": "thịt", "weight_g": 200} ],
      "need_to_buy": ["nước mắm"],
      "macros": {"calories": 420, "protein_g": 35, "carbs_g": 30, "fat_g": 18},
      "steps_vi": ["Bước 1: ..."], "steps_en": ["Step 1: ..."],
      "tip_vi": "Mẹo nấu", "tip_en": "Cooking tip"
    }
  ],
  "overall_tip_vi": "Lời khuyên tổng quát", "overall_tip_en": "Overall tip"
}`;

const MACRO_CALCULATOR_SYSTEM_PROMPT = `You are Ollie, an expert AI nutritionist for NutriTrack.
Calculate daily targets based on biometrics, goals, and lifestyle.

RULES:
1. CALCULATION: Use Mifflin-St Jeor for TDEE.
2. GOALS: Deficit (-500) for weight loss, Surplus (+300) for gain.
3. MACROS: Ensure (Protein*4 + Carbs*4 + Fat*9) equals daily_calories.
4. TONE: Professional and encouraging health consultant. Avoid casual Gen-Z slang.
5. Output STRICT JSON format only. NO markdown blocks (\`\`\`json).

EDGE CASE:
- If biometrics are absurd: return 2000 cal default and ask to update profile "để thông tin chính xác hơn".

OUTPUT SCHEMA:
{
  "daily_calories": 2000,
  "daily_protein_g": 150, "daily_carbs_g": 150, "daily_fat_g": 65,
  "reasoning_vi": "Lý do tính toán (lịch sự, rõ ràng)",
  "reasoning_en": "Calculation reasoning (professional)"
}`;

const WEEKLY_INSIGHT_SYSTEM_PROMPT = `You are Ollie, an expert AI nutritionist and Gen-Z coach for NutriTrack.
Analyze user food logs and biometrics to provide a "Weekly Insight".

RULES:
1. PROGRESS: Acknowledge wins, identify one key pattern.
2. ADVICE: One clear, easyToAction tip for next week.
3. TONE: Professional, supportive, and informative. Avoid slang like 'á', 'nhen', 'xịn'.
4. LENGTH: Exactly 3 sentences.
5. Output STRICT JSON format only. NO markdown blocks (\`\`\`json).

OUTPUT SCHEMA:
{
  "insight_vi": "Nhận xét tuần này bằng tiếng Việt (chuyên nghiệp)",
  "insight_en": "Insight in English (professional)",
  "status": "success | insufficient_data"
}`;

const AI_COACH_SYSTEM_PROMPT = `You are Ollie, a professional Vietnamese AI health and nutrition consultant for NutriTrack.
You are a knowledgeable advisor who is polite, respectful, and evidence-based. Avoid casual slang or acting like a Gen-Z peer.

SCOPE:
- Nutrition, food, healthy eating, exercise, health stats, wellness.
- Politely decline off-topic questions (e.g. "Món này không thể tiêu thụ được, quý khách vui lòng chọn món ăn khác nhé!").

TONE:
- Respectful and warm Vietnamese: dùng "bạn" / "mình", tránh tiếng lóng Gen-Z.
- Professional but approachable — như một chuyên gia dinh dưỡng tận tâm, không phải bạn bè ngang hàng.
- Ngắn gọn, tập trung vào điều người dùng thực sự hỏi.
- Khuyến khích nhẹ nhàng, không phán xét.

RULES:
1. LANGUAGE: You MUST match the user's language prefix from the prompt below. This rule is absolute.
2. MEAL SUGGESTION: Suggest 1-3 meals. Prioritize expiring items from fridge.
3. CONTEXT RULES: Use USER CONTEXT data ONLY when the user asks about nutrition/meals/health/progress. For casual conversation (greetings, small talk), respond naturally without mentioning stats or numbers.
4. CARDS: Use specific delimiters (===FOOD_CARD_START=== etc.) placed at the end.
5. Reply in natural conversational text. NEVER output raw JSON objects in the message body.
6. NEVER use markdown code fences (\`\`\`json, \`\`\`, etc.). Write plain text only.
7. NEVER expose internal data structures, field names, or API responses to the user.

CARD TEMPLATES (Place at the END of response):

===FOOD_CARD_START===
{"name": "Tên món", "description": "Lý do chọn", "calories": 450, "protein_g": 30, "carbs_g": 40, "fat_g": 10, "time": "25 phút", "emoji": "🍱", "ingredients": [{"name": "Gạo", "amount": "1 chén"}], "steps": [{"title": "Nấu cơm", "instruction": "Vo gạo nấu"}]}
===FOOD_CARD_END===

===EXERCISE_CARD_START===
{"name": "Tên bài tập", "description": "Ưu điểm", "duration_minutes": 30, "calories_burned": 250, "emoji": "🏃"}
===EXERCISE_CARD_END===

===STATS_CARD_START===
{"calories_consumed": 1800, "calories_target": 2000, "protein_g": 85, "carbs_g": 210, "fat_g": 60, "summary": "Ngon lành! Ráng ăn thêm đạm nhé."}
===STATS_CARD_END===

Append this at the very end of your message: "Ghi chú: Thông tin công thức/dinh dưỡng chỉ mang tính tham khảo, bạn có thể tùy chỉnh để phù hợp với khẩu vị cá nhân."`;

// ═══════════════════════════════════════════════════════════════
// HELPERS
// ═══════════════════════════════════════════════════════════════

async function waitForTranscription(jobName: string): Promise<string> {
    for (let i = 0; i < 25; i++) {
        await new Promise(r => setTimeout(r, 2000));
        const result = await transcribeClient.send(new GetTranscriptionJobCommand({ TranscriptionJobName: jobName }));
        const status = result.TranscriptionJob?.TranscriptionJobStatus;

        if (status === 'COMPLETED') {
            const transcriptUri = result.TranscriptionJob?.Transcript?.TranscriptFileUri;
            if (!transcriptUri) throw new Error('No transcript URI');
            const res = await fetch(transcriptUri);
            const json = await res.json();
            return json.results?.transcripts?.[0]?.transcript || '';
        }
        if (status === 'FAILED') {
            throw new Error(`Transcription failed: ${result.TranscriptionJob?.FailureReason}`);
        }
    }
    throw new Error('Transcription timed out');
}

async function callQwen(messages: any[], maxTokens = 1000): Promise<string> {
    const body = JSON.stringify({ messages, max_tokens: maxTokens });
    const command = new InvokeModelCommand({
        modelId: QWEN_MODEL_ID,
        contentType: "application/json",
        accept: "application/json",
        body,
    });
    const response = await bedrockClient.send(command);
    const responseBody = JSON.parse(new TextDecoder().decode(response.body));

    const text: string = responseBody.choices?.[0]?.message?.content
        || responseBody.output?.message?.content?.[0]?.text
        || responseBody.content?.[0]?.text
        || '';
    if (!text) {
        debug('Empty Qwen response. Raw body:', JSON.stringify(responseBody).slice(0, 500));
    }
    return text;
}

// ═══════════════════════════════════════════════════════════════
// HANDLER
// ═══════════════════════════════════════════════════════════════

export const handler = async (event: any) => {
    const { action, payload } = event.arguments || {};

    try {
        const data = payload ? JSON.parse(payload) : {};

        // ── Image Analysis (Qwen3-VL vision) ──
        if (action === 'analyzeFoodImage') {
            const { s3Key } = data;
            if (!STORAGE_BUCKET) throw new Error('STORAGE_BUCKET_NAME not configured');
            if (!s3Key || s3Key.includes('..')) throw new Error('Invalid s3Key');

            // Read image from S3, convert to base64 (avoids large payload through AppSync)
            const s3Obj = await s3Client.send(new GetObjectCommand({ Bucket: STORAGE_BUCKET, Key: s3Key }));
            const chunks: Uint8Array[] = [];
            for await (const chunk of s3Obj.Body as any) chunks.push(chunk);
            const imageBuffer = Buffer.concat(chunks);
            const base64 = imageBuffer.toString('base64');
            const contentType = s3Obj.ContentType || 'image/jpeg';

            const text = await callQwen([
                { role: "system", content: GEN_FOOD_SYSTEM_PROMPT },
                {
                    role: "user",
                    content: [
                        { type: "image_url", image_url: { url: `data:${contentType};base64,${base64}` } },
                        { type: "text", text: "Analyze this food image and estimate its nutritional profile. Use Vietnamese (tiếng Việt) for name_vi and all ingredient name_vi fields. Return ONLY the JSON object." },
                    ],
                },
            ]);

            // File stays in incoming/ for food-detail display.
            // S3 lifecycle rule (expirationInDays: 1 on incoming/) handles cleanup automatically.
            return JSON.stringify({ success: true, text });
        }

        // ── Conversational AI Coach ──
        if (action === 'generateCoachResponse') {
            const { userMessage, chatHistory, contextString } = data;

            // Deterministic language detection: check for Vietnamese diacritical marks.
            // This is far more reliable than asking the model to detect language from a
            // mixed context that contains Vietnamese food names.
            const VI_PATTERN = /[àáâãèéêìíòóôõùúýăđơưạảấầẩẫậắằẳẵặẹẻẽếềểễệỉịọỏốồổỗộớờởỡợụủứừửữựỳỷỹỵ]/i;
            const isVietnamese = VI_PATTERN.test(userMessage);
            const detectedLang = isVietnamese ? 'vi' : 'en';
            const langInstruction = detectedLang === 'vi'
                ? 'MANDATORY: Reply ENTIRELY in Vietnamese (professional and polite: nhé, vui lòng). Do NOT use casual slang or Gen-Z particles like "ê", "nha", "nè", "á".'
                : 'MANDATORY: Reply ENTIRELY in English. Do NOT use any Vietnamese words, even in food names.';

            const messages: any[] = [
                { role: "system", content: AI_COACH_SYSTEM_PROMPT },
            ];
            for (const msg of chatHistory) {
                messages.push({
                    role: msg.role === 'user' ? 'user' : 'assistant',
                    content: msg.parts[0].text
                });
            }
            // Context as system message so food names don't skew language detection.
            if (contextString) {
                messages.push({
                    role: "system",
                    content: `USER CONTEXT (background data only):\n${contextString}`
                });
            }
            // Explicit language instruction as final system message — overrides everything.
            messages.push({
                role: "system",
                content: langInstruction
            });
            messages.push({
                role: "user",
                content: userMessage
            });

            const text = await callQwen(messages, 2000);
            return JSON.stringify({ success: true, text });
        }

        // ── Food Search / Generation (DB miss → AI) ──
        if (action === 'generateFood') {
            const { foodName } = data;
            const text = await callQwen([
                { role: "system", content: GEN_FOOD_SYSTEM_PROMPT },
                { role: "user", content: `Analyze the following unknown food and estimate its nutritional profile:\nFood Query: "${foodName}"\n\nReturn ONLY the JSON object.` },
            ]);
            return JSON.stringify({ success: true, text });
        }

        // ── Fix/Correct a Food Item ──
        if (action === 'fixFood') {
            const { currentFoodJson, correctionQuery } = data;
            const text = await callQwen([
                { role: "system", content: FIX_FOOD_SYSTEM_PROMPT },
                { role: "user", content: `Please fix the following food item based on the user's request:\n\nCurrent Food Data:\n${JSON.stringify(currentFoodJson, null, 2)}\n\nCorrection Request: "${correctionQuery}"\n\nReturn ONLY the new JSON object.` },
            ]);
            return JSON.stringify({ success: true, text });
        }

        // ── Voice → Food (Transcribe + Qwen) ──
        if (action === 'voiceToFood') {
            const { s3Key } = data;
            if (!STORAGE_BUCKET) throw new Error('STORAGE_BUCKET_NAME not configured');
            if (!s3Key || !s3Key.startsWith('voice/') || s3Key.includes('..')) {
                throw new Error('Invalid s3Key');
            }

            const jobName = `nutritrack-voice-${randomUUID()}`;

            // Map file extension → AWS Transcribe MediaFormat enum
            const ext = s3Key.split('.').pop()?.toLowerCase() || 'm4a';
            const mediaFormat = ext === 'webm' ? 'webm'
                : ext === 'mp3'  ? 'mp3'
                : ext === 'wav'  ? 'wav'
                : ext === 'flac' ? 'flac'
                : 'mp4'; // m4a, mp4 → 'mp4'

            const s3Uri = `s3://${STORAGE_BUCKET}/${s3Key}`;
            await transcribeClient.send(new StartTranscriptionJobCommand({
                TranscriptionJobName: jobName,
                // Use specific language — IdentifyLanguage produces empty transcripts with WebM
                LanguageCode: 'vi-VN',
                MediaFormat: mediaFormat as any,
                Media: { MediaFileUri: s3Uri },
            }));

            const transcript = await waitForTranscription(jobName);

            // Cleanup: xóa Transcribe job và voice file khỏi S3 (ephemeral)
            await Promise.allSettled([
                transcribeClient.send(new DeleteTranscriptionJobCommand({ TranscriptionJobName: jobName })),
                s3Client.send(new DeleteObjectCommand({ Bucket: STORAGE_BUCKET, Key: s3Key })),
            ]);

            debug(`[voiceToFood] jobName=${jobName}, transcriptLength=${transcript?.length || 0}, s3Key=${s3Key}`);

            if (!transcript) {
                return JSON.stringify({ success: false, error: 'Empty transcription' });
            }

            const qwenResult = await callQwen([
                { role: "system", content: VOICE_SYSTEM_PROMPT },
                { role: "user", content: `User said: "${transcript}"\n\nAnalyze and return JSON following the output format.` },
            ]);

            // ==========================================
            // RECALCULATE with DB Data (Phase 1)
            // ==========================================
            try {
                const aiResponse = JSON.parse(qwenResult);
                if (aiResponse.action === 'log' && aiResponse.food_data) {
                    const foodData = aiResponse.food_data;
                    const ingredients = foodData.ingredients || [];
                    const tableName = await discoverTableName();
                    
                    let totalCalories = 0, totalProtein = 0, totalCarbs = 0, totalFat = 0;
                    const processedIngredients = [];

                    for (const ing of ingredients) {
                        const searchName = ing.name_vi || ing.name_en || ing.name;
                        const dbMatch = await findFoodInDB(searchName, tableName);
                        if (dbMatch) {
                            const nutrition = calculateNutrition(dbMatch, ing.weight_g || 100);
                            processedIngredients.push({
                                ...ing,
                                food_id: dbMatch.food_id,
                                name_vi_db: dbMatch.name_vi,
                                name_en_db: dbMatch.name_en,
                                matched: true,
                                source: 'database',
                                ...nutrition
                            });
                            totalCalories += (nutrition.calories || 0);
                            totalProtein += (nutrition.protein_g || 0);
                            totalCarbs += (nutrition.carbs_g || 0);
                            totalFat += (nutrition.fat_g || 0);
                        } else {
                            processedIngredients.push({
                                ...ing,
                                matched: false,
                                source: 'ai_estimated'
                            });
                            // Keep AI estimated values
                            totalCalories += (ing.calories || 0);
                            totalProtein += (ing.protein_g || 0);
                            totalCarbs += (ing.carbs_g || 0);
                            totalFat += (ing.fat_g || 0);
                        }
                    }

                    // Only overwrite macros if recalculated total is meaningful (>0),
                    // otherwise keep Qwen's original food_data.macros estimates
                    const hasRecalculatedData = totalCalories > 0 || totalProtein > 0 || totalCarbs > 0 || totalFat > 0;
                    if (hasRecalculatedData) {
                        aiResponse.food_data.macros = {
                            ...aiResponse.food_data.macros,
                            calories: Math.round(totalCalories * 10) / 10,
                            protein_g: Math.round(totalProtein * 10) / 10,
                            carbs_g: Math.round(totalCarbs * 10) / 10,
                            fat_g: Math.round(totalFat * 10) / 10
                        };
                    }
                    aiResponse.food_data.ingredients = processedIngredients;
                    aiResponse.db_verified = processedIngredients.some(i => i.matched);
                    
                    return JSON.stringify({ 
                        success: true, 
                        transcript, 
                        text: JSON.stringify(aiResponse) 
                    });
                }
            } catch (calcError) {
                console.error('Recalculation error:', calcError);
            }

            return JSON.stringify({ success: true, transcript, text: qwenResult });
        }

        // ── Helper functions for DB Lookup ──
        async function discoverTableName(): Promise<string> {
            const result = await dbClient.send(new ListTablesCommand({}));
            return result.TableNames?.find((name: string) => name.startsWith('Food-')) || "";
        }

        function normalize(text: string): string {
            if (!text) return "";
            return text.toLowerCase().trim().normalize('NFD').replace(/[\u0300-\u036f]/g, '').replace(/đ/g, 'd');
        }

        async function findFoodInDB(query: string, tableName: string): Promise<any | null> {
            if (!tableName || !query) return null;
            const q = query.toLowerCase().trim();

            // 1. Exact match (name_vi or name_en) - GSI
            const qVi = await docClient.send(new QueryCommand({
                TableName: tableName, IndexName: 'name_vi',
                KeyConditionExpression: 'name_vi = :name',
                ExpressionAttributeValues: { ':name': query }
            }));
            if (qVi.Items?.length) return qVi.Items[0];

            const qEn = await docClient.send(new QueryCommand({
                TableName: tableName, IndexName: 'name_en',
                KeyConditionExpression: 'name_en = :name',
                ExpressionAttributeValues: { ':name': query }
            }));
            if (qEn.Items?.length) return qEn.Items[0];

            // 2. Partial match (contains) - Broad search
            const scanAny = await docClient.send(new ScanCommand({
                TableName: tableName,
                FilterExpression: 'contains(name_vi, :q) OR contains(name_en, :q) OR contains(aliases_vi, :q) OR contains(aliases_en, :q)',
                ExpressionAttributeValues: { ':q': query }
            }));
            if (scanAny.Items?.length) {
                // Return shortest name match first (most likely to be the core item)
                return scanAny.Items.sort((a, b) => a.name_vi.length - b.name_vi.length)[0];
            }

            return null;
        }

        function calculateNutrition(dbFood: any, weightG: number) {
            const m = dbFood.macros || {};
            const r = weightG / 100;
            return {
                calories: Math.round((m.calories || 0) * r * 10) / 10,
                protein_g: Math.round((m.protein_g || 0) * r * 10) / 10,
                carbs_g: Math.round((m.carbs_g || 0) * r * 10) / 10,
                fat_g: Math.round((m.fat_g || 0) * r * 10) / 10
            };
        }


        // ── Ollie Coach Tip (quick nudge) ──
        if (action === 'ollieCoachTip') {
            const { promptTemplate, context } = data;
            const text = await callQwen([
                { role: "system", content: OLLIE_COACH_SYSTEM_PROMPT },
                { role: "user", content: promptTemplate || context },
            ]);
            return JSON.stringify({ success: true, text });
        }

        // ── Recipe Generator ──
        if (action === 'generateRecipe') {
            const { inventoryText, expiringText, nutritionGoal, servings } = data;
            const text = await callQwen([
                { role: "system", content: RECIPE_SYSTEM_PROMPT },
                { role: "user", content: `User's fridge inventory:\n${inventoryText}\n\nExpiring soon (MUST USE):\n${expiringText}\n\nNutrition goal: ${nutritionGoal}\nNumber of servings: ${servings}\n\nSuggest 1-3 recipes. Return JSON only.` },
            ]);
            return JSON.stringify({ success: true, text });
        }

        // ── Macro Calculator ──
        if (action === 'calculateMacros') {
            const { userProfileJson } = data;
            const text = await callQwen([
                { role: "system", content: MACRO_CALCULATOR_SYSTEM_PROMPT },
                { role: "user", content: `Calculate daily nutritional targets for the following user profile:\n\n${JSON.stringify(userProfileJson, null, 2)}\n\nReturn ONLY the JSON object.` },
            ]);
            return JSON.stringify({ success: true, text });
        }

// ── Weekly Insight ──
        if (action === 'weeklyInsight') {
            const { userProfileJson, weeklySummaryJson, notablePatterns } = data;
            const text = await callQwen([
                { role: "system", content: WEEKLY_INSIGHT_SYSTEM_PROMPT },
                { role: "user", content: `Analyze this user's weekly data and generate the Weekly Insight:\n\nUser Profile:\n${JSON.stringify(userProfileJson, null, 2)}\n\nWeekly Summary:\n${JSON.stringify(weeklySummaryJson, null, 2)}\n\nNotable Patterns:\n${notablePatterns}\n\nReturn ONLY the JSON object.` },
            ]);
            return JSON.stringify({ success: true, text });
        }

        return JSON.stringify({ success: false, error: `Unknown action: ${action}` });

    } catch (error: any) {
        debug('Bedrock Lambda Error:', error.message);
        return JSON.stringify({ success: false, error: error.message });
    }
};
```

---

[Back to functions list](../)
