---
title: "Overview"
date: 2026-01-09T16:03:43+07:00
weight: 10
chapter: false
pre: " <b> 5.1. </b> "
---

## Introduction
This workshop is designed to guide you through the process of building and deploying **NutriTrack** — a modern nutrition tracking platform that combines the power of **AWS Amplify Gen 2**, **Amazon Bedrock**, and **Amazon ECS Fargate**. Based 100% on a real-world production codebase, this workshop will help you master serverless infrastructure and explore how to optimally integrate high-performance AI tasks into practical applications.

## Overall Architecture

![NutriTrack Architecture](/images/architect.jpg)

## Infrastructure Summary

Here are the key components you will deploy throughout the Workshop:

### 1. Database (Amazon DynamoDB)
| Table Name | Primary Function | Notes |
| :--- | :--- | :--- |
| **`Food`** | Food & Nutrition Catalog | Contains Macro/Micro-nutrient data for foods. |
| **`user`** | User Profile | Stores height, weight, and calorie goals. |
| **`FoodLog`** | Consumption Diary | Meal history and intake nutrition metrics. |
| **`FridgeItem`** | Inventory Management | Tracks stock in the refrigerator. |
| **`Friendship`** | Social System | Manages friend requests and connections. |
| **`UserPublicStats`** | Public Metrics | Stores Streak and Pet Level for social viewing. |

### 2. Logic Layer (AWS Lambda Functions)
1. **`ai-engine`**: The "brain" that invokes Bedrock (AI) and coordinates Transcribe.
2. **`scan-image`**: Connects to ECS Fargate for specialized image analysis.
3. **`process-nutrition`**: Calculates nutrition data and handles AI fallback logic.
4. **`friend-request`**: Handles friendship and social interaction logic.
5. **`resize-image`**: Automatically optimizes images upon S3 upload.

### 3. AI Capabilities
* **Voice Logging**: Converts Vietnamese speech to text to extract food entities.
* **Photo Analysis**: Identifies components and estimates portions via images.
* **AI Coach (Ollie)**: Provides personalized AI-driven nutrition advice.

---

[Continue to 5.2 Prerequisites](../5.2-Prerequiste/)
