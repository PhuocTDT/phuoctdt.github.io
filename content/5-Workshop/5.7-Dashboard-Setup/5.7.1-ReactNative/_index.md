---
title: "ReactNative"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.7.1. </b> "
---

### 5.7.1 React Native (Expo) Setup

NutriTrack's frontend is built purely on React Native and Expo for multi-platform iOS/Android support.

#### Step 1: Initialize Expo
```bash
npx create-expo-app nutritrack-mobile -t expo-template-blank-typescript
cd nutritrack-mobile
```

#### Step 2: Install Amplify Libraries
```bash
npm install aws-amplify @aws-amplify/react-native
```

#### Step 3: Configure Provider
In your `App.tsx`, initialize the backend resource config.

```typescript
import { Amplify } from 'aws-amplify';
import outputs from './amplify_outputs.json';

Amplify.configure(outputs);

export default function App() {
  return <SafeAreaView>...</SafeAreaView>;
}
```