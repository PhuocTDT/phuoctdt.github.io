---
title: "Realtime Subscriptions"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.6.2. </b> "
---

# 5.6.2 Realtime Subscriptions

Every `a.model(...)` defined in `data/resource.ts` automatically gets three GraphQL subscriptions — `onCreate`, `onUpdate`, `onDelete` — exposed by AppSync with zero extra backend code. This page covers how NutriTrack uses them, how they're filtered, and what to watch out for.

## What Amplify generates for free

For each model, Amplify emits a subscription triple into the schema:

```graphql
type Subscription {
  onCreateFoodLog(filter: ModelSubscriptionFoodLogFilterInput, owner: String): FoodLog
  onUpdateFoodLog(filter: ModelSubscriptionFoodLogFilterInput, owner: String): FoodLog
  onDeleteFoodLog(filter: ModelSubscriptionFoodLogFilterInput, owner: String): FoodLog
  # ... same triple for Friendship, UserPublicStats, ChallengeParticipant, FridgeItem
}
```

Authorization follows the model's rule set. A model with `allow.owner()` only pushes an event to a client whose authenticated owner matches the row's `owner` field — users never see each other's private rows over a subscription.

## How AppSync delivers them

- Client opens a single long-lived WebSocket to `<appsync-endpoint>/graphql/realtime`.
- Each call to `.subscribe()` registers a subscription on that connection with a filter expression.
- A DynamoDB Streams record on the underlying table triggers an AppSync invalidation.
- AppSync evaluates the subscription filter server-side and pushes the matching rows to each subscribed client.
- On network drop, the Amplify JS client auto-reconnects and re-registers all active subscriptions.

Under the hood this is MQTT-over-WebSocket. From your code you never see that — you just get a typed Observable.

## Basic usage pattern

```tsx
import { useEffect } from 'react';
import { generateClient } from 'aws-amplify/api';
import type { Schema } from '../../amplify/data/resource';
import { useAuthStore } from '@/src/store/authStore';
import { useMealStore } from '@/src/store/mealStore';

const client = generateClient<Schema>();

export function useFoodLogRealtime() {
  const userId = useAuthStore((s) => s.userId);
  const addLog = useMealStore((s) => s.addLog);

  useEffect(() => {
    if (!userId) return;

    const sub = client.models.FoodLog.onCreate({
      filter: { owner: { eq: userId } },
    }).subscribe({
      next: (log) => addLog(log),
      error: (err) => console.error('[FoodLog.onCreate]', err),
    });

    return () => sub.unsubscribe();
  }, [userId]);
}
```

Three things to notice:

1. The subscription is scoped by filter — the client only receives events for rows it should see anyway. This is defense in depth; authorization is still enforced by AppSync.
2. `sub.unsubscribe()` must be called in the effect cleanup. Leaking subscriptions across screen mounts is the most common production bug with this API — each mount opens a new server-side listener and costs real money.
3. The callback runs on the JS thread. Keep it light — dispatch to a Zustand store, don't do heavy work.

## Where NutriTrack uses subscriptions

| Flow | Model | Event | Purpose |
|---|---|---|---|
| Cross-device meal sync | `FoodLog` | `onCreate`, `onDelete` | User logs a meal on their phone; the same account on a tablet updates instantly. |
| Friend request notifications | `Friendship` | `onUpdate` | When user B accepts A's request, user A's row flips to `accepted` and A gets an in-app notification. |
| Leaderboard updates | `UserPublicStats` | `onUpdate` | Friends' streaks and pet scores refresh live on the Battle tab. |
| Challenge join pings | `ChallengeParticipant` | `onCreate` | Creator sees new joiners appear as they opt in. |
| Fridge inventory | `FridgeItem` | `onCreate`, `onUpdate`, `onDelete` | Shared household fridge stays consistent across family devices. |

Each of these is a few lines of client code — no resolvers, no Lambdas, no DynamoDB Streams handlers to maintain.

## Friendship acceptance notification

```tsx
useEffect(() => {
  if (!currentOwner) return;

  const sub = client.models.Friendship.onUpdate({
    filter: {
      owner: { eq: currentOwner },
      direction: { eq: 'sent' },
      status: { eq: 'accepted' },
    },
  }).subscribe({
    next: (friendship) => {
      showToast(`${friendship.friend_name} accepted your request`);
      useFriendStore.getState().refresh();
    },
    error: (err) => console.error('[Friendship.onUpdate]', err),
  });

  return () => sub.unsubscribe();
}, [currentOwner]);
```

This is the client side of the flow started by the `friendRequest` Lambda in 5.6.1. When the Lambda flips user A's `sent` row to `accepted` via `TransactWriteItems`, AppSync picks up the DynamoDB change and pushes it to A's subscribed client.

## Filter gotchas

- **Equality only.** Generated subscription filters support `eq`, `ne`, `in`, `notIn`, `contains`, `notContains`, `beginsWith`. They do not support arithmetic, `between` on numbers, or nested `and`/`or` that spans unrelated fields in the same depth cheaply. If you need a complex filter, either subscribe broader and filter client-side, or add a custom resolver with a pipeline that emits to a dedicated channel.
- **Filters run server-side** — they don't save on authorization (AppSync still checks owner rules) but they do save on bandwidth and on your JS main thread.
- **`contains` on arrays** works, but matches the full string value, not individual array elements. For array membership, denormalize to a scalar.

## Reconnection behavior

The Amplify JS client exposes `ConnectionState` via a listener:

```typescript
import { CONNECTION_STATE_CHANGE } from 'aws-amplify/api';
import { Hub } from 'aws-amplify/utils';

Hub.listen('api', (data) => {
  if (data.payload.event === CONNECTION_STATE_CHANGE) {
    console.log('[AppSync] connection:', data.payload.data.connectionState);
  }
});
```

States you'll see: `Connecting` → `Connected` → (network drop) → `ConnectionDisrupted` → `Connecting` → `Connected`. On re-connect the client replays its subscription registrations, so the application code doesn't need to know the socket died.

Edge cases:

- If the app is backgrounded on iOS for >30 seconds, the OS may kill the WebSocket. The Amplify client reconnects on foreground, but **you lose events that happened during the gap**. Reconcile on resume by refetching the relevant list query.
- On Android, the Doze mode behaves similarly. Same reconciliation rule applies.

## Cost model

AppSync pricing has two relevant dimensions for subscriptions:

- **Real-time updates** — $2.00 per million messages delivered (ap-southeast-2, 2025 pricing; check the console for current rate).
- **Connection minutes** — $0.08 per million minutes of open connections.

What this means for NutriTrack at ~1000 DAU:

- Each user averages 4 minutes/day connected and receives ~20 events/day → well under $1/month.
- Each user connected 24/7 → ~$3.50/month per user. Avoid.

Rules to keep the bill flat:

1. Unsubscribe on screen unmount, always.
2. Scope filters as tight as the UI needs — subscribe to `onCreate FoodLog` for `owner: currentUser`, not for all FoodLogs.
3. Do not open a subscription from a deeply-nested component that remounts on every prop change — hoist it to the screen root or into a Zustand store initializer.

## Testing the subscription flow

A two-device smoke test is the fastest way to validate the whole pipeline end to end.

1. Start two Expo clients — physical phone + simulator, or two simulators.
2. Sign into the same user account on both (easier for the first test; for cross-user, use friends that have accepted each other).
3. On device A, call `client.models.FoodLog.create({ ... })`.
4. On device B, the `onCreate` subscription fires. If the UI is wired correctly, the new log appears in the meal list within ~500 ms.
5. Delete from A; the row disappears on B.

If nothing arrives on B:

- Check the browser/Metro devtools for WebSocket connection errors.
- Confirm both clients are in the same sandbox/branch (Amplify environments are isolated — a `sandbox-A` subscription won't see writes from `sandbox-B`).
- Verify the model's authorization rule actually permits the subscriber to read the row. Owner-auth rows are only streamed to the owning user.

## Back to the index

- [Section 5.6 — API & Social](../)
- [5.6.1 — FriendRequest Lambda](../5.6.1-FriendRequest/)
