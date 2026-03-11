# Transport Detection Utilities

These utilities detect the browser environment to select the correct transport.

## TypeScript

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

## JavaScript

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

- **`isInIframe()`**: Checks if the page is embedded inside another page. When true, the dApp is loaded inside Sphere's iframe and should use `PostMessageTransport.forClient()` to talk to the parent window directly.

- **`hasExtension()`**: Checks if the Sphere browser extension injected `window.sphere.isInstalled()`. When true, the dApp should use `ExtensionTransport.forClient()` for native Chrome extension messaging.

- **Bridge iframe (auto-reconnect)**: If neither iframe nor extension is detected but the origin was previously approved (`localStorage` has `sphere-connect-bridge-approved`), the dApp creates a hidden `<iframe src="WALLET_URL/connect-bridge?origin=...">` and uses `PostMessageTransport.forClient({ target: iframe.contentWindow, targetOrigin: WALLET_URL })` with `silent: true`. The session persists even when the popup is closed.

- **Fallback (popup)**: If no prior approval exists, the dApp opens Sphere as a popup window for first-time approval. After approval, the session switches to a bridge iframe for persistence (bridge takeover).
