---
title: "FriendRequest"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.6.1. </b> "
---

# 5.6.1 FriendRequest Lambda

A Lambda-backed custom mutation that handles every state transition of a friendship: send, accept, decline, remove, block. One entry point, five actions, one atomic write per action.

## Goal

- Expose `Mutation.friendRequest(action, payload)` on the AppSync API.
- Keep `Friendship` rows consistent between both sides of any relationship.
- Inject exact DynamoDB table names via env vars — no runtime `ListTables` guessing.

## Files touched

- `backend/amplify/friend-request/resource.ts` — function definition.
- `backend/amplify/friend-request/handler.ts` — business logic.
- `backend/amplify/data/resource.ts` — exposes the mutation on the schema.
- `backend/amplify/backend.ts` — IAM policy + env var escape hatches.

## Function definition

`backend/amplify/friend-request/resource.ts`:

```typescript
import { defineFunction } from '@aws-amplify/backend';

export const friendRequest = defineFunction({
  name: 'friend-request',
  entry: './handler.ts',
  runtime: 22,
  memoryMB: 256,
  timeoutSeconds: 15,
  resourceGroupName: 'data',
});
```

Why these numbers:

- `runtime: 22` — Node.js 22, which ships AWS SDK v3 on the managed runtime and gets ARM64 by default on Amplify Gen 2 (better price/performance than x86).
- `memoryMB: 256` — DynamoDB calls are I/O bound. 256 MB is enough for the `@aws-sdk/lib-dynamodb` document client and leaves margin for JSON parsing.
- `timeoutSeconds: 15` — a `TransactWriteItems` over two rows completes in <200 ms typical; 15 s covers cold starts plus any DynamoDB retry.
- `resourceGroupName: 'data'` — puts the Lambda in the same CloudFormation nested stack as `data/resource.ts` so it can cleanly reference the DynamoDB tables without triggering a cross-stack circular dependency.

## Schema wiring

In `backend/amplify/data/resource.ts`:

```typescript
import { friendRequest } from '../friend-request/resource';

const schema = a.schema({
  // ... models ...

  friendRequest: a
    .mutation()
    .arguments({
      action: a.string().required(),
      payload: a.string().required(),
    })
    .returns(a.string())
    .handler(a.handler.function(friendRequest))
    .authorization((allow) => [allow.authenticated()]),
});
```

Notes:

- `payload` is a stringified JSON blob, not a typed object. Reason: the five actions take different shapes (`{ friend_code }` vs `{ friendship_id }`) and a single typed schema would either be a union (not supported in Amplify's type system) or a bag of optional fields (loses compile-time safety). Stringified JSON is the pragmatic compromise — the Lambda parses it.
- `allow.authenticated()` — only Cognito-signed callers can invoke. The Lambda still has to validate `event.identity.sub` before trusting anything.

## Handler — entry point

`backend/amplify/friend-request/handler.ts`:

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import {
  DynamoDBDocumentClient,
  GetCommand,
  TransactWriteCommand,
  QueryCommand,
  ScanCommand,
} from '@aws-sdk/lib-dynamodb';
import { randomUUID } from 'node:crypto';

const REGION = process.env.AWS_REGION || 'ap-southeast-2';
const client = new DynamoDBClient({ region: REGION });
const docClient = DynamoDBDocumentClient.from(client);

// Table names injected by CDK at deploy time — correct for each environment
const USER_TABLE = process.env.USER_TABLE_NAME;
const FRIENDSHIP_TABLE = process.env.FRIENDSHIP_TABLE_NAME;

function getTableNames() {
  if (!USER_TABLE) throw new Error('USER_TABLE_NAME env var not set');
  if (!FRIENDSHIP_TABLE) throw new Error('FRIENDSHIP_TABLE_NAME env var not set');
  return { userTable: USER_TABLE, friendshipTable: FRIENDSHIP_TABLE };
}

type FriendAction =
  | 'sendRequest'
  | 'acceptRequest'
  | 'declineRequest'
  | 'removeFriend'
  | 'blockFriend';

export const handler = async (event: any): Promise<string> => {
  const { action, payload } = event.arguments;
  const caller = getCallerIdentity(event);
  const params = JSON.parse(payload || '{}');

  try {
    switch (action as FriendAction) {
      case 'sendRequest':
        return JSON.stringify(await sendRequest(caller, params.friend_code));
      case 'acceptRequest':
        return JSON.stringify(await acceptRequest(caller, params.friendship_id));
      case 'declineRequest':
        return JSON.stringify(await declineRequest(caller, params.friendship_id));
      case 'removeFriend':
        return JSON.stringify(await removeFriend(caller, params.friendship_id));
      case 'blockFriend':
        return JSON.stringify(await blockFriend(caller, params.friendship_id));
      default:
        return JSON.stringify({ success: false, error: `Unknown action: ${action}` });
    }
  } catch (error: any) {
    return JSON.stringify({ success: false, error: error.message || 'Internal error' });
  }
};
```

### Caller identity

AppSync injects `event.identity` with the caller's Cognito claims. We extract `sub` (stable user ID) and build Amplify's owner format `sub::cognitoUsername`:

```typescript
function getCallerIdentity(event: any) {
  const sub = event.identity?.sub || event.identity?.claims?.sub;
  if (!sub) throw new Error('Unauthorized: no user identity');
  const username =
    event.identity?.username ||
    event.identity?.claims?.['cognito:username'] ||
    sub;
  return { sub, owner: `${sub}::${username}` };
}
```

The `owner` format matters because Amplify's owner-based authorization stores rows keyed by that exact string. When the frontend subscribes to `Friendship.onUpdate` with `{ filter: { owner: { eq: currentOwner } } }`, the string must match byte-for-byte.

## Action 1 — sendRequest

1. Look up the target user by `friend_code` (6-char alphanumeric, stored on the `user` model).
2. Reject self-friending.
3. Reject if an existing friendship already exists (any status).
4. Reject if the caller has 20+ pending sent requests.
5. Create **two mirror rows** via `TransactWriteCommand`.

```typescript
async function sendRequest(caller: CallerIdentity, friendCode: string) {
  if (!friendCode) throw new Error('friend_code is required');
  const { userTable, friendshipTable } = getTableNames();

  const friendUser = await findUserByFriendCode(userTable, friendCode);
  if (!friendUser) throw new Error('User not found for this code');
  if (friendUser.user_id === caller.sub) throw new Error('Cannot friend yourself');

  const callerUser = await getUserById(userTable, caller.sub);
  if (!callerUser) throw new Error('Caller profile not found');

  const existing = await findExistingFriendship(
    friendshipTable,
    caller.owner,
    friendUser.user_id,
  );
  if (existing) {
    if (existing.status === 'accepted') throw new Error('Already friends');
    if (existing.status === 'pending') throw new Error('Request already pending');
    if (existing.status === 'blocked') throw new Error('Cannot send request');
  }

  const pendingCount = await countPendingRequests(friendshipTable, caller.owner);
  if (pendingCount >= 20) throw new Error('Pending request limit reached (20)');

  const friendOwner = friendUser.owner;
  if (!friendOwner) throw new Error('Friend user record missing owner field');

  const now = new Date().toISOString();
  const sentId = randomUUID();
  const receivedId = randomUUID();

  await docClient.send(new TransactWriteCommand({
    TransactItems: [
      {
        Put: {
          TableName: friendshipTable,
          Item: {
            id: sentId,
            owner: caller.owner,
            friend_id: friendUser.user_id,
            friend_code: friendCode,
            friend_name: friendUser.display_name || friendUser.email || 'User',
            friend_avatar: friendUser.avatar_url || null,
            status: 'pending',
            direction: 'sent',
            linked_id: receivedId,
            createdAt: now,
            updatedAt: now,
          },
        },
      },
      {
        Put: {
          TableName: friendshipTable,
          Item: {
            id: receivedId,
            owner: friendOwner,
            friend_id: caller.sub,
            friend_code: callerUser.friend_code || '',
            friend_name: callerUser.display_name || callerUser.email || 'User',
            friend_avatar: callerUser.avatar_url || null,
            status: 'pending',
            direction: 'received',
            linked_id: sentId,
            createdAt: now,
            updatedAt: now,
          },
        },
      },
    ],
  }));

  return { success: true, friendship_id: sentId };
}
```

Key points:

- The two `Put` calls are in a single `TransactWriteItems` — DynamoDB commits both or neither. No partial state.
- `linked_id` on each row points to the UUID of the mirror row. Later actions use this to touch the other side without a secondary scan.
- `findUserByFriendCode` uses the `usersByFriend_code` GSI (the secondary index defined on the `user` model).

## Action 2 — acceptRequest

Caller supplies the `friendship_id` of their **received** row. Lambda updates both rows to `accepted` atomically:

```typescript
async function acceptRequest(caller: CallerIdentity, friendshipId: string) {
  const { friendshipTable } = getTableNames();
  const receivedRecord = await getFriendshipById(friendshipTable, friendshipId);
  if (!receivedRecord) throw new Error('Friendship record not found');

  const ownerSub = receivedRecord.owner?.split('::')[0];
  if (ownerSub !== caller.sub) throw new Error('Unauthorized');
  if (receivedRecord.direction !== 'received') throw new Error('Can only accept received requests');
  if (receivedRecord.status !== 'pending') throw new Error('Request is not pending');

  const linkedId = receivedRecord.linked_id;
  if (!linkedId) throw new Error('Linked friendship record not found');

  const now = new Date().toISOString();
  await docClient.send(new TransactWriteCommand({
    TransactItems: [
      {
        Update: {
          TableName: friendshipTable,
          Key: { id: friendshipId },
          UpdateExpression: 'SET #s = :status, updatedAt = :now',
          ExpressionAttributeNames: { '#s': 'status' },
          ExpressionAttributeValues: { ':status': 'accepted', ':now': now },
        },
      },
      {
        Update: {
          TableName: friendshipTable,
          Key: { id: linkedId },
          UpdateExpression: 'SET #s = :status, updatedAt = :now',
          ExpressionAttributeNames: { '#s': 'status' },
          ExpressionAttributeValues: { ':status': 'accepted', ':now': now },
        },
      },
    ],
  }));

  return { success: true };
}
```

Authorization check: `receivedRecord.owner` is of the form `sub::username`. We split on `::` and compare the `sub` portion with `caller.sub` — this prevents user A from accepting a request that lives on user B's row.

## Action 3 — declineRequest

Deletes both rows. `removeFriend` is the same code — unfriending is structurally identical to declining a pending request.

```typescript
async function declineRequest(caller: CallerIdentity, friendshipId: string) {
  const { friendshipTable } = getTableNames();
  const record = await getFriendshipById(friendshipTable, friendshipId);
  if (!record) throw new Error('Friendship record not found');

  const ownerSub = record.owner?.split('::')[0];
  if (ownerSub !== caller.sub) throw new Error('Unauthorized');

  const linkedId = record.linked_id;
  const transactItems: any[] = [
    { Delete: { TableName: friendshipTable, Key: { id: friendshipId } } },
  ];
  if (linkedId) {
    transactItems.push({
      Delete: { TableName: friendshipTable, Key: { id: linkedId } },
    });
  }

  await docClient.send(new TransactWriteCommand({ TransactItems: transactItems }));
  return { success: true };
}

async function removeFriend(caller: CallerIdentity, friendshipId: string) {
  return declineRequest(caller, friendshipId);
}
```

## Action 5 — blockFriend

Sets the caller's row to `blocked` and **deletes** the other side. After a block, the blocked user has no database trace of the friendship — they won't see it in their list, they can't re-send a request (the scan-based duplicate check won't find anything on their side), but the blocker still has a record and can unblock if needed.

```typescript
async function blockFriend(caller: CallerIdentity, friendshipId: string) {
  const { friendshipTable } = getTableNames();
  const record = await getFriendshipById(friendshipTable, friendshipId);
  if (!record) throw new Error('Friendship record not found');

  const ownerSub = record.owner?.split('::')[0];
  if (ownerSub !== caller.sub) throw new Error('Unauthorized');

  const linkedId = record.linked_id;
  const now = new Date().toISOString();

  const transactItems: any[] = [
    {
      Update: {
        TableName: friendshipTable,
        Key: { id: friendshipId },
        UpdateExpression: 'SET #s = :status, updatedAt = :now',
        ExpressionAttributeNames: { '#s': 'status' },
        ExpressionAttributeValues: { ':status': 'blocked', ':now': now },
      },
    },
  ];
  if (linkedId) {
    transactItems.push({
      Delete: { TableName: friendshipTable, Key: { id: linkedId } },
    });
  }
  await docClient.send(new TransactWriteCommand({ TransactItems: transactItems }));
  return { success: true };
}
```

## Env var injection — the CDK escape hatch

Amplify Gen 2 does not give you a first-class API to pass a DynamoDB table name into a Lambda function. The table is created by the `data` resource and its physical name contains a CloudFormation-generated suffix like `Friendship-abc123xyz-NONE`. You need that exact string at runtime.

An older pattern used `ListTablesCommand` at cold start to discover tables by prefix. That fails as soon as you have two Amplify sandboxes active in the same account — both list `Friendship-*` tables and the Lambda picks whichever DynamoDB returns first.

The fix is to use CDK's `addPropertyOverride` escape hatch to inject the exact logical table names as environment variables at deploy time. In `backend/amplify/backend.ts`:

```typescript
import * as iam from 'aws-cdk-lib/aws-iam';
import * as cdk from 'aws-cdk-lib';
import { friendRequest } from './friend-request/resource';

const backend = defineBackend({ auth, data, friendRequest, /* ... */ });

const friendRequestLambda = backend.friendRequest.resources.lambda;

// IAM — grant access to both tables and their GSIs
friendRequestLambda.addToRolePolicy(new iam.PolicyStatement({
  effect: iam.Effect.ALLOW,
  actions: [
    'dynamodb:GetItem',
    'dynamodb:PutItem',
    'dynamodb:UpdateItem',
    'dynamodb:DeleteItem',
    'dynamodb:Query',
    'dynamodb:Scan',
    'dynamodb:BatchGetItem',
    'dynamodb:BatchWriteItem',
    'dynamodb:DescribeTable',
    'dynamodb:TransactWriteItems',
  ],
  resources: [
    'arn:aws:dynamodb:*:*:table/user-*',
    'arn:aws:dynamodb:*:*:table/user-*/index/*',
    'arn:aws:dynamodb:*:*:table/Friendship-*',
    'arn:aws:dynamodb:*:*:table/Friendship-*/index/*',
  ],
}));

// Inject exact table names — CDK resolves the right name per environment
const cfnFn = friendRequestLambda.node.defaultChild as cdk.aws_lambda.CfnFunction;
cfnFn.addPropertyOverride(
  'Environment.Variables.USER_TABLE_NAME',
  backend.data.resources.tables['user'].tableName,
);
cfnFn.addPropertyOverride(
  'Environment.Variables.FRIENDSHIP_TABLE_NAME',
  backend.data.resources.tables['Friendship'].tableName,
);
```

Why this works:

- `backend.data.resources.tables['user'].tableName` returns a CDK `Token` (a lazy reference). At synth time CloudFormation replaces it with a `Ref` to the actual `AWS::DynamoDB::Table` resource.
- Each sandbox/branch gets its own token → its own resolved table name → no cross-environment leakage.
- Because the Lambda lives in `resourceGroupName: 'data'`, both resources are in the same nested stack and no circular dependency is introduced.

IAM note: resources are scoped with wildcards (`table/user-*`) rather than the exact token because fine-grained resource references to a sibling resource in the same stack would also introduce a circular dependency on some Amplify versions.

## Frontend call site

From any authenticated screen on the mobile app:

```typescript
import { generateClient } from 'aws-amplify/api';
import type { Schema } from '../amplify/data/resource';

const client = generateClient<Schema>();

async function sendFriendRequest(code: string) {
  const result = await client.mutations.friendRequest({
    action: 'sendRequest',
    payload: JSON.stringify({ friend_code: code }),
  });
  const data = JSON.parse(result.data ?? '{}');
  if (!data.success) throw new Error(data.error);
  return data.friendship_id as string;
}

async function acceptFriendRequest(friendshipId: string) {
  await client.mutations.friendRequest({
    action: 'acceptRequest',
    payload: JSON.stringify({ friendship_id: friendshipId }),
  });
}
```

## Error cases worth knowing

| Error | Cause | Fix |
|---|---|---|
| `USER_TABLE_NAME env var not set` | Env var injection skipped — sandbox not redeployed after editing `backend.ts`. | Restart `npx ampx sandbox`. |
| `ConditionalCheckFailedException` | Two clients sent the same request at the same moment. | Retry with exponential backoff, then surface "already pending" to the user. |
| `TransactionCanceledException` | One of the two `Put` calls hit a throughput limit or item size cap. | Check CloudWatch for the cancellation reason; the message includes a reason code per item. |
| `Unauthorized` on accept | Caller's `sub` doesn't match `owner` on the row. | Frontend must pass the `friendship_id` from a row the caller actually owns. |

## Verification

1. Sign in as user A in one Expo client, sign in as user B in another.
2. From A, call `sendRequest` with B's `friend_code`.
3. In DynamoDB console, query the `Friendship` table. You should see **two rows** with the same `linked_id`s pointing at each other, both `status: pending`, one `direction: sent`, the other `direction: received`.
4. From B, call `acceptRequest` with the received row's `id`. Re-scan the table — both rows now `status: accepted`.
5. Inspect CloudWatch log group `/aws/lambda/amplify-<app>-friend-request-<sandbox>` for the execution trace.

![Friend request flow](/images/workshop/friend-request-flow.png)

## Next

- [5.6.2 — Realtime Subscriptions](../5.6.2-Realtime-Subscriptions/)
