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

## Wallet Events (auto-pushed)

These events are pushed automatically by `ConnectHost` — **no `sphere_subscribe` needed**. Always handle them.

| Event | Constant | Payload | Description |
|-------|----------|---------|-------------|
| `wallet:locked` | `WALLET_EVENTS.LOCKED` | `{}` | Wallet logged out or session ended. Handling differs by transport — see below. |
| `identity:changed` | `WALLET_EVENTS.IDENTITY_CHANGED` | `PublicIdentity` | User switched address. dApp should update displayed identity. Also signals unlock after a `wallet:locked` in extension/iframe mode. |

### Popup vs Extension/Iframe: different LOCKED handling

The `wallet:locked` event means different things depending on the transport:

| Transport | What happened | Correct response |
|-----------|--------------|-----------------|
| **Popup** | Wallet was destroyed (logged out, navigated away). The popup window may still exist but the Sphere instance is gone. | Full disconnect: clear client, transport, and saved session. Do **not** programmatically close the popup. |
| **Extension / Iframe** | Wallet is locked but the host (service worker / parent frame) is still alive. | Set a `isWalletLocked` flag and wait. When the user unlocks, the wallet fires `identity:changed` which clears the flag. |

> **Host-side:** When the wallet's `Sphere` instance is destroyed (logout, lock), the host **must** call `notifyWalletLocked()` on the `ConnectHost` to push this event to all connected dApps. Forgetting this call leaves dApps in a stale connected state.

```typescript
import { WALLET_EVENTS } from '@unicitylabs/sphere-sdk/connect';

// Handle differently based on transport
client.on(WALLET_EVENTS.LOCKED, () => {
  if (transportType === 'popup') {
    // Popup: full disconnect — wallet instance is gone
    disconnect();
  } else {
    // Extension / iframe: wallet locked, host still alive — wait for unlock
    setIsWalletLocked(true);
  }
});

client.on(WALLET_EVENTS.IDENTITY_CHANGED, (data) => {
  // Update displayed address/nametag — also clears locked state
  const identity = data as PublicIdentity;
  setIsWalletLocked(false);
  console.log('Address changed to:', identity.nametag);
});
```

### Error-based disconnect (fallback)

If `wallet:locked` doesn't arrive (e.g., popup crashed), detect dead connections via request errors:

```typescript
try {
  await client.query('sphere_getBalance');
} catch (err) {
  if (/not.connected|timeout|transport|closed|session/i.test(err.message)) {
    // Transport is dead — disconnect and show Connect button
    disconnect();
  }
}
```

## Subscribable Events

Events below require `sphere_subscribe` or `client.on()`. The wallet proxies `Sphere.on()` events to connected dApps.

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

## Protocol Version & Compatibility

The Connect protocol is currently at **`2.0`** (`SPHERE_CONNECT_VERSION = '2.0'`).

**Same MAJOR = compatible.** A dApp on `2.0` and a wallet on `2.1` interoperate. **Different MAJOR = rejected.** A v1 dApp attempting to connect to a v2 wallet receives `UNSUPPORTED_PROTOCOL_VERSION` (4007) and must update its SDK.

The v1 → v2 cut is a one-time hard break. All dApps must update to SDK ≥ 0.9.x and declare `network` in `ConnectClientConfig`.

## Network Configuration

**Connect v2 requires every dApp to declare its target network.** The wallet rejects the handshake with `INCOMPATIBLE_NETWORK` (4008) if the network id does not match or if `network` is omitted.

### SPHERE_NETWORKS

Use `SPHERE_NETWORKS` (exported from `@unicitylabs/sphere-sdk/connect`) — single-sourced from `constants.NETWORKS` so the numeric id cannot drift from the embedded trust base.

```typescript
import { SPHERE_NETWORKS } from '@unicitylabs/sphere-sdk/connect';

// With ConnectClient:
const client = new ConnectClient({ /* ... */, network: SPHERE_NETWORKS.testnet2 });

// With autoConnect:
const result = await autoConnect({ /* ... */, network: SPHERE_NETWORKS.testnet2 });
```

`SPHERE_NETWORKS` currently exposes one entry: `testnet2 = { id: 4, name: 'testnet2' }`. Richer descriptor fields (`gatewayUrl`, `symbol`, `explorer`, `icon`) and runtime network switching are deferred. There is **no `switch_network` intent, no `network:changed` event, and no `switchNetwork()` method**.

### NetworkInfo type

```typescript
interface NetworkInfo {
  readonly id: number;    // canonical match key — RootTrustBase.networkId (testnet2 = 4)
  readonly name?: string; // human-readable metadata only
}
```

The wallet matches solely on `id`. Custom networks use the same shape: `network: { id, name }`.

### ConnectClientConfig.network

```typescript
import { ConnectClient, SPHERE_NETWORKS } from '@unicitylabs/sphere-sdk/connect';

const client = new ConnectClient({
  transport,
  dapp: { name: 'My App', url: location.origin },
  network: SPHERE_NETWORKS.testnet2, // REQUIRED — omitting this causes INCOMPATIBLE_NETWORK (4008)
  silent: false,
});
```

After a successful connect, the wallet's echoed network is available via `client.walletNetwork` (`NetworkInfo | null`).

### ConnectHostConfig.onConnectionRejected

Wallet hosts can wire this callback to surface rejection reasons in the wallet UI. It is notify-only — the gate decision is already made when this fires.

```typescript
const host = new ConnectHost({
  sphere, transport,
  onConnectionRequest: async (dapp, permissions, silent, clientInfo) => { /* ... */ },
  onIntent: async (action, params, session) => { /* ... */ },

  // Called when the compatibility gate rejects a connection.
  // Does NOT affect the decision. silent=true for auto-connect attempts.
  onConnectionRejected: (dapp, error, silent) => {
    if (!silent) showRejectionBanner(dapp?.name, error.message);
  },
});
```

Signature: `onConnectionRejected?(dapp: DAppMetadata | undefined, error: SphereRpcError, silent?: boolean): void`

## ConnectError

`client.connect()` rejects with a **`ConnectError`** when the compatibility gate refuses the connection.

```typescript
import { ConnectError, ERROR_CODES } from '@unicitylabs/sphere-sdk/connect';
```

`ConnectError` has:
- `.code: number` — numeric error code
- `.data?: unknown` — structured rejection details

**Important:** discriminate on the numeric `.code`, not `instanceof ConnectError`. The `instanceof` check is unreliable when multiple bundle copies of the SDK are present.

```typescript
try {
  await client.connect();
} catch (e) {
  const code = (e as { code?: number })?.code;
  if (code === ERROR_CODES.INCOMPATIBLE_NETWORK) {
    // data: { reason: 'network_incompatible', walletNetwork: { id: number }, clientNetwork: NetworkInfo | null }
    showWrongNetwork((e as ConnectError).data);
  } else if (code === ERROR_CODES.UNSUPPORTED_PROTOCOL_VERSION) {
    // data: { reason: 'protocol_incompatible', walletProtocol: '2.0', clientProtocol: '1.0' }
    showUpdateRequired((e as ConnectError).data);
  } else {
    showGenericError();
  }
}
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
| `4007` | `UNSUPPORTED_PROTOCOL_VERSION` | Connect MAJOR version mismatch — dApp must update its SDK |
| `4008` | `INCOMPATIBLE_NETWORK` | dApp targets a different network than the wallet, or omitted `network` |
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
import { SPHERE_CONNECT_VERSION, HOST_READY_TYPE, HOST_READY_TIMEOUT } from '@unicitylabs/sphere-sdk/connect';

SPHERE_CONNECT_VERSION // '2.0'
HOST_READY_TYPE        // 'sphere-connect:host-ready'
HOST_READY_TIMEOUT     // 30000 (ms)
```

## Import Paths

```typescript
// Recommended: autoConnect (handles transport detection + connection automatically)
import { autoConnect } from '@unicitylabs/sphere-sdk/connect/browser';
import type { AutoConnectResult, DetectedTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Detection utilities (also used internally by autoConnect)
import { isInIframe, hasExtension, detectTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Core protocol (ConnectClient, ConnectError, types, constants)
import { ConnectClient, ConnectError, SPHERE_NETWORKS, ERROR_CODES, RPC_METHODS, INTENT_ACTIONS, PERMISSION_SCOPES } from '@unicitylabs/sphere-sdk/connect';
import type { ConnectTransport, ConnectHostConfig, NetworkInfo, PublicIdentity, RpcMethod, IntentAction, PermissionScope, ConnectResult } from '@unicitylabs/sphere-sdk/connect';

// Browser transports (low-level — only if not using autoConnect)
import { PostMessageTransport, ExtensionTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Node.js transport
import { WebSocketTransport } from '@unicitylabs/sphere-sdk/connect/nodejs';
```
