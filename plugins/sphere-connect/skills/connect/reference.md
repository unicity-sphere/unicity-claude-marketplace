# Sphere Connect API Reference

Complete reference for the Connect protocol. Use this when generating code to ensure correct method names, parameters, permissions, and error handling.

## RPC Methods (Queries)

| Method | Permission Scope | Description |
|--------|-----------------|-------------|
| `sphere_getIdentity` | `identity:read` | Public identity (nametag, addresses) |
| `sphere_getBalance` | `balance:read` | L3 token balances |
| `sphere_getFiatBalance` | `balance:read` | USD fiat value |
| `sphere_getAssets` | `balance:read` | Assets grouped by coin |
| `sphere_getTokens` | `tokens:read` | Individual tokens |
| `sphere_getHistory` | `history:read` | L3 transaction history |
| `sphere_l1GetBalance` | `l1:read` | L1 (ALPHA) balance |
| `sphere_l1GetHistory` | `l1:read` | L1 transaction history |
| `sphere_resolve` | `resolve:peer` | Resolve nametag/address to PeerInfo |
| `sphere_subscribe` | `events:subscribe` | Subscribe to real-time events |
| `sphere_unsubscribe` | `events:subscribe` | Unsubscribe from events |
| `sphere_getConversations` | `dm:read` | Get DM conversation list |
| `sphere_getMessages` | `dm:read` | Get messages in a conversation |
| `sphere_getDMUnreadCount` | `dm:read` | Get unread DM count |
| `sphere_markAsRead` | `dm:read` | Mark messages as read |
| `sphere_disconnect` | *(none)* | Disconnect session |

### Query usage
```typescript
const balance = await client.query('sphere_getBalance');
const assets = await client.query('sphere_getAssets');
const identity = await client.query('sphere_getIdentity');
const history = await client.query('sphere_getHistory');
const peer = await client.query('sphere_resolve', { identifier: '@alice' });
```

## Intent Actions

| Action | Permission Scope | User Sees |
|--------|-----------------|-----------|
| `send` | `transfer:request` | Send token modal |
| `l1_send` | `l1:transfer` | L1 send modal |
| `dm` | `dm:request` | Direct message modal |
| `payment_request` | `payment:request` | Payment request modal |
| `receive` | `identity:read` | Receive address/QR |
| `sign_message` | `sign:request` | Sign message modal |

### Intent usage
```typescript
// Send L3 tokens
await client.intent('send', { recipient: '@alice', amount: '1000000', coinId: 'UCT' });

// Send L1 transaction
await client.intent('l1_send', { to: 'alpha1...', amount: '100000', feeRate: 5 });

// Send direct message
await client.intent('dm', { recipient: '@bob', message: 'Hello!' });

// Create payment request
await client.intent('payment_request', { recipient: '@bob', amount: '500000', coinId: 'UCT', message: 'For order #42' });

// Show receive address
await client.intent('receive', {});

// Sign a message (returns hex signature string)
const sig = await client.intent('sign_message', { message: 'I agree to the Terms of Service' });
```

> **Backend auth pattern:** Use `sign_message` to authenticate users to your server via challenge-response → JWT.
> See [backend-auth.md](backend-auth.md) for full implementation.

## Permission Scopes

| Scope | Grants access to |
|-------|-----------------|
| `identity:read` | Identity info, receive intent |
| `balance:read` | Balance, fiat balance, assets queries |
| `tokens:read` | Individual token list |
| `history:read` | Transaction history |
| `l1:read` | L1 balance & history |
| `events:subscribe` | Real-time event subscriptions |
| `resolve:peer` | Nametag/address resolution |
| `transfer:request` | Send L3 tokens intent |
| `l1:transfer` | Send L1 transaction intent |
| `dm:request` | Send DM intent |
| `dm:read` | Read conversations, messages, unread count |
| `payment:request` | Payment request intent |
| `sign:request` | Sign message intent |

**Default (always granted):** `identity:read`

## Events

Events can be subscribed via `client.on()`. The wallet proxies `Sphere.on()` events to connected dApps.

### Transfers
| Event | Payload | Description |
|-------|---------|-------------|
| `transfer:incoming` | `IncomingTransfer` | Tokens received |
| `transfer:confirmed` | `TransferResult` | Outgoing transfer confirmed |
| `transfer:failed` | `TransferResult` | Outgoing transfer failed |

### Payment Requests
| Event | Payload | Description |
|-------|---------|-------------|
| `payment_request:incoming` | `IncomingPaymentRequest` | Received payment request |
| `payment_request:accepted` | `IncomingPaymentRequest` | Payment request accepted |
| `payment_request:rejected` | `IncomingPaymentRequest` | Payment request rejected |
| `payment_request:paid` | `IncomingPaymentRequest` | Payment request paid |
| `payment_request:response` | `PaymentRequestResponse` | Response received |

### Messages
| Event | Payload | Description |
|-------|---------|-------------|
| `message:dm` | `DirectMessage` | Incoming DM |
| `message:read` | `{ messageIds, peerPubkey }` | Messages marked as read |
| `message:typing` | `{ senderPubkey, senderNametag?, timestamp }` | Typing indicator |
| `composing:started` | `ComposingIndicator` | Composing started |
| `message:broadcast` | `BroadcastMessage` | Broadcast message |

### Sync
| Event | Payload | Description |
|-------|---------|-------------|
| `sync:started` | `{ source }` | Sync started |
| `sync:completed` | `{ source, count }` | Sync completed |
| `sync:provider` | `{ providerId, success, added?, removed?, error? }` | Per-provider sync result |
| `sync:error` | `{ source, error }` | Sync error |
| `sync:remote-update` | `{ providerId, name, sequence, cid, added, removed }` | Remote IPFS update |

### Identity & Addresses
| Event | Payload | Description |
|-------|---------|-------------|
| `identity:changed` | `{ l1Address, directAddress, chainPubkey, nametag, addressIndex }` | Address switched |
| `nametag:registered` | `{ nametag, addressIndex }` | Nametag registered |
| `nametag:recovered` | `{ nametag }` | Nametag recovered from Nostr |
| `address:activated` | `{ address: TrackedAddress }` | New address tracked |
| `address:hidden` | `{ index, addressId }` | Address hidden |
| `address:unhidden` | `{ index, addressId }` | Address unhidden |

### Connection
| Event | Payload | Description |
|-------|---------|-------------|
| `connection:changed` | `{ provider, connected, status?, enabled?, error? }` | Provider connection state |

### Group Chat
| Event | Payload | Description |
|-------|---------|-------------|
| `groupchat:message` | `GroupMessageData` | Group chat message |
| `groupchat:joined` | `{ groupId, groupName }` | Joined group |
| `groupchat:left` | `{ groupId }` | Left group |
| `groupchat:kicked` | `{ groupId, groupName }` | Kicked from group |
| `groupchat:group_deleted` | `{ groupId, groupName }` | Group deleted |
| `groupchat:updated` | `{}` | Group list updated |
| `groupchat:connection` | `{ connected }` | Group relay connection |

### History
| Event | Payload | Description |
|-------|---------|-------------|
| `history:updated` | `TransactionHistoryEntry` | New history entry |

### Event subscription usage
```typescript
// Subscribe
const unsub = client.on('transfer:incoming', (data) => {
  console.log('Received tokens:', data);
});

// Unsubscribe
unsub();
```

## Error Codes

| Code | Name | Description |
|------|------|-------------|
| `-32700` | `PARSE_ERROR` | Invalid JSON |
| `-32600` | `INVALID_REQUEST` | Invalid request structure |
| `-32601` | `METHOD_NOT_FOUND` | Unknown RPC method |
| `-32602` | `INVALID_PARAMS` | Invalid parameters |
| `-32603` | `INTERNAL_ERROR` | Internal wallet error |
| `4001` | `NOT_CONNECTED` | Session not established |
| `4002` | `PERMISSION_DENIED` | Missing required permission |
| `4003` | `USER_REJECTED` | User rejected in wallet UI |
| `4004` | `SESSION_EXPIRED` | Session TTL expired |
| `4005` | `ORIGIN_BLOCKED` | Origin blocked by wallet |
| `4006` | `RATE_LIMITED` | Too many requests |
| `4100` | `INSUFFICIENT_BALANCE` | Not enough tokens |
| `4101` | `INVALID_RECIPIENT` | Bad recipient address |
| `4102` | `TRANSFER_FAILED` | Transfer execution failed |
| `4200` | `INTENT_CANCELLED` | Intent cancelled |

### Error handling
```typescript
try {
  await client.intent('send', { recipient: '@alice', amount: '1000000', coinId: 'UCT' });
} catch (err) {
  if (err.code === 4003) console.log('User rejected the transaction');
  else if (err.code === 4100) console.log('Insufficient balance');
  else if (err.code === 4002) console.log('Missing permission — request transfer:request scope');
  else throw err;
}
```

## Protocol Constants

```typescript
import { HOST_READY_TYPE, HOST_READY_TIMEOUT } from '@unicitylabs/sphere-sdk/connect';

HOST_READY_TYPE    // 'sphere-connect:host-ready'
HOST_READY_TIMEOUT // 30000 (ms)
```

## Import Paths

```typescript
// Recommended: autoConnect (handles transport detection + connection automatically)
import { autoConnect } from '@unicitylabs/sphere-sdk/connect/browser';
import type { AutoConnectResult, DetectedTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Detection utilities (also used internally by autoConnect)
import { isInIframe, hasExtension, detectTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Core protocol (ConnectClient, types, constants)
import { ConnectClient, RPC_METHODS, INTENT_ACTIONS, PERMISSION_SCOPES, ERROR_CODES } from '@unicitylabs/sphere-sdk/connect';
import type { ConnectTransport, PublicIdentity, RpcMethod, IntentAction, PermissionScope, ConnectResult } from '@unicitylabs/sphere-sdk/connect';

// Browser transports (low-level — only if not using autoConnect)
import { PostMessageTransport, ExtensionTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Node.js transport
import { WebSocketTransport } from '@unicitylabs/sphere-sdk/connect/nodejs';
```
