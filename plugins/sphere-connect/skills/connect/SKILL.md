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
- **Bundler / meta-framework**: this decides the env-var syntax and where browser-only code may run.
  Check for `next` in dependencies or a `next.config.*` (Next.js), otherwise `vite.config.*` (Vite),
  otherwise `webpack.config.*` / CRA (`react-scripts`).
  - **Vite** — `import.meta.env.VITE_*`, and top-level browser code is fine.
  - **Next.js** — `process.env.NEXT_PUBLIC_*`, the hook file needs `'use client'`, and
    `isInIframe()` (and anything else reading `window` / `location`) must only be called inside
    `useEffect` — `window` does not exist during the server render, and a value that differs
    between the server and the first client render is a hydration mismatch. See the Next.js
    section of [react-template.md](react-template.md).
  - **Anything else** — use that bundler's own public-env prefix and keep the rest of the template unchanged.
- **Language**: Check for `tsconfig.json` (TypeScript) or plain JavaScript
- **Package manager**: Check for `bun.lockb` (bun), `pnpm-lock.yaml` (pnpm), `yarn.lock` (yarn), or `package-lock.json` (npm)
- **Existing SDK**: Check if `@unicitylabs/sphere-sdk` is in dependencies

## Step 2: Install SDK if missing

If `@unicitylabs/sphere-sdk` is not installed, install it:
- Browser: `npm install @unicitylabs/sphere-sdk`
- Node.js: `npm install @unicitylabs/sphere-sdk ws` — plus `npm install -D @types/ws` on a
  TypeScript project. `ws` ships no declarations of its own, so without it `import WebSocket from 'ws'`
  fails with TS7016 under `strict`.

Install `0.17.x`. The wallet refuses a handshake from a dApp reporting an npm SDK below
**`0.14.1`** (`DEFAULT_MIN_CLIENT_SDK_VERSION = '0.14.1-0'`) with `UNSUPPORTED_PROTOCOL_VERSION`
(4007) before any UI appears.

## Step 3: Generate integration code

Based on the detected framework:

### React projects (recommended: autoConnect)
Generate one file + env:
1. **Main hook** — `src/hooks/useWalletConnect.ts` (see [react-template.md](react-template.md))
2. **Environment** — add the wallet URL and the target network to `.env`, using the prefix the
   bundler detected in step 1:
   - Vite: `VITE_WALLET_URL=https://sphere.unicity.network` and `VITE_SPHERE_NETWORK=testnet2`
   - Next.js: `NEXT_PUBLIC_WALLET_URL=…` and `NEXT_PUBLIC_SPHERE_NETWORK=testnet2`

   The network is a **config value, not a constant**: both `mainnet` and `testnet2` are live, and a
   dApp that hardcodes the wrong one is refused with `INCOMPATIBLE_NETWORK` (4008) at the handshake.

For Vue or Svelte, generate the same logic as a **framework-neutral module** (a plain class or a
factory over `autoConnect()`), not the React hook — then wire it into that framework's own
reactivity (a Vue composable over `ref`s, a Svelte store). Do not hand a Vue or Svelte project a
file that imports from `react`.

No separate detection file needed — `autoConnect()` handles transport detection internally.

### Node.js projects — dApp connecting to a wallet, vs. own-wallet bot

These are two different shapes — check which one the user actually wants before generating:

- **"Connect to a wallet" / "talk to Sphere over Connect" / "Node dApp"** → this is a client that
  connects *to* someone else's already-running wallet. Generate one file:
  1. **Client wrapper** — `src/lib/sphere-client.ts` (see [nodejs-template.md](nodejs-template.md))
- **"Build a bot" / "give it its own wallet" / "own-wallet agent" / "run a wallet headlessly"** →
  this bot **is** the wallet — its own keys, its own storage, direct SDK usage, no Connect protocol
  at all. Generate the bot's own-wallet init + runtime + coin helpers (see
  [bot-template.md](bot-template.md)). `WALLET_API_URL` is **required**: the payments vertical is
  composed from the wallet-api transport config, and `Sphere.init` without it throws
  `INVALID_CONFIG`. There is no "send + DM only" mode to fall back to.

### Vanilla JS projects
Generate one file:
1. **Client module** — `src/sphere-connect.js` — use `autoConnect()` from the SDK (see vanilla
   example below). `autoConnect()` detects the transport itself, so do **not** generate a separate
   `src/sphere-detection.js`; [detection.md](detection.md) is reference material for a project that
   deliberately wants its own detection, not a file to scaffold by default.

## Step 4: Check `moduleResolution` (if TypeScript)

The SDK publishes `./connect` and `./connect/browser` through the `exports` map only. Whether
TypeScript can see them is decided by one setting — **check it, do not add path mappings
reflexively.**

Read `compilerOptions.moduleResolution` in the project's `tsconfig.json`:

- **`bundler`, `node16` or `nodenext`** → **do nothing.** These read `exports`, the subpaths resolve,
  and adding `paths` here only pins the app to a file layout the SDK is free to change.
- **`node` / `node10`, or unset with a CommonJS `module`** → recommend switching to `bundler`
  (Vite, webpack, esbuild, Rollup) or `nodenext` (plain Node). That is the real fix, and it is what
  [sphere-sdk#789](https://github.com/unicity-sphere/sphere-sdk/issues/789) asks consumers to do.

Only if the project cannot move off `node10` resolution, add the fallback mapping:

```json
{
  "paths": {
    "@unicitylabs/sphere-sdk/connect": ["./node_modules/@unicitylabs/sphere-sdk/dist/connect/index.d.ts"],
    "@unicitylabs/sphere-sdk/connect/browser": ["./node_modules/@unicitylabs/sphere-sdk/dist/impl/browser/connect/index.d.ts"]
  }
}
```

**This fallback is for applications only.** Do not put it in a **library** that emits its own
declarations (`"declaration": true`): the paths are not part of the published package contract, so
the emitted `.d.ts` can end up pointing at `dist/...` files that the library's own consumers never
resolve, and a file that moves becomes a silent `any` rather than an error.

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
await wallet.intent('send', { to: '@alice', amount: '1000000000000000000', coinId: '<lowercase 64-hex coin id>' }); // amount in base units
const unsub = wallet.on('transfer:incoming', (data) => console.log('Received:', data));
```

### Vanilla JS example
```html
<button id="connect">Connect Wallet</button>
<script type="module">
import { autoConnect, isInIframe } from '@unicitylabs/sphere-sdk/connect/browser';
import { SPHERE_NETWORKS, ERROR_CODES } from '@unicitylabs/sphere-sdk/connect';

// The target network is configuration, not a constant: mainnet (id 1) and testnet2 (id 4) are
// both live, and the wallet refuses a handshake for the other one with INCOMPATIBLE_NETWORK
// (4008). With no bundler to inject an env var, read it from wherever this app keeps config —
// here, a data attribute on <html data-sphere-network="testnet2">.
const NETWORK_NAME = document.documentElement.dataset.sphereNetwork ?? 'testnet2';
const NETWORK = SPHERE_NETWORKS[NETWORK_NAME];
if (!NETWORK) throw new Error(`Unknown Sphere network: ${NETWORK_NAME}`);

const SESSION_KEY = 'sphere_connect_session';

let wallet = null;

// Silent auto-reconnect on page load — only when there is something to resume.
// Inside the wallet's own iframe the host is already there; standalone, a silent attempt
// with no saved session just opens and closes a popup for nothing.
const savedSession = sessionStorage.getItem(SESSION_KEY);
if (isInIframe() || savedSession) {
  try {
    wallet = await autoConnect({
      dapp: { name: 'My App', url: location.origin },
      walletUrl: 'https://sphere.unicity.network',
      network: NETWORK, // required by the v2 compatibility gate
      resumeSessionId: savedSession ?? undefined,
      silent: true,
    });
    sessionStorage.setItem(SESSION_KEY, wallet.connection.sessionId);
    document.getElementById('connect').textContent = `Connected: ${wallet.connection.identity.nametag}`;
  } catch {
    // Not approved yet — wait for button click
    sessionStorage.removeItem(SESSION_KEY);
  }
}

document.getElementById('connect').onclick = async () => {
  try {
    wallet = await autoConnect({
      dapp: { name: 'My App', url: location.origin },
      walletUrl: 'https://sphere.unicity.network',
      network: NETWORK,
    });
    sessionStorage.setItem(SESSION_KEY, wallet.connection.sessionId);
    console.log('Connected:', wallet.connection.identity);
  } catch (err) {
    const code = err?.code;
    // A gate refusal carries the versions it compared in err.data, and names both sides
    // in err.message. Say them — do not replace them with "please update".
    if (code === ERROR_CODES.INCOMPATIBLE_NETWORK) {
      alert(`Wrong network — your wallet is on network ${err.data?.walletNetwork?.id}, this app targets ${NETWORK.name}.`);
    } else if (code === ERROR_CODES.UNSUPPORTED_PROTOCOL_VERSION) {
      alert(err.data?.requiredSdk
        ? `Update this app: it uses sphere-sdk ${err.data.actualSdk ?? '(not reported)'}, the wallet requires ${err.data.requiredSdk} or newer.`
        : err.message);
    } else {
      alert('Connection failed: ' + err.message);
    }
  }
};
</script>
```

### Testing a local dApp against the hosted wallet

Serve the dApp over **https on a publicly reachable host** and load it as a **custom agent**, which
embeds it in the wallet's own iframe — the P1 transport, the same one a published dApp gets:

```
https://sphere.unicity.network/agents/custom?url=<url-encoded https dApp url>
```

An https tunnel in front of the dev server (`cloudflared`, `ngrok`) is the quickest way to get such
a URL, and a tunnel host is the shape of URL that was measured as reachable. Two independent gates
stand between a plain `http://localhost:5173` dev server and that iframe, and **neither of them is
the wallet refusing the popup route**:

- **The CDN rejects local URLs in the query string.** Measured with `curl` on 2026-09-17 (HTTP
  status codes only): `GET https://sphere.unicity.network/connect` → **200**,
  `/connect?origin=https%3A%2F%2Fexample.com` → **200**,
  `/agents/custom?url=https%3A%2F%2Ffoo.ngrok.app` → **200**, plain home page → **200**. But **any**
  query string containing `localhost` or `127.0.0.1` → **403**, served by CloudFront ("ERROR: The
  request could not be satisfied"), on every route tested and with or without browser-like
  `User-Agent` / `Accept` headers. So it is a CDN/WAF rule about local URLs in the query — not the
  wallet, and not specific to `/connect`. It is also what makes the **popup** path look broken
  against the hosted wallet: `autoConnect()` opens `<walletUrl>/connect?origin=<your origin>`, so a
  dApp on `http://localhost:5173` puts `localhost` in the query and the request is refused before
  the wallet sees it.
- **The wallet frames a custom tab only when the URL is `https`.** `isHttpsUrl` is a protocol-only
  check (sphere `src/components/desktop/DesktopLayout.tsx:80`), so an `http://` URL is dropped
  **silently**: the tab falls back to the wallet's "Load Custom URL" prompt with no error and no
  console message, which reads like the dApp failed to load.

Typing an `https://` URL into that in-app "Load Custom URL" prompt avoids the query string entirely
and should clear the https gate, but **that path was not tested end to end** — do not present it as
the known-good route. (The prompt normalises a bare `localhost:5173` to `http://localhost:5173`,
which the https gate then drops, so type the full `https://` URL.)

Against a wallet the developer runs themselves (the sphere dev server on `localhost:5173`) none of
this applies: popup mode and localhost are fine there.

## Key concepts

### Wallet events (critical — always handle)

The wallet pushes four events automatically after connection — **no `sphere_subscribe` needed**:

- **`wallet:locked`** — the wallet is locked. **The session, the permissions and the transport are
  still alive** (Connect ≥ 2.1); requests answer `WALLET_LOCKED` (4009) with
  `data: { reason: 'locked' }` until it is unlocked. Set a flag, show a locked state, and **do
  not** disconnect or clear the saved session — in any transport mode. Only if
  `client.walletProtocol` is `2.0` does this still mean "session revoked".
- **`wallet:unlocked`** — the same session continues; no re-handshake, no re-approval, and nothing
  to re-subscribe (the host re-arms your subscriptions before pushing this). The payload carries
  `{ identity? }`: **compare `identity.chainPubkey` with the one you connected as before resuming
  anything**, because the lock screen's "restore from recovery phrase" installs a different seed.
- **`wallet:disconnected`** — the session is gone (logout, wallet deleted, expiry, a different seed
  behind the lock screen). This is the only one that means "clear everything and re-handshake".
- **`identity:changed`** — user switched address. Update the displayed identity.

While locked, `sphere_getIdentity`, `sphere_subscribe`, `sphere_unsubscribe` and
`sphere_disconnect` are still answered normally; everything else — including every intent — gets
4009 in the same tick. Balances, tokens and
history are never served and never cached — and neither is anything else: the ten refused
methods (fourteen RPC methods minus the four above) include `sphere_resolve` and **all four DM
reads**, so messaging does NOT keep working while locked. Stop issuing reads and wait for the
unlock; do not poll into refusals.

A wallet that **cold-starts locked** (a wallet-page reload or a fresh popup — the password is
memory-only) holds no session, so the HANDSHAKE itself is refused with an errorless empty
response: `connect()` rejects with no `.code` at all. Do not write
`if (err.code === ERROR_CODES.WALLET_LOCKED)` expecting to catch that case — treat an unexpected
rejection as "not ready yet" and retry on the next `HOST_READY`.

A resume handshake whose `sessionId` matches **succeeds** while the wallet is locked and the
result carries `locked: true`, so check `result.locked` after `connect()`.

> **Host-side note:** a wallet maps each transition to exactly one verb — `setLocked()` for a lock
> (session preserved), `updateSphere()` for an unlock, `revokeSession()` for a logout,
> `setUnavailable()` when Sphere is gone for a non-lock reason. `notifyWalletLocked()` has been
> **removed**: its old meaning was *revoke* and its new meaning would be *lock*, so an alias would
> have inverted every call site silently.
>
> **The unlock UI is raised by the wallet, from its own chrome, only after a human clicks.** No
> dApp request — query, intent or handshake — can raise the password field. A locked request
> lights a passive badge and nothing more.

Additionally, classify a failed `query()`/`intent()` by the numeric `.code`, never by the message
text. See [react-template.md](react-template.md) for implementation.

### Session persistence (popup mode)

Popup connections are not persistent — if the user reloads the page, the connection is lost unless the session is saved. The React template saves `connection.sessionId` to `sessionStorage` after connect and passes `resumeSessionId` to `autoConnect()` on mount. This allows the popup session to resume after a page reload (as long as the popup is still open). On disconnect or connection failure, the saved session is cleared.

### safeSend for WebSocket integrations

When using `WebSocketTransport` in Node.js, always guard against writes to a closing or closed socket. The [nodejs-template.md](nodejs-template.md) patches `ws.send` automatically, or you can use a standalone guard:
```typescript
const safeSend = (data: string) => {
  if (ws.readyState === WebSocket.OPEN) ws.send(data);
};
```
This prevents `"WebSocket is not open"` errors during disconnect races.

### autoConnect() — the recommended way

`autoConnect()` from `@unicitylabs/sphere-sdk/connect/browser` handles everything automatically:
- Detects the transport (iframe → extension → popup; the extension probe never matches in practice)
- Handles the full handshake lifecycle
- Supports silent auto-reconnect on page reload
- Returns a `client` for queries, intents, and events

```typescript
import { autoConnect } from '@unicitylabs/sphere-sdk/connect/browser';
import { SPHERE_NETWORKS } from '@unicitylabs/sphere-sdk/connect';

// One function — that's it.
// network is required by the v2 compatibility gate — the wallet rejects
// the handshake with INCOMPATIBLE_NETWORK (4008) if it is missing or wrong.
// Read it from config (SPHERE_NETWORKS.mainnet / .testnet2) rather than hardcoding it.
const result = await autoConnect({
  dapp: { name: 'My App', url: location.origin },
  walletUrl: 'https://sphere.unicity.network',
  network: SPHERE_NETWORKS.testnet2,
  silent: true, // auto-reconnect without UI — see the guard below
});

result.client.query('sphere_getBalance');
result.client.intent('send', { to: '@alice', amount: '1000000000000000000', coinId: '<lowercase 64-hex coin id>' }); // amount in base units
result.client.on('transfer:incoming', (data) => console.log(data));
await result.disconnect();
```

### Transport priority (browser)

| Priority | Mode | When | Persistent? | Status |
|----------|------|------|-------------|--------|
| P1 | Iframe | `isInIframe()` — dApp embedded by the Sphere wallet | Yes | **The supported production path** |
| P2 | Extension | `hasExtension()` — legacy Chrome extension | Yes | **Dead** — no supported wallet is behind it |
| P3 | Popup | Fallback when standalone | No — popup must stay open | Local development against a self-hosted wallet |

**Ship for P1.** A production dApp runs **inside the Sphere wallet**, which frames it and speaks
`PostMessageTransport` to it. That is the mode to design the UI for and the one to test.

**Do not recommend the browser extension.** The SDK still exports `ExtensionTransport` and
`hasExtension()`, and `autoConnect()` still probes for it, but the extension wallet is
discontinued — there is no supported wallet behind that transport. `hasExtension()` is therefore
false in practice; treat a truthy result as an unsupported environment, never as the good path.
Do not build an "Install the extension" call to action, and do not gate features on it.

**Popup (P3)** is a development affordance against a wallet the developer runs themselves. Pointed
at the hosted wallet from a local dev server it never reaches the wallet at all: `autoConnect()`
opens `<walletUrl>/connect?origin=<your origin>`, and the CDN answers **403** to any query string
containing `localhost` / `127.0.0.1` (measured with curl; the same route with an `https` origin
returns 200). See "Testing a local dApp against the hosted wallet" above.

### Silent auto-connect on page load

`silent: true` avoids flashing the Connect button — but **guard it**. Fire it only where there is
something to resume: inside the wallet's iframe, or with a session saved from an earlier connect.
Standalone with nothing saved, a silent attempt opens and closes a popup window on every page load
for a handshake that is refused anyway.

```typescript
const saved = sessionStorage.getItem('sphere_connect_session');
if (isInIframe() || saved) {
  try {
    const result = await autoConnect({
      dapp, walletUrl, network: SPHERE_NETWORKS.testnet2,
      resumeSessionId: saved ?? undefined,
      silent: true,
    });
    // Reconnected — origin was already approved
  } catch {
    // Not approved — show Connect button
  }
}
```

### Forcing a specific transport

```typescript
await autoConnect({ dapp, walletUrl, forceTransport: 'iframe' }); // embedded in the wallet
await autoConnect({ dapp, walletUrl, forceTransport: 'popup' });  // self-hosted wallet, dev only
```

`forceTransport: 'extension'` exists but has no wallet behind it — see the table above.

### Imports
```typescript
// Recommended: autoConnect (handles everything)
import { autoConnect } from '@unicitylabs/sphere-sdk/connect/browser';
import type { AutoConnectResult, DetectedTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Detection utilities (also available standalone)
import { isInIframe, hasExtension, detectTransport } from '@unicitylabs/sphere-sdk/connect/browser';

// Low-level (only if you need manual control)
import { ConnectClient, ConnectError, SPHERE_NETWORKS, ERROR_CODES } from '@unicitylabs/sphere-sdk/connect';
import { PostMessageTransport } from '@unicitylabs/sphere-sdk/connect/browser';
import type { ConnectTransport, NetworkInfo, PublicIdentity, RpcMethod, IntentAction, PermissionScope } from '@unicitylabs/sphere-sdk/connect';

// Node.js
import { WebSocketTransport } from '@unicitylabs/sphere-sdk/connect/nodejs';
```

> **Do not annotate an `autoConnect()` client with `ConnectClient`.** Up to and including 0.17.2,
> `./connect/browser` ships its **own** declaration of `ConnectClient`, so
> `const c: ConnectClient = (await autoConnect(…)).client` fails with **TS2322** — *"Types have
> separate declarations of a private property 'transport'"*. Let it infer, or name it
> `AutoConnectResult['client']`:
>
> ```typescript
> import type { AutoConnectResult } from '@unicitylabs/sphere-sdk/connect/browser';
> let client: AutoConnectResult['client'] | null = null;
> ```
>
> The same split makes `err instanceof ConnectError` **false** for errors thrown by `autoConnect()`
> — discriminate on `err.code`. Both are fixed by
> [sphere-sdk#789](https://github.com/unicity-sphere/sphere-sdk/issues/789); until that ships, the
> only alternative to the alias is an `as unknown as ConnectClient` cast, which is worse.

## DO NOT

- Generate wallet-side `ConnectHost` code — that's only for wallet developers
- Recommend the Chrome extension, or generate an "Install the extension" flow — that wallet is discontinued
- Install packages without asking the user first
- Hardcode API keys or private keys
- Hardcode the target network — read it from config; mainnet and testnet2 are both live
- Override existing connect integration if files already exist
- Generate overly complex abstractions — keep it minimal and readable
- Use hidden bridge iframes for cross-origin connections (broken by third-party storage partitioning in Chrome v115+)

## Backend authentication

If the developer asks to **"authenticate" / "verify a signature" / "sign in with wallet" / "backend"
login** (as opposed to connecting a frontend to a wallet), route there directly — do not generate a
Connect-only integration for this ask. See [backend-auth.md](backend-auth.md) for the full
challenge-response → JWT flow (byte-exact challenge reconstruction, signature recovery, no `Sphere`
instance required to verify).

## Own-wallet bots

If the developer asks to **"build a bot" / "give an agent its own wallet" / "own wallet"**, that is
not a Connect integration either — see the Node.js section above and
[bot-template.md](bot-template.md).

## Full API reference

For complete RPC methods, intents, permissions, events, and error codes, see [reference.md](reference.md).
