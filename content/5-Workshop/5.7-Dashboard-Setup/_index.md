---
title: "Dashboard Setup"
date: 2026-01-09T16:03:43+07:00
weight: 40
chapter: false
pre: " <b> 5.7. </b> "
---

## Phase 5: React Native Frontend

In this phase, you will build the React Native (Expo) mobile application — the user-facing interface of NutriTrack with bilingual support, AI food analysis, and gamification.

#### Step 1: Initialize Expo Project

```bash
cd NutriTrack
npx create-expo-app frontend --template blank-typescript
cd frontend
npm install --legacy-peer-deps
```

Install core dependencies:

```bash
npm install aws-amplify @aws-amplify/react-native @react-native-community/netinfo
npm install @react-navigation/native @react-navigation/bottom-tabs expo-router
npm install zustand react-native-reanimated react-native-gesture-handler
npm install expo-camera expo-image-picker expo-av expo-secure-store expo-local-authentication
```

#### Step 2: Configure Amplify Client

Create `frontend/src/lib/amplify.ts`:

```typescript
import { Amplify } from 'aws-amplify';
import outputs from '../../amplify_outputs.json';

Amplify.configure(outputs);
```

Import this in your root layout to initialize Amplify on app start.

#### Step 3: App Architecture (Expo Router)

```
frontend/app/
├── _layout.tsx           # Root: Auth guard + LanguageProvider
├── (tabs)/
│   ├── _layout.tsx       # Tab bar with 5 tabs + center "+" button
│   ├── home.tsx          # Dashboard: daily macros, streak, pet
│   ├── kitchen.tsx       # Fridge inventory + AI recipe suggestions
│   ├── battle.tsx        # Challenges & leaderboard
│   ├── ai-coach.tsx      # Chat with AI coach "Ollie"
│   └── progress.tsx      # Weekly/monthly nutrition charts
├── add.tsx               # Food logging: photo, voice, manual, barcode
├── welcome.tsx           # Onboarding screen
├── login.tsx             # Email/password + Google OAuth
├── signup.tsx            # Registration form
└── verify-otp.tsx        # Email OTP verification
```

#### Step 4: State Management (Zustand)

Create stores in `frontend/src/store/`:

```typescript
// authStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface AuthState {
  user: any | null;
  isAuthenticated: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      isAuthenticated: false,
      login: async (email, password) => { /* Cognito signIn */ },
      logout: async () => { /* Cognito signOut */ },
    }),
    { name: 'auth-storage', storage: createJSONStorage(() => AsyncStorage) }
  )
);
```

#### Step 5: AI Service Integration

Create `frontend/src/services/aiService.ts`:

```typescript
import { generateClient } from 'aws-amplify/data';
import type { Schema } from '../../amplify/data/resource';

const client = generateClient<Schema>();

export async function analyzeFoodImage(imageBase64: string) {
  const result = await client.queries.aiEngine({
    action: 'analyzeFoodImage',
    payload: JSON.stringify({ imageBase64 }),
  });
  return extractAndParseJSON(result.data);
}

// Helper: Extract JSON from AI response (handles code block wrapping)
function extractAndParseJSON(response: string) {
  const jsonMatch = response.match(/```json\n?([\s\S]*?)\n?```/);
  if (jsonMatch) return JSON.parse(jsonMatch[1]);
  return JSON.parse(response);
}
```

#### Step 6: Bilingual Support (i18n)

Create `frontend/src/i18n/LanguageProvider.tsx`:

```typescript
const translations = {
  en: { home: 'Home', kitchen: 'Kitchen', addFood: 'Add Food', ... },
  vi: { home: 'Trang chủ', kitchen: 'Nhà bếp', addFood: 'Thêm Món', ... },
};

export function useAppLanguage() {
  const [language, setLanguage] = useState<'en' | 'vi'>('vi');
  const t = (key: string) => translations[language][key] || key;
  return { language, setLanguage, t };
}
```

#### Step 7: Design System

**Color Palette:**
- Primary: Dark Navy `#1B2838`
- Accent: Green `#2ECC71`
- Background: `#F8F9FA`
- Card: `#FFFFFF` with subtle shadow

**Typography:** Inter (primary), Roboto (fallback)

#### Verification

```bash
cd frontend
npm start
# Scan QR code with Expo Go app
```

1. Verify auth flow: Welcome → Login → Home (with Cognito)
2. Test food photo analysis: Camera → Snap photo → Check AI response
3. Verify bilingual toggle works on all screens

> 🎯 **Checkpoint:** The mobile app launches, authenticates via Cognito, and successfully analyzes food photos via the AI engine.
