---
name: connect
description: >
  Sphere wallet Connect protocol integration. Use when the developer wants to
  connect their dApp to a Sphere wallet, imports @unicitylabs/sphere-sdk/connect,
  asks about wallet integration, ConnectClient, autoConnect, PostMessageTransport,
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

### React projects (recommended: autoConnect)
Generate one file + env:
1. **Main hook** — `src/hooks/useWalletConnect.ts` (see [react-template.md](react-template.md))
2. **Environment** — Add `VITE_WALLET_URL=https://sphere.unicity.network` to `.env`

No separate detection file needed — `autoConnect()` handles transport detection internally.

### Node.js projects
Generate one file:
1. **Client wrapper** — `src/lib/sphere-client.ts` (see [nodejs-template.md](nodejs-template.md))

### Vanilla JS projects
Generate two files:
1. **Detection** — `src/sphere-detection.js` (see [detection.md](detection.md))
2. **Client module** — `src/sphere-connect.js` — use `autoConnect()` from SDK (see vanilla example below)

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

```tsx
import { useWalletConnect } from './hooks/useWalletConnect';

function App() {
  const wallet = useWalletConnect();

  if (wallet.isAutoConnecting) return <div>Connecting...</div>;
  if (!wallet.isConnected) return <button onClick={wallet.connect}>Connect Wallet</button>;

  return <div>Connected as {wallet.identity?.nametag}</div>;
}

// In your components:
const balance = await wallet.query('sphere_getBalance');
await wallet.intent('send', { recipient: '@alice', amount: '1000000', coinId: 'UCT' });
const unsub = wallet.on('transfer:incoming', (data) => console.log('Received:', data));
```

### Vanilla JS example
```html
<button id="connect">Connect Wallet</button>
<script type="module">
import { autoConnect } from '@unicitylabs/sphere-sdk/connect/browser';

let wallet = null;

// Try silent auto-reconnect on page load
try {
  wallet = await autoConnect({
    dapp: { name: 'My App', url: location.origin },
    walletUrl: 'https://sphere.unicity.network',
    silent: true,
  });
  document.getElementById('connect').textContent = `Connected: ${wallet.connection.identity.nametag}`;
} catch {
  // Not approved yet — wait for button click
}

document.getElementById('connect').onclick = async () => {
  wallet = await autoConnect({
    dapp: { name: 'My App', url: location.origin },
    walletUrl: 'https://sphere.unicity.network',
  });
  console.log('Connected:', wallet.connection.identity);
};
</script>
```

## Key concepts

### autoConnect() — the recommended way

`autoConnect()` from `@unicitylabs/sphere-sdk/connect/browser` handles everything automatically:
- Detects the best transport (iframe → extension → popup)
- Handles the full handshake lifecycle
- Supports silent auto-reconnect on page reload
- Returns a `client` for queries, intents, and events

```typescript
import { autoConnect } from '@unicitylabs/sphere-sdk/connect/browser';

// One function — that's it
const result = await autoConnect({
  dapp: { name: 'My App', url: location.origin },
  walletUrl: 'https://sphere.unicity.network',
  silent: true, // auto-reconnect without UI
});

result.client.query('sphere_getBalance');
result.client.intent('send', { recipient: '@alice', amount: '1000000', coinId: 'UCT' });
result.client.on('transfer:incoming', (data) => console.log(data));
await result.disconnect();
```

### Transport priority (browser)

| Priority | Mode | When | Persistent? |
|----------|------|------|-------------|
| P1 | Iframe | `isInIframe()` — dApp embedded in Sphere | Yes |
| P2 | Extension | `hasExtension()` — Chrome extension installed | Yes (best UX) |
| P3 | Popup | Fallback | No — popup must stay open |

**Extension (P2) is the best mode for production** — the background service worker is always running, so:
- Silent auto-reconnect works on every page reload
- No popup needed after first approval
- Wallet remembers approved origins in `chrome.storage.local`

### Silent auto-connect on page load

Always try `silent: true` first to avoid flashing the Connect button:
```typescript
try {
  const result = await autoConnect({ dapp, walletUrl, silent: true });
  // Reconnected — origin was already approved
} catch {
  // Not approved — show Connect button
}
```

For **extension mode**, silent connect works even if the wallet popup is not open — the background service worker handles it.

### Forcing a specific transport

```typescript
await autoConnect({ dapp, walletUrl, forceTransport: 'extension' });
await autoConnect({ dapp, walletUrl, forceTransport: 'popup' });
```

### Imports
```typescript
// Recommended: autoConnect (handles everything)
import { autoConnect } from '@unicitylabs/sphere-sdk/connect/browser';
import type { AutoConnectResult, DetectedTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Detection utilities (also available standalone)
import { isInIframe, hasExtension, detectTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Low-level (only if you need manual control)
import { ConnectClient } from '@unicitylabs/sphere-sdk/connect';
import { PostMessageTransport, ExtensionTransport } from '@unicitylabs/sphere-sdk/connect/browser';
import type { ConnectTransport, PublicIdentity, RpcMethod, IntentAction, PermissionScope } from '@unicitylabs/sphere-sdk/connect';

// Node.js
import { WebSocketTransport } from '@unicitylabs/sphere-sdk/connect/nodejs';
```

## DO NOT

- Generate wallet-side `ConnectHost` code — that's only for wallet developers
- Install packages without asking the user first
- Hardcode API keys or private keys
- Override existing connect integration if files already exist
- Generate overly complex abstractions — keep it minimal and readable
- Use hidden bridge iframes for cross-origin connections (broken by third-party storage partitioning in Chrome v115+)

## Full API reference

For complete RPC methods, intents, permissions, events, and error codes, see [reference.md](reference.md).
