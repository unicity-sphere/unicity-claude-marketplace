---
name: connect
description: >
  Sphere wallet Connect protocol integration. Use when the developer wants to
  connect their dApp to a Sphere wallet, imports @unicitylabs/sphere-sdk/connect,
  asks about wallet integration, ConnectClient, PostMessageTransport,
  ExtensionTransport, or WebSocketTransport.
user-invocable: false
---

# Sphere Connect Integration

You are helping a developer integrate the Sphere wallet Connect protocol into their project. Follow these steps:

## Step 1: Detect project context

- **Framework**: Check `package.json` for React (`react`), Vue (`vue`), Svelte (`svelte`), or Node.js (no browser framework)
- **Language**: Check for `tsconfig.json` (TypeScript) or plain JavaScript
- **Package manager**: Check for `bun.lockb` (bun), `pnpm-lock.yaml` (pnpm), `yarn.lock` (yarn), or `package-lock.json` (npm)
- **Existing SDK**: Check if `@unicitylabs/sphere-sdk` is in dependencies

## Step 2: Install SDK if missing

If `@unicitylabs/sphere-sdk` is not installed, install it:
- Browser: `npm install @unicitylabs/sphere-sdk`
- Node.js: `npm install @unicitylabs/sphere-sdk ws`

## Step 3: Generate integration code

Based on the detected framework:

### React projects
Generate three files based on the templates in this skill:
1. **Transport detection** — `src/lib/sphere-detection.ts` (see [detection.md](detection.md))
2. **Main hook** — `src/hooks/useWalletConnect.ts` (see [react-template.md](react-template.md))
3. **Environment** — Add `VITE_WALLET_URL=https://sphere.unicity.network` to `.env`

### Node.js projects
Generate one file:
1. **Client wrapper** — `src/lib/sphere-client.ts` (see [nodejs-template.md](nodejs-template.md))

### Vanilla JS projects
Generate two files:
1. **Detection** — `src/sphere-detection.js` (see [detection.md](detection.md))
2. **Client module** — `src/sphere-connect.js` — adapt the React hook pattern into a plain class/module

## Step 4: Add TypeScript path mappings (if TypeScript + browser)

Add to `tsconfig.json` `compilerOptions.paths`:
```json
{
  "paths": {
    "@unicitylabs/sphere-sdk/connect": ["./node_modules/@unicitylabs/sphere-sdk/dist/connect/index.d.ts"],
    "@unicitylabs/sphere-sdk/connect/browser": ["./node_modules/@unicitylabs/sphere-sdk/dist/impl/browser/connect/index.d.ts"]
  }
}
```

## Step 5: Show usage example

After generating files, show a concise usage example:

```typescript
import { useWalletConnect } from './hooks/useWalletConnect';

function App() {
  const { isConnected, isAutoConnecting, connect, disconnect, query, intent, on } = useWalletConnect();

  if (isAutoConnecting) return <div>Connecting...</div>;
  if (!isConnected) return <button onClick={connect}>Connect Wallet</button>;

  // Query balance
  const balance = await query('sphere_getBalance');

  // Send tokens (opens approval in wallet)
  await intent('send', { recipient: '@alice', amount: '1000000', coinId: 'UCT' });

  // Listen for events
  const unsub = on('transfer:incoming', (data) => console.log('Received:', data));
}
```

## Key patterns

### Transport priority (browser)
1. **P1 — Iframe** (`isInIframe()`) → `PostMessageTransport.forClient()` — dApp embedded inside Sphere
2. **P2 — Extension** (`hasExtension()`) → `ExtensionTransport.forClient()` — Chrome extension installed (best for production — persistent, no popup needed after first approval)
3. **P3 — Popup** (no extension) → `PostMessageTransport.forClient({ target: popup })` — popup must stay open for the connection to work

> **Why not a hidden bridge iframe?**
> Cross-origin iframes cannot access the wallet's IndexedDB in modern Chrome (third-party storage partitioning since v115). `BroadcastChannel` is also partitioned. `requestStorageAccess()` requires a user gesture inside the iframe, which is impossible for a hidden element. For persistent connections without the extension, deploy wallet and dApp on the same origin or keep the popup open.

### Silent auto-connect on mount
Always try silent connect first to avoid button flash:
```typescript
const client = new ConnectClient({ transport, dapp, silent: true });
client.connect().then(onSuccess).catch(() => { /* show Connect button */ });
```

### Popup lifecycle
- Open: `window.open(WALLET_URL + '/connect?origin=' + encodeURIComponent(location.origin), 'sphere-wallet', 'width=420,height=650')`
- Wait for ready: listen for `window.message` with `event.data.type === 'sphere-connect:host-ready'`
- Poll for close: `setInterval(() => { if (popup.closed) disconnect(); }, 1000)`
- Session resume: save `sessionId` to `sessionStorage`, pass as `resumeSessionId` on reconnect
- **Important:** closing the popup terminates the connection

### Imports
```typescript
// Core
import { ConnectClient } from '@unicitylabs/sphere-sdk/connect';
import type { ConnectTransport, PublicIdentity, RpcMethod, IntentAction, PermissionScope } from '@unicitylabs/sphere-sdk/connect';

// Browser transports
import { PostMessageTransport, ExtensionTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Node.js transport
import { WebSocketTransport } from '@unicitylabs/sphere-sdk/connect/nodejs';

// Constants
import { HOST_READY_TYPE, HOST_READY_TIMEOUT } from '@unicitylabs/sphere-sdk/connect';
```

## DO NOT

- Generate wallet-side `ConnectHost` code — that's only for wallet developers
- Install packages without asking the user first
- Hardcode API keys or private keys
- Override existing connect integration if files already exist
- Generate overly complex abstractions — keep it minimal and readable

## Full API reference

For complete RPC methods, intents, permissions, events, and error codes, see [reference.md](reference.md).
