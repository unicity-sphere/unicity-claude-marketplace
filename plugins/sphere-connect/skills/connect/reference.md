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

| Action | Permission Scope | Since | User Sees |
|--------|-----------------|-------|-----------|
| `send` | `transfer:request` | 2.0 | Send token modal |
| `dm` | `dm:request` | 2.0 | Direct message modal |
| `payment_request` | `payment:request` | 2.0 | Payment request modal |
| `receive` | `identity:read` | 2.0 | Receive address/QR |
| `sign_message` | `sign:request` | 2.0 | Sign message modal |
| `mint` | `mint:request` | 2.0 | Mint confirmation modal |
| `send_nft` | `nft:transfer` | 2.2 | **SDK-only** — the Sphere wallet answers `-32601` |
| `mint_nft` | `nft:mint` | 2.3 | NFT mint confirmation modal — **always** prompted |

### Intent usage
```typescript
// Send L3 tokens. amount is in BASE UNITS (smallest unit), a string — convert a
// human amount with parseTokenAmount(human, decimals) (or ethers/viem parseUnits).
await client.intent('send', { to: '@alice', amount: '1000000000000000000', coinId: '<lowercase 64-hex coin id>' }); // = 1 of an 18-decimals coin
```

**What `send` resolves to** — this matters more than the request shape:

```typescript
{
  success: true,
  transferId?: string,  // OMITTED when the send is delivery-pending
  status: 'pending' | 'submitted' | 'confirmed' | 'delivered' | 'completed' | 'failed',
  deliveryPending: boolean,
}
```

`deliveryPending: true` means the spend is committed on-chain — or, on possibly-certified resolutions, may already
be — but delivery to the recipient has not landed; the wallet journaled it and will retry under the original transfer. It arrives as
`{ success: true, status: 'pending', deliveryPending: true }` — a success **with no `transferId`**, because
pending results carry an empty id by design.

**Never re-send on `deliveryPending`.** A second `send` consumes a different source token and pays twice.
Treat it as a pending success and tell the user the money may already have moved (at minimum it is in flight and must not be re-sent).

```typescript
const result = await client.intent('send', { to, amount, coinId });
if (result.deliveryPending) {
  // Money is sent (or may already be). Delivery is queued. Do NOT retry.
  showPendingDelivery(result.status);
} else if (result.transferId) {
  showDelivered(result.transferId);
}
```

```typescript
// Send direct message
await client.intent('dm', { to: '@bob', message: 'Hello!' });

// Create payment request (amount in base units, like send)
await client.intent('payment_request', { to: '@bob', amount: '5000000000000000000', coinId: '<lowercase 64-hex coin id>', message: 'For order #42' });

// Show receive address
await client.intent('receive', {});

// Sign a message. Resolves to an OBJECT, not a bare string.
const { signature, publicKey } = await client.intent<{ signature: string; publicKey?: string }>(
  'sign_message',
  { message: 'I agree to the Terms of Service' },
);

// Self-mint a fungible token to the connected wallet (coinId is lowercase hex; works where the network allows self-mint, e.g. testnet2)
const { tokenId } = await client.intent('mint', { coinId: '1111111111111111111111111111111111111111111111111111111111111111', amount: '1000000' });
```

> **`sign_message` returns `{ signature, publicKey }`.** `signature` is the hex signature.
> **`publicKey` is self-reported by the wallet — never trust it and never forward it as the
> signer's identity.** The only sound way to learn who signed is to recover the public key from the
> signature over the exact challenge bytes, server-side. A backend that keys a session on the
> `publicKey` the client sent authenticates whoever typed it.

> **Backend auth pattern:** Use `sign_message` to authenticate users to your server via challenge-response → JWT.
> See [backend-auth.md](backend-auth.md) for full implementation.

> **There is no invoice surface.** `create_invoice`, `pay_invoice`, `sphere_getInvoices` and
> `sphere_getInvoiceStatus` are not in `INTENT_ACTIONS` / `RPC_METHODS` and never reach a handler:
> an unmapped name has no permission mapping, so it is refused with **`PERMISSION_DENIED` (4002)**,
> which reads misleadingly like a missing scope. Do not generate calls to them.
> `coinId` is the canonical lowercase 64-hex id (a symbol like `UCT` is rejected).

> **Amount units:** `send` / `payment_request` / `mint` all take `amount` in **base units**
> (the smallest indivisible unit), as a string — never whole tokens. Convert a user's human
> amount at your app's edge with `parseTokenAmount(human, decimals)` (or ethers/viem `parseUnits`).

### `mint_nft` (Connect 2.3)

Mints an NFT into the connected wallet from content the **dApp** chooses, signed as the user.

```typescript
import { nftContentToWire } from '@unicitylabs/sphere-sdk/connect';
import type { MintNftIntentParams, MintNftIntentResult } from '@unicitylabs/sphere-sdk/connect';

// NftContent is one of: a 'metadata' record, an inline 'media' file, or a 'link'.
// nftContentToWire() base64-encodes every inline Uint8Array, because Connect frames are JSON.
const params: MintNftIntentParams = {
  content: nftContentToWire({
    kind: 'metadata',
    name: 'Badge #1',
    description: 'Proof of attendance',
    image: { kind: 'media', media_type: 'image/png', bytes: pngBytes }, // Uint8Array
    animation_url: null,
    external_url: null,
    attributes: [{ trait_type: 'level', value: 3 }],
    collection: null,
    collection_id: null,
  }),
  sign: true, // default — the wallet wraps the payload with its chain key as creator
};

const { tokenId } = await client.intent<MintNftIntentResult>('mint_nft', params);
```

- **Scope: `nft:mint`.** Nothing else implies it — neither `mint:request` nor `nft:transfer`.
- **Always prompted.** The host keeps `mint_nft` on an always-ask list, so no auto-approval or
  pre-granted permission can mint without a fresh human confirmation. Minting signs dApp-chosen
  bytes as the user; that is why it is never silent.
- **Payload cap ~1 MiB** (`NFT_MAX_PAYLOAD_BYTES = 1048576`), measured on the **encoded** payload,
  and refused before anything is journaled or minted. Link large files with a `link` content item
  (`{ kind: 'link', media_type, uri, sha256 }`) instead of inlining them.
- **A Connect 2.2 wallet answers `PERMISSION_DENIED` (4002)**, not `-32601`: it has no `nft:mint`
  scope to map the action to, so the permission check refuses it first. Treat a 4002 on `mint_nft`
  as "this wallet is too old", not as a scope the user can grant.
- **On `INTENT_OUTCOME_UNKNOWN` (4201), never re-issue the mint** — see the error-code section.

### `send_nft` (Connect 2.2) — SDK-only

`send_nft` moves a coinless token (an NFT) by `tokenId`: `{ to, tokenId, memo? }`, scope
`nft:transfer`. It is defined in the protocol and routed by the SDK, but **the Sphere wallet does
not implement it** — it answers `METHOD_NOT_FOUND` (`-32601`), *"Intent ... is not supported by this
wallet"*. Do not build a UI around it against the Sphere wallet; use it only with a wallet whose
own docs say it handles the action.

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
| `nft:transfer` | `send_nft` intent — moving a coinless token. Distinct from `transfer:request` (2.2) |
| `nft:mint` | `mint_nft` intent — signing dApp-chosen content as the user. Implied by nothing else (2.3) |

Fifteen scopes in total (`PERMISSION_SCOPES`). Request only the ones the dApp actually uses: every
extra scope is something the user is asked to grant on the connect screen.

**Default (always granted):** `identity:read`

## Wallet Events (auto-pushed)

These events are pushed automatically by `ConnectHost` — **no `sphere_subscribe` needed**. Always
handle them.

A `sphere_subscribe` for one of these four is **answered with success and attaches nothing**
(`{ subscribed: true, event }`). That is not a refusal and not a bug: the host pushes them
unconditionally, so "you are subscribed" is true — it is simply satisfied by a different mechanism.
It must not be attached to `Sphere.on()`, which accepts any string and would then never emit for
these names. A 2.1+ client skips the call entirely; the success answer exists so that older dApps,
which do send it, keep working.

| Event | Constant | Payload | Description |
|-------|----------|---------|-------------|
| `wallet:locked` | `WALLET_EVENTS.LOCKED` | `{}` | Wallet locked. **The session is still alive** — requests answer `WALLET_LOCKED` (4009) until unlock. Do not disconnect. |
| `wallet:unlocked` | `WALLET_EVENTS.UNLOCKED` | `{ identity?: PublicIdentity }` | Same session resumed; subscriptions already re-armed by the host. Compare `identity.chainPubkey` before resuming anything. |
| `wallet:disconnected` | `WALLET_EVENTS.DISCONNECTED` | `{}` | Session destroyed (logout, wallet deleted, expiry, a different seed behind the lock screen). Clear state and re-handshake. |
| `identity:changed` | `WALLET_EVENTS.IDENTITY_CHANGED` | `PublicIdentity` | User switched address. Update the displayed identity. |

### The lock is a state, not a teardown

Handling does **not** depend on the transport any more. In every transport mode a lock preserves
the session; only `wallet:disconnected` ends it.

| Transport | On `wallet:locked` |
|-----------|--------------------|
| **Popup** | Keep the client, the transport and the saved `sessionId`. Do **not** close the popup — the user unlocks in it. |
| **Iframe** (the production transport) | Identical: set the flag and wait for `wallet:unlocked`. |

Served while locked: `sphere_getIdentity` (from the wallet's frozen snapshot), `sphere_subscribe`,
`sphere_unsubscribe`, `sphere_disconnect`. Everything else, and every intent, is answered 4009 in
the same tick — the host never parks a request and never waits for a human. Balances, tokens and
history are never served and never cached — and neither is anything else: `RPC_METHODS` has
fourteen entries, four are on the allow-list above, and the other **ten** — including
`sphere_resolve` and **all four DM reads** — are refused, so messaging does NOT keep working
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

> **Talking to a 2.0 wallet:** the protocol is `2.3`, and the gate compares MAJOR only, so a `2.0`
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
documented recommendation, not a contract — never match on it.

A few SDK failures still carry no code at all, so keep a narrow message fallback for exactly those.
This is the complete set in 0.17.2 — every plain `Error` the client-side Connect surface raises,
three in `connect/client/ConnectClient.ts` and four in `impl/browser/connect/autoConnect.ts`:

| Codeless failure | Raised by | Means |
|------------------|-----------|-------|
| `Connection timeout` | `ConnectClient.connect()` | no handshake response within `timeout` |
| `Query timeout: <method>` | `ConnectClient.query()` | no response within `timeout` |
| `Connection rejected by wallet` | `ConnectClient.connect()` | the wallet answered the handshake unapproved and sent no error |
| `autoConnect: walletUrl is required when no extension or iframe is available` | `autoConnect()` popup path | config error — standalone page and no `walletUrl`; nothing was opened, so there is no connection to tear down |
| `autoConnect: Failed to open wallet popup — check popup blocker settings` | `autoConnect()` popup path | `window.open()` returned null — a popup blocker; ask for a user gesture and retry |
| `autoConnect: Wallet popup did not respond in time` | `autoConnect()` popup path | no `HOST_READY` from the popup |
| `autoConnect: Wallet popup was closed before connecting` | `autoConnect()` popup path | the user closed the popup |

**`Not connected`, `Disconnected` and the intent timeout are *not* on that list** — they are typed
today: the first two are `ConnectError` with `NOT_CONNECTED` (4001), and a timed-out or
mid-flight-dropped **intent** rejects with `INTENT_OUTCOME_UNKNOWN` (4201), never with a plain
`Error`. Keep matching them by text and a 4201 lands in the "connection is gone, reset" branch — the
one place a retry must never be offered.

The old advice, `/not.connected|timeout|transport|closed|session/i`, disconnected on any error whose
text merely mentioned a session.

The two `walletUrl` / popup-blocker rows fail **before** a transport exists, so they belong in a
"could not start" branch, not in the codeless-teardown regex a request path uses.

On Node, `WebSocketTransport.connect()` additionally rejects with whatever the `ws` implementation
threw at construction or emitted as `onerror`. That error is not authored by the SDK and its text is
not a contract: match nothing there, and treat any non-`ConnectError` from a Node connect as a
transport failure.

One more failure on this surface is typed but **not** a `ConnectError`: `nftContentToWire()`
validates before anything is sent and throws a `SphereError` whose `code` is the string
`'VALIDATION_ERROR'`, not a numeric Connect code. Catch it where you build the `mint_nft` payload —
that call never reaches the wallet.

## Subscribable Events

Events below require `sphere_subscribe` or `client.on()`. The host attaches the name to
`Sphere.on()` — or, for a legacy name, to a compatibility adapter fed from the v2 event it replaced.

> **An unknown name subscribes successfully and then never fires.** `Sphere.on()` accepts any string,
> so a typo, or a name that used to exist, is answered `{ subscribed: true }` and delivers nothing
> forever. Copy the names from the tables below — the source of truth is `SphereEventType` in the SDK.
>
> The lists below dropped `payment_request:accepted`, `payment_request:response`, `sync:started`,
> `sync:provider` and `sync:error`: those names have no emitter on either side and were exactly that
> silent-forever case.

### Current events (`SphereEventType`)

#### Transfers & inventory
| Event | Payload | Description |
|-------|---------|-------------|
| `transfer:incoming` | `IncomingTransfer` | Tokens received |
| `transfer:updated` | `TransferResult` | A transfer advanced (send/receive/mint) — carries the full result |
| `transfer:attention` | `{ transferId, code, detail? }` | A transfer needs attention (undeliverable, checkpoint-stuck, …) |
| `inventory:updated` | `{}` | Held tokens changed — re-read `tokens()` / `assets()` |

#### Payment requests
| Event | Payload | Description |
|-------|---------|-------------|
| `payment_request:incoming` | legacy `IncomingPaymentRequest` shape | Received payment request. The host maps it through the compat adapter, so `symbol` is always present (`''` when unknown) |
| `payment_request:updated` | `{ id, status: 'pending' \| 'settling' \| 'paid' \| 'rejected' \| 'expired' }` | One request changed state |

#### Messages
| Event | Payload | Description |
|-------|---------|-------------|
| `message:dm` | `DirectMessage` | Incoming DM |
| `message:read` | `{ messageIds, peerPubkey }` | Messages marked as read |
| `message:typing` | `{ senderPubkey, senderNametag?, timestamp }` | Typing indicator |
| `composing:started` | `ComposingIndicator` | Composing started |
| `message:broadcast` | `BroadcastMessage` | Broadcast message |
| `communications:ready` | `{ conversationCount }` | Messaging finished loading |

#### Identity & addresses
| Event | Payload | Description |
|-------|---------|-------------|
| `identity:changed` | `{ directAddress?, chainPubkey, nametag?, addressIndex }` | Address switched. **Also auto-pushed** — see the auto-pushed table; you do not need to subscribe |
| `nametag:registered` | `{ nametag, addressIndex }` | Nametag registered |
| `nametag:recovered` | `{ nametag }` | Nametag recovered from Nostr |
| `address:activated` | `{ address: TrackedAddress }` | New address tracked |
| `address:hidden` | `{ index, addressId }` | Address hidden |
| `address:unhidden` | `{ index, addressId }` | Address unhidden |

#### Connection
| Event | Payload | Description |
|-------|---------|-------------|
| `connection:status` | `{ status: 'connected' \| 'degraded' \| 'offline' }` | Wallet backend session connectivity |
| `connection:changed` | `{ provider, connected, status?, enabled?, error? }` | Per-provider connection state |

#### Group chat
| Event | Payload | Description |
|-------|---------|-------------|
| `groupchat:message` | `GroupMessageData` | Group chat message |
| `groupchat:joined` | `{ groupId, groupName }` | Joined group |
| `groupchat:left` | `{ groupId }` | Left group |
| `groupchat:kicked` | `{ groupId, groupName }` | Kicked from group |
| `groupchat:group_deleted` | `{ groupId, groupName }` | Group deleted |
| `groupchat:updated` | `{}` | Group list updated |
| `groupchat:connection` | `{ connected }` | Group relay connection |
| `groupchat:ready` | `{ groupCount }` | Group chat finished loading |

#### History
| Event | Payload | Description |
|-------|---------|-------------|
| `history:updated` | `HistoryEntry` | New history entry |

### Legacy names still served (compatibility adapters)

These names have no emitter of their own any more. The host re-emits them from the current events
above, so a dApp written against the older API keeps working unchanged. **New code should subscribe
to the current name** and branch on the payload itself.

| Legacy event | Payload | Re-emitted from |
|--------------|---------|-----------------|
| `transfer:confirmed` | `TransferResult` | `transfer:updated` where the status settled and `deliveryPending !== true` |
| `transfer:delivery_pending` | `TransferResult` | `transfer:updated` with `deliveryPending === true` and not failed |
| `transfer:failed` | `TransferResult` | `transfer:updated` with `status === 'failed'` |
| `payment_request:paid` | legacy `IncomingPaymentRequest` | `payment_request:updated` with that status |
| `payment_request:rejected` | legacy `IncomingPaymentRequest` | `payment_request:updated` with that status |
| `payment_request:expired` | legacy `IncomingPaymentRequest` | `payment_request:updated` with that status |
| `split:checkpoint-stuck` | `{ transferId, code, error }` | `transfer:attention` with that code |
| `delivery:undeliverable` | `{ transferId, recipientPubkey, attempts, error }` | `transfer:attention` with that code |
| `delivery:deferred` | `{ transferId, recipientPubkey, reason, deferredUntil }` | `transfer:attention` with that code |
| `realtime:status` | `{ status: 'connected' \| 'reconnecting' \| 'closed' }` | `connection:status` |
| `storage:degraded` | `{ providerId, error }` | `connection:status` when degraded |
| `sync:completed` | `{ source, count }` | `inventory:updated` |
| `sync:remote-update` | `{ providerId, name, sequence, cid, added, removed }` | `inventory:updated` |

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

The Connect protocol is currently at **`2.3`** (`SPHERE_CONNECT_VERSION = '2.3'`).

**Same MAJOR = compatible.** A dApp on `2.0` and a wallet on `2.3` interoperate. **Different MAJOR = rejected.** A v1 dApp attempting to connect to a v2 wallet receives `UNSUPPORTED_PROTOCOL_VERSION` (4007) and must update its SDK.

What the MINOR versions added — all additive, so an older dApp keeps working and only misses the new
surface:

| Version | Adds |
|---------|------|
| `2.1` | The lock is a **state**, not a teardown: `wallet:locked` preserves the session, `wallet:unlocked` resumes it, `WALLET_LOCKED` (4009) answers requests meanwhile, and a resume handshake succeeds with `locked: true` |
| `2.2` | The `send_nft` intent and the `nft:transfer` scope (SDK-only — the Sphere wallet answers `-32601`) |
| `2.3` | The `mint_nft` intent, the `nft:mint` scope, and the NFT wire codec (`nftContentToWire`) |

Feature-detect by version rather than by trial: read `client.walletProtocol` (what the wallet
reported at handshake) and hide a `mint_nft` action when its MINOR is below 3. A 2.2 wallet answers
`mint_nft` with `PERMISSION_DENIED` (4002), which is indistinguishable from a scope the user could
have granted.

### Version floor

The wallet also enforces an **npm SDK floor** on the dApp: `DEFAULT_MIN_CLIENT_SDK_VERSION =
'0.14.1-0'`. A dApp reporting an `@unicitylabs/sphere-sdk` below **0.14.1** is refused with
`UNSUPPORTED_PROTOCOL_VERSION` (4007) before any UI appears. A wallet may raise the floor via
`ConnectHostConfig.minSdkVersion`, so treat 0.14.1 as the floor, not as a target.

**Use `0.17.x`.** Everything documented here — `mint_nft`, `SPHERE_NETWORKS.mainnet`, protocol 2.3 —
needs it, and the floor fields in a refusal (`requiredSdk` / `actualSdk`) tell the user what to move
to when they are on something older.

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

`SPHERE_NETWORKS` exposes **two live entries**:

| Key | Value | Notes |
|-----|-------|-------|
| `SPHERE_NETWORKS.mainnet` | `{ id: 1, name: 'mainnet' }` | Live. Real value moves here |
| `SPHERE_NETWORKS.testnet2` | `{ id: 4, name: 'testnet2' }` | Live |

**Make the target network a config value, not a literal.** Both networks are live and a wallet is on
one of them; a dApp that hardcodes the wrong one is refused at the handshake with
`INCOMPATIBLE_NETWORK` (4008) and shows the user nothing but a failure. Read it from the environment
and map it once:

```typescript
// Vite: VITE_SPHERE_NETWORK=testnet2 | mainnet   (Next.js: NEXT_PUBLIC_SPHERE_NETWORK)
const name = import.meta.env.VITE_SPHERE_NETWORK ?? 'testnet2';
const network = SPHERE_NETWORKS[name as keyof typeof SPHERE_NETWORKS];
if (!network) throw new Error(`Unknown Sphere network: ${name}`);
```

Then say which network you targeted when a 4008 comes back — the refusal carries
`data.walletNetwork.id` and `data.clientNetwork`, so both sides can be named.

Richer descriptor fields (`gatewayUrl`, `symbol`, `explorer`, `icon`) are not part of this constant.
There is **no `switch_network` intent, no `network:changed` event, and no `switchNetwork()` method**:
a dApp targets one network per session and asks the user to switch the wallet instead.

### NetworkInfo type

```typescript
interface NetworkInfo {
  readonly id: number;    // canonical match key — RootTrustBase.networkId (mainnet = 1, testnet2 = 4)
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

**Important: discriminate on the numeric `.code`, never on `instanceof ConnectError`.**

The reason is not "some apps bundle the SDK twice" — it is the SDK's own packaging. Until the SDK
release that ships the sphere-sdk#790 fix, `./connect/browser` ships **its own copy** of the Connect core, so there are two
`ConnectError` classes in one install. An error thrown by `autoConnect()` is an instance of the copy
inside `./connect/browser`, and `err instanceof ConnectError` — with `ConnectError` imported from
`./connect` — is therefore **`false`**. Nothing about the app's own bundling changes that.

The same split is a type error too: `const c: ConnectClient = (await autoConnect(…)).client` fails
with **TS2322**, *"Types have separate declarations of a private property 'transport'"*. Type the
client as `AutoConnectResult['client']` rather than casting.

Both are tracked in
[sphere-sdk#789](https://github.com/unicity-sphere/sphere-sdk/issues/789); the `.code` check is
correct either way and will stay correct after the fix ships.

```typescript
try {
  await client.connect();
} catch (e) {
  const code = (e as { code?: number })?.code;
  if (code === ERROR_CODES.INCOMPATIBLE_NETWORK) {
    // data: { reason: 'network_incompatible', walletNetwork: { id: number }, clientNetwork: NetworkInfo | null }
    showWrongNetwork((e as ConnectError).data);
  } else if (code === ERROR_CODES.UNSUPPORTED_PROTOCOL_VERSION) {
    // data: { reason: 'protocol_incompatible', walletProtocol: '2.3', clientProtocol: '1.0' }
    // A version floor also sends what it demanded:
    //   npm floor      → requiredSdk: string, actualSdk: string | null (null = dApp sent none)
    //   protocol floor → requiredProtocol: string
    showUpdateRequired((e as ConnectError).data);
  } else {
    showGenericError();
  }
}
```

**Quote the versions.** `e.message` already names both sides — `SDK version 0.11.9 is below the
required minimum 0.12.0` — and `e.data` carries them structured. Replacing either with a fixed
string like "please update this app" strips the only fact that makes the refusal actionable:
*which* version to move to. Show `e.message` verbatim if you have no custom copy.

## Error Codes

| Code | Name | Description |
|------|------|-------------|
| `-32700` | `PARSE_ERROR` | Invalid JSON |
| `-32600` | `INVALID_REQUEST` | Invalid request structure |
| `-32601` | `METHOD_NOT_FOUND` | **This wallet does not implement that method or intent.** Not "you got the name wrong": `send_nft` is a valid 2.2 intent the Sphere wallet answers this way. Do not retry it and do not surface it as a user error — hide the action instead |
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
| `4201` | `INTENT_OUTCOME_UNKNOWN` | The wallet took the intent and the answer was lost (host deadline, lock, logout). **The outcome is unknown — the money may or may not have moved. Never retry**; reconcile out of band first. May carry `data.tokenId` — see below |

### 4201 with a `tokenId`

A `mint` or `mint_nft` that was journaled before the answer was lost answers 4201 with
`data: { tokenId }` — the id of the token the wallet **already began minting** and will resume on
its own. The message names it too (*"…it may still complete: this wallet resumes it. Token ID: …"*).

**Never re-issue the mint on this.** A second `mint_nft` signs and mints a second token; a second
`mint` creates a second float. Show the id, poll for it (`payments.nft(tokenId)` on the wallet side,
or your own indexer), and only act once you know whether it landed. The same rule applies without a
`tokenId`: 4201 is the one code that means "reconcile", never "retry".

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

### Mint: the subscription warm-up gate

When the wallet runs with subscriptions enabled it provisions a per-wallet key before it can certify a mint.
Until that key lands, a `mint` intent is rejected with an internal error whose message is
`Subscription is still being set up — try again in a moment`. This is **transient**, not a failure:
tell the user to retry in a moment rather than surfacing it as a hard error. It never fires on wallets
running without subscriptions.

## Protocol Constants

```typescript
import { SPHERE_CONNECT_VERSION, HOST_READY_TYPE, HOST_READY_TIMEOUT } from '@unicitylabs/sphere-sdk/connect';

SPHERE_CONNECT_VERSION // '2.3'
HOST_READY_TYPE        // 'sphere-connect:host-ready'
HOST_READY_TIMEOUT     // 30000 (ms)
```

```typescript
// The NFT payload cap is a payments constant, not a Connect one — it lives on the package root.
import { NFT_MAX_PAYLOAD_BYTES } from '@unicitylabs/sphere-sdk'; // 1048576 (1 MiB)
```

`DEFAULT_MIN_CLIENT_SDK_VERSION` (`'0.14.1-0'`) is the npm floor a host enforces. It is **not**
exported from `@unicitylabs/sphere-sdk/connect` — do not import it; a refused handshake reports the
floor in `err.data.requiredSdk` instead.

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

// NFT wire codec for the mint_nft intent
import { nftContentToWire } from '@unicitylabs/sphere-sdk/connect';
import type { MintNftIntentParams, MintNftIntentResult, NftContent } from '@unicitylabs/sphere-sdk/connect';

// Browser transport (low-level — only if not using autoConnect)
import { PostMessageTransport } from '@unicitylabs/sphere-sdk/connect/browser';
// ExtensionTransport is still exported, but the extension wallet is discontinued — do not build on it.

// Node.js transport
import { WebSocketTransport } from '@unicitylabs/sphere-sdk/connect/nodejs';
```

> **Do not annotate an `autoConnect()` client with `ConnectClient`** — it fails with TS2322 while
> [sphere-sdk#789](https://github.com/unicity-sphere/sphere-sdk/issues/789) is open. Use
> `AutoConnectResult['client']`, or let it infer.
