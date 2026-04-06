---
title: "FriendRequest"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.6.1. </b> "
---

# 5.6.1 FriendRequest Lambda

Mutation tùy chỉnh chạy bằng Lambda, xử lý mọi chuyển trạng thái của một quan hệ bạn bè: gửi, chấp nhận, từ chối, gỡ, chặn. Một điểm vào, năm action, mỗi action một lần ghi có tính nguyên tử.

## Mục tiêu

- Phơi bày `Mutation.friendRequest(action, payload)` trên AppSync API.
- Giữ các hàng `Friendship` nhất quán giữa hai phía của mọi quan hệ.
- Tiêm tên bảng DynamoDB chính xác qua env var — không đoán mò bằng `ListTables` lúc runtime.

## File liên quan

- `backend/amplify/friend-request/resource.ts` — định nghĩa function.
- `backend/amplify/friend-request/handler.ts` — logic nghiệp vụ.
- `backend/amplify/data/resource.ts` — đăng ký mutation trên schema.
- `backend/amplify/backend.ts` — policy IAM + escape hatch tiêm env var.

## Định nghĩa function

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

Vì sao các con số này:

- `runtime: 22` — Node.js 22, đi kèm AWS SDK v3 trên managed runtime và mặc định ARM64 trên Amplify Gen 2 (giá/hiệu năng tốt hơn x86).
- `memoryMB: 256` — các lệnh DynamoDB là I/O bound. 256 MB đủ cho `@aws-sdk/lib-dynamodb` document client và còn dư cho parse JSON.
- `timeoutSeconds: 15` — một `TransactWriteItems` trên hai hàng thường hoàn thành trong <200 ms; 15 giây phủ cold start cộng retry của DynamoDB.
- `resourceGroupName: 'data'` — đặt Lambda cùng nested CloudFormation stack với `data/resource.ts` để tham chiếu bảng DynamoDB mà không gây circular dependency.

## Gắn vào schema

Trong `backend/amplify/data/resource.ts`:

```typescript
import { friendRequest } from '../friend-request/resource';

const schema = a.schema({
  // ... các model ...

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

Chú ý:

- `payload` là một chuỗi JSON, không phải object có kiểu. Lý do: năm action có shape khác nhau (`{ friend_code }` vs `{ friendship_id }`) và schema có kiểu duy nhất sẽ phải union (Amplify chưa hỗ trợ) hoặc bag field optional (mất type-safety). Chuỗi JSON là thỏa hiệp thực dụng — Lambda tự parse.
- `allow.authenticated()` — chỉ caller đã đăng nhập Cognito mới gọi được. Lambda vẫn phải xác thực `event.identity.sub` trước khi tin bất kỳ thứ gì.

## Handler — điểm vào

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

// Tên bảng được CDK tiêm lúc deploy — đúng cho từng môi trường
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

### Danh tính caller

AppSync tiêm `event.identity` chứa các claim Cognito của caller. Ta lấy `sub` (user id ổn định) và dựng định dạng owner `sub::cognitoUsername` của Amplify:

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

Định dạng `owner` quan trọng vì authorization kiểu owner của Amplify lưu hàng theo chuỗi đó chính xác từng byte. Khi frontend subscribe `Friendship.onUpdate` với `{ filter: { owner: { eq: currentOwner } } }`, chuỗi phải khớp tuyệt đối.

## Action 1 — sendRequest

1. Tra user đích qua `friend_code` (6 ký tự alphanumeric, lưu trên model `user`).
2. Chặn tự kết bạn.
3. Chặn nếu đã có quan hệ bạn bè (bất kỳ status).
4. Chặn nếu caller đã có ≥20 lời mời đang chờ.
5. Tạo **hai hàng đối xứng** bằng `TransactWriteCommand`.

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

Điểm chính:

- Hai `Put` nằm trong một `TransactWriteItems` — DynamoDB commit cả hai hoặc không commit gì. Không có trạng thái nửa vời.
- `linked_id` trên mỗi hàng trỏ tới UUID của hàng đối xứng. Các action sau dùng nó để chạm phía còn lại mà không cần scan phụ.
- `findUserByFriendCode` dùng GSI `usersByFriend_code` (secondary index khai báo trên model `user`).

## Action 2 — acceptRequest

Caller truyền `friendship_id` của hàng **received** thuộc về mình. Lambda cập nhật cả hai hàng sang `accepted` đồng thời:

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

Kiểm tra authorization: `receivedRecord.owner` có dạng `sub::username`. Ta tách theo `::` và so phần `sub` với `caller.sub` — tránh trường hợp user A chấp nhận lời mời nằm trên hàng của user B.

## Action 3 — declineRequest

Xoá cả hai hàng. `removeFriend` dùng y hệt code — gỡ bạn về mặt cấu trúc giống hệt từ chối lời mời đang chờ.

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

Chuyển hàng của caller sang `blocked` và **xoá** hàng phía bên kia. Sau khi bị chặn, người bị chặn không còn dấu vết database của quan hệ — họ không thấy trong danh sách, không gửi lại lời mời được (kiểm tra trùng bằng scan không tìm thấy gì phía họ), nhưng bên chặn vẫn còn record và có thể bỏ chặn sau.

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

## Tiêm env var — CDK escape hatch

Amplify Gen 2 không có API chính thức để truyền tên bảng DynamoDB vào Lambda. Bảng do resource `data` tạo và tên vật lý có hậu tố CloudFormation sinh, ví dụ `Friendship-abc123xyz-NONE`. Lambda cần chuỗi đó chính xác lúc runtime.

Pattern cũ dùng `ListTablesCommand` lúc cold start để dò bảng theo prefix. Cách đó fail ngay khi có hai Amplify sandbox cùng tồn tại trong một account — cả hai đều list `Friendship-*` và Lambda vớ phải bảng nào DynamoDB trả trước.

Giải pháp là dùng escape hatch `addPropertyOverride` của CDK để tiêm tên bảng chính xác làm env var lúc deploy. Trong `backend/amplify/backend.ts`:

```typescript
import * as iam from 'aws-cdk-lib/aws-iam';
import * as cdk from 'aws-cdk-lib';
import { friendRequest } from './friend-request/resource';

const backend = defineBackend({ auth, data, friendRequest, /* ... */ });

const friendRequestLambda = backend.friendRequest.resources.lambda;

// IAM — cấp quyền hai bảng và các GSI
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

// Tiêm tên bảng chính xác — CDK tự resolve đúng tên cho từng môi trường
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

Vì sao chạy được:

- `backend.data.resources.tables['user'].tableName` trả về một CDK `Token` (tham chiếu lười). Lúc synth, CloudFormation thay bằng `Ref` tới resource `AWS::DynamoDB::Table` thực.
- Mỗi sandbox/branch có token riêng → tên bảng resolve riêng → không lẫn giữa các môi trường.
- Vì Lambda nằm trong `resourceGroupName: 'data'`, cả hai resource cùng nested stack, không tạo circular dependency.

Ghi chú về IAM: resource dùng wildcard (`table/user-*`) thay vì tham chiếu chính xác tới resource cùng stack, vì tham chiếu chính xác có thể gây circular dependency ở một số phiên bản Amplify.

## Điểm gọi từ frontend

Từ bất kỳ màn hình nào đã đăng nhập trên mobile:

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

## Các lỗi cần biết

| Lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `USER_TABLE_NAME env var not set` | Chưa tiêm env var — sandbox chưa deploy lại sau khi sửa `backend.ts`. | Chạy lại `npx ampx sandbox`. |
| `ConditionalCheckFailedException` | Hai client gửi cùng lời mời cùng lúc. | Retry backoff rồi hiển thị "đã pending" cho người dùng. |
| `TransactionCanceledException` | Một trong hai `Put` chạm throughput limit hoặc giới hạn kích thước item. | Xem CloudWatch lấy mã reason cho từng item. |
| `Unauthorized` khi accept | `sub` của caller không khớp `owner` trên hàng. | Frontend phải truyền `friendship_id` thuộc hàng mà caller sở hữu. |

## Xác minh

1. Đăng nhập user A ở một Expo client, user B ở client khác.
2. Từ A, gọi `sendRequest` với `friend_code` của B.
3. Trong console DynamoDB, query bảng `Friendship`. Phải thấy **hai hàng** có `linked_id` trỏ nhau, cả hai `status: pending`, một `direction: sent`, một `direction: received`.
4. Từ B, gọi `acceptRequest` với `id` của hàng received. Scan lại bảng — cả hai hàng giờ là `status: accepted`.
5. Kiểm tra CloudWatch log group `/aws/lambda/amplify-<app>-friend-request-<sandbox>` để xem trace.

![Friend request flow](/images/workshop/friend-request-flow.png)

## Tiếp theo

- [5.6.2 — Realtime Subscriptions](../5.6.2-Realtime-Subscriptions/)
