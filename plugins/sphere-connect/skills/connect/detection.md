# Transport Detection Utilities

Detection utilities for selecting the correct transport. These are **also exported from the SDK** — you only need a separate file if you want to customize the logic.

> **Reference only — do not scaffold this file by default.** `autoConnect()` already detects the
> transport, so a generated dApp needs no detection file at all. Write one only when the project
> deliberately wants its own logic.

> **The extension branch is dead weight.** The Sphere Chrome extension is discontinued: no supported
> wallet injects `window.sphere.isInstalled()`, so `hasExtension()` is false in practice. It is kept
> here because the SDK still exports it and `autoConnect()` still probes for it — not because it is a
> path to build on. The production transport is the **iframe**: the Sphere wallet embeds the dApp and
> speaks `PostMessageTransport` to it.

## SDK exports (recommended)

```typescript
import { isInIframe, hasExtension, detectTransport } from '@unicitylabs/sphere-sdk/connect/browser';
import type { DetectedTransport } from '@unicitylabs/sphere-sdk/connect/browser';

detectTransport(); // → 'iframe' | 'extension' | 'popup'
```

> **Note:** If you use `autoConnect()`, you don't need these at all — transport detection is built in.

## Custom detection file (TypeScript)

Only create this if you need custom detection logic.

```typescript
// src/lib/sphere-detection.ts

/** Returns true when the page is running inside an iframe (e.g., embedded in Sphere). */
export function isInIframe(): boolean {
  try {
    return window.parent !== window && window.self !== window.top;
  } catch {
    // Cross-origin access throws — we're in an iframe
    return true;
  }
}

/** Returns true when the Sphere browser extension is installed and active. */
export function hasExtension(): boolean {
  try {
    const sphere = (window as unknown as Record<string, unknown>).sphere;
    if (!sphere || typeof sphere !== 'object') return false;
    const isInstalled = (sphere as Record<string, unknown>).isInstalled;
    if (typeof isInstalled !== 'function') return false;
    return (isInstalled as () => boolean)() === true;
  } catch {
    return false;
  }
}
```

## Custom detection file (JavaScript)

```javascript
// src/sphere-detection.js

export function isInIframe() {
  try {
    return window.parent !== window && window.self !== window.top;
  } catch {
    return true;
  }
}

export function hasExtension() {
  try {
    return window.sphere?.isInstalled?.() === true;
  } catch {
    return false;
  }
}
```

## How detection works

- **`isInIframe()`**: the page is embedded inside another page (the Sphere wallet's iframe) → use
  `PostMessageTransport.forClient()`. **This is the production path**, and the only one a deployed
  dApp should be designed around.

- **`hasExtension()`**: the legacy Sphere browser extension injected `window.sphere.isInstalled()` →
  `ExtensionTransport.forClient()`. **Discontinued** — no supported wallet is behind this transport
  any more, so this branch does not fire. Never treat a truthy result as the preferred mode, and
  never tell a user to install the extension.

- **Fallback (popup)**: not in an iframe → open a wallet as a popup window. The popup must stay open
  for the connection to work, and in practice this works only against a wallet the developer runs
  themselves: `autoConnect()` opens `<walletUrl>/connect?origin=<your origin>`, and the hosted
  wallet's CDN answers **403** to any query string containing `localhost` / `127.0.0.1` (measured
  with curl — the same route with an `https` origin returns 200), so a local dApp's popup is refused
  before it reaches the wallet. To test a local dApp against the live wallet, expose it over https
  and load it as a custom agent instead (`https://sphere.unicity.network/agents/custom?url=…`,
  **https URLs only** — see SKILL.md).
