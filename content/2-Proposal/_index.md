---
title: "Proposal"
date: 2026-01-09T15:00:11+07:00
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# NutriTrack Platform

## AI-Integrated AWS Serverless Solution for Nutrition Tracking

---

### 1. Project Summary

NutriTrack is built for Gen Z and Millennial users in Vietnam — those who want meaningful personal health control without the complexity. The platform aims for 1,000+ active users, featuring a cross-platform mobile interface developed with React Native (Expo), allowing users to log meal data via photos, voice, or manual entry.

The entire infrastructure complies with the AWS Well-Architected Framework, leveraging the serverless ecosystem (AWS Amplify Gen 2, AppSync, Lambda) and Generative AI capabilities from Amazon Bedrock (Qwen3-VL 235B model). The result is a real-time nutrition tracking system with predictive AI, rich gamification mechanisms, and optimized operational costs — all protected by Amazon Cognito combined with Google OAuth2.

---

### 2. Context and Solution

#### Current Problems

Most current nutrition tracking apps require manual data entry — tedious and easy to give up on. Food databases are often poor in Vietnamese cuisine, lacking familiar dishes like pho, banh mi, or bun bo Hue. Furthermore, third-party nutrition APIs (Nutritionix, FatSecret) are expensive to scale and limited in their data models.

Managing the fridge, planning daily menus, or utilizing available ingredients are also problems that haven't been well-solved — leading to food waste and inconsistent eating habits.

#### How NutriTrack Solves It?

NutriTrack approaches this with a fully serverless AWS-native architecture:

- **AWS AppSync (GraphQL)** handles meal data via mutations and real-time subscriptions.
- **AWS Lambda** with 4 specialized functions handles all backend logic — from AI orchestration to friend management and image optimization.
- **Amazon DynamoDB** stores 8 data models with auto-scaling capabilities.
- **Amazon Bedrock (Qwen3-VL 235B)** analyzes food photos, scans barcodes/nutrition labels, and generates nutrition data for items not in the database.
- **AWS Amplify Gen 2** combined with React Native/Expo creates a smooth Vietnamese–English bilingual interface.

**Key Features:**

| Feature | Description |
|-----------|-------|
| 📸 Food Image Analysis | Take one photo — get a detailed nutrition table immediately |
| 🎙️ Voice Logging | AWS Transcribe converts speech into meal data |
| 🍳 Smart Fridge | Track food storage, suggest recipes from available ingredients |
| 🤖 AI Coach "Ollie" | Personalized nutrition advice and weekly analysis |
| 🎮 Gamification | Daily streaks, evolving virtual pets, community challenges |
| 👥 Social Features | Friend system, leaderboards, competitive challenges |

#### Cost Efficiency

Estimated operational cost is **~$60.87/month** for 1,000 active users — by leveraging AWS Free Tier and the pay-as-you-go serverless model. The break-even point depends on the conversion rate to premium plans (unlimited image analysis, deep AI reports, priority coach) and will be confirmed after launch.

---

### 3. Solution Architecture

Data moves from the React Native app → CloudFront/WAF (Edge security) → AppSync GraphQL API → Lambda handlers → DynamoDB, enriched by Bedrock AI. The entire infrastructure is orchestrated via AWS Amplify Gen 2 with automated CI/CD across 3 deployment environments.

#### AWS Services Used

| Service | Role |
|---------|---------|
| **AWS Amplify Gen 2** | Orchestrates infrastructure using TypeScript CDK, manages CI/CD pipeline and 3 environments (Sandbox, feat/phase3, main) |
| **AWS AppSync** | Secure GraphQL API with real-time subscriptions between mobile app and Lambda resolvers |
| **AWS Lambda** | 4 logical handlers: `ai-engine`, `process-nutrition`, `friend-request`, `resize-image` |
| **Amazon DynamoDB** | 8 NoSQL tables: `user`, `Food` (~200 Vietnamese dishes), `FoodLog`, `FridgeItem`, `Challenge`, `ChallengeParticipant`, `Friendship`, `UserPublicStats` |
| **Amazon Bedrock** | Qwen3-VL 235B (ap-southeast-2) for image analysis, barcode scanning, label reading, recipe suggestions, AI coaching |
| **AWS Transcribe** | Converts audio recordings (S3 prefix `/voice`) to text for the voice logging feature |
| **Amazon S3** | Media storage with 4 prefixes: `incoming/` (deleted after 1 day), `voice/`, `avatar/`, `media/` |
| **Amazon Cognito** | Email/OTP and Google OAuth2 authentication, managing JWT tokens and user pools |
| **Amazon CloudFront** | CDN for global content delivery acceleration |
| **AWS WAF** | Firewall protecting API from common attacks and DDoS |
| **Route 53** | DNS resolution for the application domain |
| **Amazon ECR** | Container image repository for ECS Fargate |
| **Amazon ECS Fargate** | Serverless container for FastAPI backend in VPC (private subnets, 2 AZ, ALB, Auto Scaling) |
| **AWS CloudWatch** | Monitoring with 4 custom metrics: `Bedrock_AI_Error_Rate`, `Image_Processing_Time`, `User_Daily_Active`, `Food_Log_Count` |
| **AWS CloudTrail** | API logging for auditing and compliance |
| **AWS KMS** | Encryption of sensitive data at rest |
| **AWS Secrets Manager** | Secures API keys, JWT master key, and app configuration |
| **AWS Textract** | OCR for extracting text from nutrition labels |

#### Layered Design

**Client** — React Native/Expo app collects input via Camera, Microphone, and manual forms. Data is visualized through interactive charts and a gamification interface with virtual pets that evolve based on daily streaks.

**Edge Security** — Route 53 → WAF → CloudFront handles DNS resolution, DDoS mitigation, and content acceleration before requests reach the backend.

**Authentication** — Cognito manages registration (email/OTP), Google OAuth2 login, and JWT lifecycle. The app adds a local biometric protection layer (FaceID/TouchID).

**Processing** — 4 Lambda handlers handle all logic:

- `aiEngine` — Orchestrates 10 Bedrock tasks: `analyzeFoodImage`, `voiceToFood`, `generateRecipe`, `generateCoachResponse`, `searchFoodNutrition`, `fixFood`, `ollieCoachTip`, `calculateMacros`, `challengeSummary`, `weeklyInsight`
- `processNutrition` — Hybrid lookup: fuzzy match on DynamoDB (~200 Vietnamese dishes) first, Bedrock AI fallback if not found
- `friendRequest` — Social logic (send/accept/reject/block) via DynamoDB `TransactWriteItems`
- `resizeImage` — S3 event trigger on `incoming/`, using `sharp` to scale the longest side to 1280px (maintaining aspect ratio, auto-rotating via EXIF), encoding to progressive JPEG at 85 quality, written to `media/{entity_id}/`

**Data** — 8 DynamoDB tables with owner-scoped permissions and GSIs optimized for common query patterns.

**AI/ML** — Bedrock (Qwen3-VL 235B) and Transcribe form the core intelligence layer, all routed through `aiEngine` with structured prompt templates.

**Container** — ECS Fargate runs the FastAPI backend in a VPC (2 AZs in ap-southeast-2a), load balanced via ALB, connecting to AWS services through VPC Endpoints.

---

### 4. Technical Implementation

#### Technology Stack

- **Frontend:** React Native, Expo, TypeScript, Zustand, react-native-reanimated, expo-router, i18n (Vietnamese/English)
- **Backend:** Amplify Gen 2 (TypeScript CDK), Lambda (Node.js 22), AppSync, DynamoDB (8 tables + GSIs), Cognito (User Pools + Identity Pools + Google federation), S3 (4 prefixes + lifecycle rules)
- **AI:** Bedrock API (Qwen3-VL 235B, ap-southeast-2), Transcribe (async jobs), prompt engineering with JSON schema enforcement
- **Infra:** Docker/ECS Fargate, VPC (public/private subnets), ALB, Auto Scaling, ECR

#### Development Roadmap

**Month 0 — Research & Architecture**
Exploration of React Native/Expo with Amplify Gen 2, Designing serverless-AI architecture, drafting solution diagrams, and defining data models.

**Month 1 — Infrastructure Foundation**
Cost estimation using AWS Pricing Calculator (confirmed under $65/month for 1,000 users). Initializing Amplify Gen 2 sandbox, Cognito + Google federation, and 8-model GraphQL schema.

**Month 2 — Core Logic**
Implementing 4 TypeScript Lambda handlers, connecting AppSync resolvers, building React Native UI for 6 main tabs, integrating S3 image upload and resize pipeline.

**Month 3 — Integration & Launch**
Integrating Bedrock (Qwen3-VL) for 10 AI tasks, E2E testing across 3 environments, handling edge cases (JWT federation, `discoverTables()`, `NoValidAuthTokens`), and production deployment.

**Post-Launch — Continuous Optimization**
UX improvements based on user feedback, gamification upgrades, prompt engineering refinement, and premium package development.

---

### 5. Schedule & Milestones

| Phase | Duration | Key Content |
|-----------|------------|----------------|
| **Month 0 — Pre-production** | 1 month | UI/UX planning on Figma, migration from Flutter to React Native, Architecture design on draw.io, AWS service evaluation |
| **Month 1 — Infrastructure** | 1 month | Amplify Gen 2 initialization, Cognito + Google OAuth config, defining 8 DynamoDB models, building core UI (Home, Add Food, Kitchen) |
| **Month 2 — Development** | 1 month | AppSync GraphQL connection, implementing `processNutrition` and `friendRequest`, building Fridge interface, S3 + `resizeImage` integration |
| **Month 3 — Launch** | 1 month | `aiEngine` implementation (10 Bedrock tasks), gamification, 3-environment E2E testing, JWT + `discoverTables()` bug fixes, production release |
| **Post-Launch** | Continuous | Performance optimization, user feedback, iOS EAS Build pipeline, first-year scaling |

---

### 6. Budget Estimate

Calculated for **1,000 active users × 3 sessions/day × 30 days** = 90,000 AI interactions/month.

#### Monthly Infrastructure Cost

| Component | Cost | Notes |
|------------|---------|---------|
| Amazon S3 | $2.03 | 3GB storage + 5GB transfer out + PUT/GET requests |
| AWS Lambda | $0.26 | 270K requests, avg 0.2s, 512MB ARM64 |
| AppSync GraphQL | $0.34 | 270K operations @ $4/million |
| Amazon DynamoDB | $0.47 | 2GB storage + 810K reads + 180K writes |
| Amazon Cognito | $5.50 | 1,000 MAU @ $0.0055/MAU (Lite plan) |
| CloudWatch | $1.00 | 5 custom metrics + API logging |
| CloudTrail | $0.05 | 50,000 management events |
| Secrets Manager | $1.20 | 3 secrets (API keys, JWT key, DB config) |
| AWS KMS | $1.00 | 1 customer-managed key |
| AWS Textract | $3.60 | 2,400 label scans @ $0.0015/page |
| **Amazon Bedrock (AI)** | **$45.42** | 90K calls: input tokens ($16.70) + output tokens ($28.73) |
| **Total** | **$60.87/month** | **$730.44/year** |

#### Comparison of Bedrock (Qwen3-VL 235B) prices by region

| Region | Input | Output | Difference |
|--------|-------|--------|------------|
| US East (Virginia) | $0.00053/1K tokens | $0.00266/1K tokens | Baseline |
| Asia Pacific (Tokyo) | $0.00064/1K tokens | $0.00322/1K tokens | +21% |
| Asia Pacific (Mumbai) | $0.00062/1K tokens | $0.00313/1K tokens | +18% |
| Asia Pacific (Sydney) | *(Used for production)* | *(Used for production)* | ap-southeast-2 |

**Software cost:** $0 — all development tools are open source. Apple Developer account ($99/year) required if distributed on iOS.

---

### 7. Risk Assessment

| Risk | Impact | Probability | Mitigation Strategy |
|--------|----------|----------|------------|
| **Bedrock costs exceeding budget** | Medium | Medium | Cache AI results in DynamoDB, rate limit requests, set AWS Budget alert at $80/month |
| **Cognito/JWT authentication errors** | High | Low | Full E2E testing for federation flows, backup local AsyncStorage session, handle `UserNotConfirmedException` and `NotAuthorizedException` |
| **AI hallucination** | Medium | Medium | Prompt templates with mandatory JSON schema, allow users to verify and edit, maintain ~200 verified Vietnamese dishes in DynamoDB |
| **iOS build hurdles** | Low | High | Prioritize Android for MVP, use EAS Build cloud for iOS, plan macOS CI runners in later stages |
| **Inaccurate DynamoDB table discovery** | Medium | Low | **Root Cause:** `friendRequest` Lambda used `discoverTables()` via `ListTables`, returning wrong table names when multiple environments exist. **Fix:** Inject precise table names via CDK escape hatch — `cfnFriendRequestFn.addPropertyOverride('Environment.Variables.USER_TABLE_NAME', backend.data.resources.tables['user'].tableName)`, CDK auto-resolves correct ARN suffix on deploy |
| **Bedrock capacity limits** | Low | Low | Currently using ap-southeast-2; backup us-east-1 if needed |

#### Contingency Plan

- When AI/Bedrock fails: fallback to manual entry or fuzzy match on ~200 pre-loaded Vietnamese dishes.
- When backend deployment fails: rollback via CloudFormation/Amplify across 3 environments.
- When Qwen3-VL costs escalate: switch to Claude Haiku or Llama on Bedrock.

---

### 8. Expected Outcomes

#### Technical Improvements

- Logging time reduced from ~3 minutes (manual) to ~10 seconds (AI via photo/voice).
- Serverless infrastructure ready to handle 10,000+ concurrent users with nearly zero idle costs ($0.47/month basic DynamoDB).
- Hybrid AI strategy (DynamoDB fuzzy match + Bedrock AI fallback) balances cost-efficiency and data accuracy.

#### Long-term Value

- Building a database of verified Vietnamese dishes, expanding naturally through user and AI contributions — currently ~200 items and growing.
- Reusable AI architectural patterns (prompt templates, Lambda orchestration, Bedrock integration) applicable to future health features.
- Social platform with gamification (evolving pets, day streaks, challenges) drives daily usage and long-term user retention.

---

### 9. Next Steps

**iOS Pipeline** — Transition from Android-first MVP to iOS via EAS Build cloud runners; activate macOS CI runner when user volume justifies the cost.

**Vietnamese food database expansion** — From the initial ~200 seed items, collect AI-generated entries (`source: "AI Generated"` → `verified: false`) for team review; goal of 1,000+ verified dishes after the first year.

**Premium Package** — Validate pricing model (conversion rate high enough to cover $60.87/month baseline), release unlimited image analysis, weekly AI coach reports, and advanced macro charts.

**Observability** — Connect 4 CloudWatch custom metrics to a centralized dashboard + set Budget alarm at $80/month threshold.

**Fallback Bedrock cross-region** — If Qwen3-VL capacity at `ap-southeast-2` is limited, add backup route to Claude Haiku `us-east-1` to ensure SLA.
