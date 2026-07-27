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
| `sphere_resolve` | `resolve:peer` | Resolve nametag/address to PeerInfo |
| `sphere_subscribe` | `events:subscribe` | Subscribe to real-time events |
| `sphere_unsubscribe` | `events:subscribe` | Unsubscribe from events |
| `sphere_getConversations` | `dm:read` | Get DM conversation list |
| `sphere_getMessages` | `dm:read` | Get messages in a conversation |
| `sphere_getDMUnreadCount` | `dm:read` | Get unread DM count |
| `sphere_markAsRead` | `dm:manage` | Mark messages as read |
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
| `dm` | `dm:request` | Direct message modal |
| `payment_request` | `payment:request` | Payment request modal |
| `receive` | `identity:read` | Receive address/QR |
| `sign_message` | `sign:request` | Sign message modal |
| `mint` | `mint:request` | Mint confirmation modal |

### Intent usage
```typescript
// Send L3 tokens. amount is in BASE UNITS (smallest unit), a string — convert a
// human amount with parseTokenAmount(human, decimals) (or ethers/viem parseUnits).
await client.intent('send', { to: '@alice', amount: '1000000000000000000', coinId: '<lowercase 64-hex coin id>' }); // = 1 of an 18-decimals coin

// Send direct message
await client.intent('dm', { to: '@bob', message: 'Hello!' });

// Create payment request (amount in base units, like send)
await client.intent('payment_request', { to: '@bob', amount: '5000000000000000000', coinId: '<lowercase 64-hex coin id>', message: 'For order #42' });

// Show receive address
await client.intent('receive', {});

// Sign a message (returns hex signature string)
const sig = await client.intent('sign_message', { message: 'I agree to the Terms of Service' });

// Self-mint a fungible token to the connected wallet (coinId is lowercase hex; works where the network allows self-mint, e.g. testnet2)
const { tokenId } = await client.intent('mint', { coinId: '1111111111111111111111111111111111111111111111111111111111111111', amount: '1000000' });
```

> **Backend auth pattern:** Use `sign_message` to authenticate users to your server via challenge-response → JWT.
> See [backend-auth.md](backend-auth.md) for full implementation.

> **Note:** invoice/accounting intents (`create_invoice`, `pay_invoice`, …) and the invoice
> queries (`sphere_getInvoices`, `sphere_getInvoiceStatus`) exist in the Connect protocol but are
> **experimental and not supported by the Sphere wallet** — do not generate calls to them.
> `coinId` is the canonical lowercase 64-hex id (a symbol like `UCT` is rejected).

> **Amount units:** `send` / `payment_request` / `mint` all take `amount` in **base units**
> (the smallest indivisible unit), as a string — never whole tokens. Convert a user's human
> amount at your app's edge with `parseTokenAmount(human, decimals)` (or ethers/viem `parseUnits`).

## Permission Scopes

| Scope | Grants access to |
|-------|-----------------|
| `identity:read` | Identity info, receive intent |
| `balance:read` | Balance, fiat balance, assets queries |
| `tokens:read` | Individual token list |
| `history:read` | Transaction history |
| `events:subscribe` | Real-time event subscriptions |
| `resolve:peer` | Nametag/address resolution |
| `transfer:request` | Send L3 tokens intent |
| `dm:request` | Send DM intent |
| `dm:read` | Read conversations, messages, unread count |
| `dm:manage` | Mark DM messages as read |
| `payment:request` | Payment request intent |
| `sign:request` | Sign message intent |
| `mint:request` | Mint (self-mint a fungible token) intent |

**Default (always granted):** `identity:read`

## Wallet Events (auto-pushed)

These events are pushed automatically by `ConnectHost` — **no `sphere_subscribe` needed**. Always handle them. Routing one through `sphere_subscribe` is refused, because `Sphere.on()` would accept the name and silently never emit.

| Event | Constant | Payload | Description |
|-------|----------|---------|-------------|
| `wallet:locked` | `WALLET_EVENTS.LOCKED` | `{}` | Wallet locked. **The session is still alive** — requests answer `WALLET_LOCKED` (4009) until unlock. Do not disconnect. |
| `wallet:unlocked` | `WALLET_EVENTS.UNLOCKED` | `{ identity?: PublicIdentity }` | Same session resumed; subscriptions already re-armed by the host. Compare `identity.chainPubkey` before resuming anything. |
| `wallet:disconnected` | `WALLET_EVENTS.DISCONNECTED` | `{}` | Session destroyed (logout, wallet deleted, expiry, a different seed behind the lock screen). Clear state and re-handshake. |
| `identity:changed` | `WALLET_EVENTS.IDENTITY_CHANGED` | `PublicIdentity` | User switched address. Update the displayed identity. |

### The lock is a state, not a teardown

Handling does **not** depend on the transport any more. In popup, extension and iframe mode alike
a lock preserves the session; only `wallet:disconnected` ends it.

| Transport | On `wallet:locked` |
|-----------|--------------------|
| **Popup** | Keep the client, the transport and the saved `sessionId`. Do **not** close the popup — the user unlocks in it. |
| **Extension / Iframe** | Identical: set the flag and wait for `wallet:unlocked`. |

Served while locked: `sphere_getIdentity` (from the wallet's frozen snapshot), `sphere_subscribe`,
`sphere_unsubscribe`, `sphere_disconnect`. Everything else, and every intent, is answered 4009 in
the same tick — the host never parks a request and never waits for a human. Balances, tokens and
history are never served and never cached — and neither is anything else: the twelve refused
methods include `sphere_resolve` and **all four DM reads**, so messaging does NOT keep working
while locked. Stop issuing reads and wait for the unlock; do not poll into refusals.

A wallet that **cold-starts locked** (a wallet-page reload or a fresh popup — the password is
memory-only) holds no session, so the HANDSHAKE itself is refused with an errorless empty
response: `connect()` rejects with no `.code` at all. Do not write
`if (err.code === ERROR_CODES.WALLET_LOCKED)` expecting to catch that case — treat an unexpected
rejection as "not ready yet" and retry on the next `HOST_READY`.

A resume handshake whose `sessionId` matches **succeeds** while locked and the response carries
`locked: true` (`ConnectResult.locked`, and `client.walletLocked`). Any handshake while locked is
forced silent, so an origin without an approval still gets the usual empty refusal and learns
nothing about the lock.

> **Talking to a 2.0 wallet:** the protocol is `2.1`, and the gate compares MAJOR only, so a `2.0`
> wallet still connects — but there `wallet:locked` also revoked the session and `wallet:unlocked`
> never arrives. Read `client.walletProtocol` (the version the wallet reported at handshake) and
> fall back to a full teardown when its MINOR is below 1. Treat an unknown version as legacy.

> **Host-side:** a wallet calls `setLocked()` for a lock (session preserved), `updateSphere()` for
> an unlock, `revokeSession()` for a logout, `setUnavailable()` when Sphere is gone for a non-lock
> reason. `notifyWalletLocked()` has been **removed**, not aliased — its old meaning was the
> opposite of its new one. The unlock UI is raised by the wallet from its own chrome after a human
> click; no dApp request may raise the password field.

```typescript
import { WALLET_EVENTS } from '@unicitylabs/sphere-sdk/connect';

// Same handling in every transport mode.
client.on(WALLET_EVENTS.LOCKED, () => {
  setIsWalletLocked(true);            // keep client, transport and sessionId
});

client.on(WALLET_EVENTS.UNLOCKED, (data) => {
  const next = (data as { identity?: PublicIdentity }).identity ?? null;
  if (!next || next.chainPubkey !== connectedIdentity.chainPubkey) {
    setWalletChanged(true);           // a DIFFERENT seed came back — resume nothing
    setIsWalletLocked(false);
    return;
  }
  setIsWalletLocked(false);
  refetchReads();                     // never an intent — see below
  // Nothing to re-subscribe: the host replayed your subscription keys before pushing this.
});

client.on(WALLET_EVENTS.DISCONNECTED, () => {
  clearSessionAndState();             // the only real teardown of the four
});

client.on(WALLET_EVENTS.IDENTITY_CHANGED, (data) => {
  setIdentity(data as PublicIdentity);
  setIsWalletLocked(false);
});
```

**Never auto-replay an intent after an unlock.** It moves money, and it would execute with no
fresh user gesture at the exact moment the wallet came back. Reads are safe; intents are not.

### Classify failures by code, not by message text

```typescript
try {
  await client.query('sphere_getBalance');
} catch (err) {
  const code = typeof err === 'object' && err !== null && 'code' in err ? (err as { code: unknown }).code : undefined;
  if (code === ERROR_CODES.WALLET_LOCKED)        setIsWalletLocked(true);  // stay connected
  else if (code === ERROR_CODES.NOT_CONNECTED ||
           code === ERROR_CODES.SESSION_EXPIRED) clearSessionAndState();   // really gone
  else                                           surface(err);             // everything else
}
```

A 4009 carries `data: { reason: 'locked' }` if you want the detail. The refusal **text** is a
documented recommendation, not a contract — never match on it. A few SDK failures carry no code at
all (`Not connected`, `Query timeout: …`, `Intent timeout: …`, `Connection timeout`,
`Disconnected`), so keep a narrow message fallback for exactly those. The old advice,
`/not.connected|timeout|transport|closed|session/i`, disconnected on any error whose text merely
mentioned a session.

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
| `identity:changed` | `{ directAddress, chainPubkey, nametag, addressIndex }` | Address switched |
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

The Connect protocol is currently at **`2.1`** (`SPHERE_CONNECT_VERSION = '2.1'`).

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
| `4009` | `WALLET_LOCKED` | Wallet is locked — **the session is still alive**; `data.reason === 'locked'`; retry after `wallet:unlocked` |
| `4100` | `INSUFFICIENT_BALANCE` | Not enough tokens |
| `4101` | `INVALID_RECIPIENT` | Bad recipient address |
| `4102` | `TRANSFER_FAILED` | Transfer execution failed |
| `4200` | `INTENT_CANCELLED` | Intent cancelled — the user declined and **nothing happened**. Safe to re-offer. |
| `4201` | `INTENT_OUTCOME_UNKNOWN` | The wallet took the intent and the answer was lost (host deadline, lock, logout). **The outcome is unknown — the money may or may not have moved. Never retry**; reconcile out of band first. |

### Error handling
```typescript
try {
  await client.intent('send', { to: '@alice', amount: '1000000000000000000', coinId: '<lowercase 64-hex coin id>' }); // base units
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

SPHERE_CONNECT_VERSION // '2.1'
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
