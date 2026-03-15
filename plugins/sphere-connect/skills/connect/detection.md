# Transport Detection Utilities

Detection utilities for selecting the correct transport. These are **also exported from the SDK** — you only need a separate file if you want to customize the logic.

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

- **`isInIframe()`**: Page is embedded inside another page (Sphere's iframe) → use `PostMessageTransport.forClient()`.

- **`hasExtension()`**: Sphere browser extension injected `window.sphere.isInstalled()` → use `ExtensionTransport.forClient()`. This is the best mode — persistent connection via background service worker, auto-reconnects on page reload.

- **Fallback (popup)**: No extension, not in iframe → open Sphere as a popup window. The popup must stay open for the connection to work.
