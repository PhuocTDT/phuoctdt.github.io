---
title: "UIComponents"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.7.2. </b> "
---

### 5.7.2 UI Components Mapping

Our UI heavily features Gamification (Streaks, Ollie Pet) and AI interactions.

#### Core Flow
1. **Camera Scanner**: Utilizes `expo-camera` to capture plates and streams base64 strictly to S3 bucket.
2. **AI Result Modal**: Subscribes to AppSync for instantaneous mutation results from the Bedrock AI process.
3. **Voice Logger**: Incorporates `expo-av` for Audio recording, shipping `.wav` to AWS Transcribe endpoint.
4. **Zustand State**: Stores local User Session, JWT Tokens, and caching to significantly bypass excessive DynamoDB GET queries.

Proceed to deployment once local simulators emulate successfully.