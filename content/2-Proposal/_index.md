---
title: "Proposal"
date: 2026-01-09T15:00:11+07:00
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# NutriTrack Platform

## AWS Serverless Solution with Integrated AI for Nutrition Tracking

---

### 1. Project Summary

NutriTrack was built for Gen Z and Millennials in Vietnam—those who want to monitor their personal health in a practical, uncomplicated way. The platform targets over 1,000 active users, with a cross-platform mobile interface developed using React Native (Expo), allowing for meal data capture via photos, voice, or manual input.

The entire infrastructure complies with the AWS Well-Architected Framework, leveraging the serverless ecosystem (AWS Amplify Gen 2, AppSync, Lambda) and the Generative AI capabilities of Amazon Bedrock (model Qwen3-VL 235B). The result is a real-time nutrition tracking system with AI prediction, rich gamification mechanisms, and optimized operating costs—all protected by Amazon Cognito combined with Google OAuth2.

---

### 2. Context and Solution

#### Current Problem

Most current nutrition tracking applications require users to manually input data—boring and easy to give up on. Food databases are often poor in Vietnamese cuisine, lacking familiar dishes like pho, banh mi, or bun bo Hue. Furthermore, third-party nutrition APIs (Nutritionix, FatSecret) are expensive to scale and limited in data models.

Refrigerator management, daily meal planning, and utilizing available ingredients are also unresolved issues—leading to food waste and inconsistent eating habits.

#### How does NutriTrack solve this?

NutriTrack adopts a completely serverless AWS-native architecture:

- **AWS AppSync (GraphQL)** receives meal data via mutations and real-time subscriptions.

- **AWS Lambda** with 5 specialized functions handles all backend logic — from AI coordination and nutrition processing to friend management and image optimization.

- **Amazon DynamoDB** stores 6 core data models with the ability to scale automatically according to load.

- **Amazon Bedrock (Qwen3-VL 235B)** analyzes food images, suggests recipes, and generates intelligent nutritional data.

- **AWS Amplify Gen 2** combines React Native/Expo to create a smooth bilingual Vietnamese-English interface.

**Key Features:**

| Feature | Description |
| ----------- | ------- |
| 📸 Food Image Analysis | Take a picture — instantly receive a detailed nutrition chart via Vision AI |
| 🎙️ Voice Logging | AWS Transcribe converts speech into specific food items |
| 🍳 Smart Refrigerator | Tracks food, suggests menus from ingredients nearing their expiration date |
| 🤖 AI Coach "Ollie" | Personalized nutrition advice |
| 🎮 Gamification | 180-day Dragon evolution journey, pets, challenges |
| 👥 Social Features | Make friends, public Streak and Pet Score leaderboards |


---

### 3. Solution Architecture

Data migrates from React Native application → CloudFront/WAF (edge-to-edge security) → AppSync GraphQL API → Lambda handlers → DynamoDB. Heavy image processing tasks are delegated to ECS Fargate, while artificial intelligence is handled by Amazon Bedrock.

![NutriTrack Architecture](/images/architect.jpg)

#### AWS Services

| Service | Role |
| --------- | --------- |
| **AWS Amplify Gen 2** | Infrastructure orchestration using TypeScript CDK, CI/CD pipeline management, and 3 decoupled environments |
| **AWS AppSync** | GraphQL API with real-time subscriptions and owner-based authorization |
| **AWS Lambda** | 5 logic functions: `ai-engine`, `scan-image`, `process-nutrition`, `friend-request`, `resize-image` |
| **Amazon DynamoDB** | 6 NoSQL tables: `user`, `Food`, `FoodLog`, `FridgeItem`, `Friendship`, `UserPublicStats` |
| **Amazon Bedrock** | Qwen3-VL 235B (ap-southeast-2) handles 9 AI tasks (Vision, NLP, Coaching) |
| **AWS Transcribe** | Converts voice recordings (.m4a) from S3 to text for food processing |
| **Amazon S3** | Partitioned media storage: `incoming/` (temp), `voice/`, `avatar/`, `media/` (permanent) |
| **Amazon Cognito** | Centralized authentication: Email/OTP and Google OAuth2 federation |
| **Amazon ECS Fargate** | Runs FastAPI backend in VPC for high-intensity Vision AI processing |
| **AWS CloudWatch** | Centralized monitoring with Custom Metrics and Budget Alerts |
| **AWS Secrets Manager** | Secure API Keys for ECS and Bedrock Model IDs |

#### Specific Engineering Design

**Hybrid AI Processing** — The system prioritizes fuzzy match queries from the local database before fallback to Bedrock AI to ensure speed and cost savings.

**VPC Optimization** — The Lambda `scan-image` function is placed outside the VPC to connect directly to S3 and Bedrock, reducing latency by 90% and completely eliminating NAT Gateway costs for AI tasks.

**AI Engine (9 Tasks)** — The central coordinating brain:

- `analyzeFoodImage`: Analyzes food images directly.
- `voiceToFood`: Converts speech to nutritional data.
- `generateCoachResponse`: Responds to Coach Ollie's conversation.
- `generateFood`: Generates data for a non-food item.
Included in the database.
- `calculateMacros`: Calculates calorie targets based on body mass index.
- And other tasks such as: `fixFood`, `ollieCoachTip`, `generateRecipe`, `weeklyInsight`.

---

### 4. Technical Implementation

#### Technologies Used

- **Frontend:** React Native, Expo Router, TypeScript, Zustand (State), i18n.

- **Backend:** Amplify Gen 2, AppSync, DynamoDB, Lambda (Node.js 22).

- **AI/ML:** Bedrock (Qwen3-VL 235B), Transcribe, Vision AI on ECS Fargate.

- **DevOps:** Automated CI/CD, CDK Escape Hatches to resolve Table Discovery errors.

#### Development Roadmap

**Phase 1 — Infrastructure & Core**
Design a 6-table schema, configure Cognito Auth, and build basic UI for data entry.

**Phase 2 — AI & Voice Integration**
Deploy `aiEngine` in conjunction with Transcribe and Bedrock. Optimize automatic image resizing flow via S3 triggers.

**Phase 3 — Social & Gamification**
Build a friend system, leaderboards, and pet evolution logic (Minh Long Dragon) based on real-world movement sequences.

**Phase 4 — Stabilization & Scaling**
Handle Table Discovery bugs using CDK overrides, deploy production in the Sydney region (ap-southeast-2).

---

### 5. Schedule & Milestones (Dragon Journey)

The gamification system is designed with a 180-day journey:

- **Egg (Days 0-35)**: Starting state.

- **Newborn (Days 36-71)**: Hatching stage.

- **Teenager (Days 72-107)**: Size growth.

- **Adult (Days 108-143)**: Peak energy.

- **Legendary (Days 144+)**: Ultimate state with special privileges.

---

### 6. Risk Assessment & Handling

| Risks | Handling Methods |
| -------- | ---------- |
| **Table Discovery Error** | Inject the table name incorrectly via the Escape Hatches CDK (`addPropertyOverride`). |
| **ESCENT AI Costs** | Set Budget Alert to $80 and cache search results in DynamoDB. |
| **Hallucination (AI Error)** | Use Prompt Engineering with mandatory JSON Schema and allow manual editing. |
| **VPC Latency** | Move Lambda AI processing outside the VPC (Public) to optimize bandwidth. |

---

### 7. Expected Results

- **Performance**: Food posting time reduced from 3 minutes to 10 seconds thanks to AI.

- **Accuracy**: Achieve 95% accuracy for Vietnamese dishes thanks to the advanced Vision AI model.

- **Scalability**: Ready to handle 10,000+ users with near-zero maintenance costs when idle.

---

### 8. Next Steps

- **Expanding the Vietnamese Dishes DB**: Aim to reach 1,000+ verified dishes by the end of the year. - **EAS Build for iOS**: Completed the application packaging pipeline for the App Store.

- **Premium Package**: Launched advanced analytics features (Weekly Insights) to create sustainable revenue streams.

Issue: Fallback to manual data entry or fuzzy matching on approximately 200 pre-loaded Vietnamese dishes.

- When backend deployment fails: Rollback via CloudFormation/Amplify across 3 environments.

- When Qwen3-VL costs escalate: Switch to Claude Haiku or Llama on Bedrock.

---

### 9. Expected Results

#### Technical Improvements

- Logging time reduced from approximately 3 minutes (manual entry) to approximately 10 seconds (AI via image/voice).

- Serverless infrastructure ready to handle 10,000+ concurrent users with near-zero idle costs ($0.47/month for basic DynamoDB).

- Hybrid AI strategy (DynamoDB fuzzy match + Bedrock AI backup) balances cost efficiency and data accuracy.

#### Long-Term Value

- Building a verified Vietnamese food database, naturally expanding through user and AI contributions — currently ~200 dishes and growing.

- Reusable AI architecture patterns (prompt templates, Lambda orchestration, Bedrock integration) applicable to future health features.

- Social platform with gamification (pet evolution, daily streaks, challenges) promotes daily usage habits and long-term user retention.

---

### 10. Next Steps

**iOS Pipeline** — Migrating from Android-first MVP to iOS via EAS. Build cloud runners; activate macOS CI runner when user numbers are sufficient to offset costs.

**Expanding the Vietnamese Dish Database** — From an initial ~200 seeded items, collect AI-generated entries (`source: "AI Generated"` → `verified: false`) for the review team; the goal is 1,000+ verified dishes after the first year.

**Observability** — Connect 4 CloudWatch custom metrics to a centralized dashboard + set a Budget alarm at $80/month.

**Fallback Bedrock Cross-Region** — If the Qwen3-VL capacity at `ap-southeast-2` is limited, add a backup route to Claude Haiku `us-east-1` to ensure SLA.