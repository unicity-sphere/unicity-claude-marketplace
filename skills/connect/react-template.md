# React Template: useWalletConnect Hook

This is a simplified `useWalletConnect` hook for React projects. Adapt it to the developer's project structure.

## Dependencies

Requires `@unicitylabs/sphere-sdk` installed and the detection utilities from [detection.md](detection.md).

## Template

```typescript
// src/hooks/useWalletConnect.ts

import { useState, useRef, useCallback, useEffect } from 'react';
import { ConnectClient, HOST_READY_TYPE, HOST_READY_TIMEOUT } from '@unicitylabs/sphere-sdk/connect';
import { PostMessageTransport, ExtensionTransport } from '@unicitylabs/sphere-sdk/connect/browser';
import type { ConnectTransport, PublicIdentity, RpcMethod, IntentAction, PermissionScope } from '@unicitylabs/sphere-sdk/connect';
import { isInIframe, hasExtension } from '../lib/sphere-detection';

export interface UseWalletConnect {
  isConnected: boolean;
  isConnecting: boolean;
  isAutoConnecting: boolean;
  identity: PublicIdentity | null;
  permissions: readonly PermissionScope[];
  error: string | null;
  connect: () => Promise<void>;
  connectViaExtension: () => Promise<void>;
  connectViaPopup: () => Promise<void>;
  disconnect: () => Promise<void>;
  query: <T = unknown>(method: RpcMethod | string, params?: Record<string, unknown>) => Promise<T>;
  intent: <T = unknown>(action: IntentAction | string, params: Record<string, unknown>) => Promise<T>;
  on: (event: string, handler: (data: unknown) => void) => () => void;
  extensionInstalled: boolean;
}

const WALLET_URL = import.meta.env.VITE_WALLET_URL || 'https://sphere.unicity.network';
const SESSION_KEY = 'sphere-connect-session';

function waitForHostReady(): Promise<void> {
  return new Promise((resolve, reject) => {
    const timeout = setTimeout(() => {
      window.removeEventListener('message', handler);
      reject(new Error('Wallet popup did not become ready in time'));
    }, HOST_READY_TIMEOUT);
    function handler(event: MessageEvent) {
      if (event.data?.type === HOST_READY_TYPE) {
        clearTimeout(timeout);
        window.removeEventListener('message', handler);
        resolve();
      }
    }
    window.addEventListener('message', handler);
  });
}

export function useWalletConnect(): UseWalletConnect {
  const willSilentCheck = isInIframe() || hasExtension() || !!sessionStorage.getItem(SESSION_KEY);
  const [isAutoConnecting, setIsAutoConnecting] = useState(willSilentCheck);
  const [state, setState] = useState({
    isConnected: false,
    isConnecting: false,
    identity: null as PublicIdentity | null,
    permissions: [] as readonly PermissionScope[],
    error: null as string | null,
  });

  const clientRef = useRef<ConnectClient | null>(null);
  const transportRef = useRef<ConnectTransport | null>(null);
  const popupRef = useRef<Window | null>(null);
  const popupMode = useRef(false);

  // TODO: Replace with your app's metadata
  const dapp = { name: 'My App', description: 'My dApp', url: location.origin } as const;

  // ---------------------------------------------------------------------------
  // Popup helpers
  // ---------------------------------------------------------------------------

  const openPopupAndConnect = useCallback(async (): Promise<ConnectClient> => {
    if (!popupRef.current || popupRef.current.closed) {
      const popup = window.open(
        WALLET_URL + '/connect?origin=' + encodeURIComponent(location.origin),
        'sphere-wallet', 'width=420,height=650',
      );
      if (!popup) throw new Error('Popup blocked. Please allow popups for this site.');
      popupRef.current = popup;
    } else {
      popupRef.current.focus();
    }
    transportRef.current?.destroy();
    const transport = PostMessageTransport.forClient({ target: popupRef.current, targetOrigin: WALLET_URL });
    transportRef.current = transport;
    await waitForHostReady();
    const resumeSessionId = sessionStorage.getItem(SESSION_KEY) ?? undefined;
    const client = new ConnectClient({ transport, dapp, resumeSessionId });
    clientRef.current = client;
    const result = await client.connect();
    sessionStorage.setItem(SESSION_KEY, result.sessionId);
    setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
    return client;
  }, [dapp]);

  // ---------------------------------------------------------------------------
  // Public connect methods
  // ---------------------------------------------------------------------------

  const connectViaExtension = useCallback(async () => {
    setState(s => ({ ...s, isConnecting: true, error: null }));
    try {
      popupMode.current = false;
      const transport = ExtensionTransport.forClient();
      transportRef.current = transport;
      const client = new ConnectClient({ transport, dapp });
      clientRef.current = client;
      const result = await client.connect();
      setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
    } catch (err) {
      setState(s => ({ ...s, isConnecting: false, error: err instanceof Error ? err.message : 'Connection failed' }));
    }
  }, [dapp]);

  const connectViaPopup = useCallback(async () => {
    setState(s => ({ ...s, isConnecting: true, error: null }));
    try {
      if (isInIframe()) {
        // Inside Sphere iframe — connect to parent via PostMessage
        popupMode.current = false;
        const transport = PostMessageTransport.forClient();
        transportRef.current = transport;
        const client = new ConnectClient({ transport, dapp });
        clientRef.current = client;
        const result = await client.connect();
        setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
      } else {
        // Outside iframe — open popup window
        popupMode.current = true;
        await openPopupAndConnect();
      }
    } catch (err) {
      setState(s => ({ ...s, isConnecting: false, error: err instanceof Error ? err.message : 'Connection failed' }));
    }
  }, [openPopupAndConnect, dapp]);

  const connect = useCallback(async () => {
    setState(s => ({ ...s, isConnecting: true, error: null }));
    try {
      if (isInIframe()) {
        // P1: Inside Sphere iframe
        popupMode.current = false;
        const transport = PostMessageTransport.forClient();
        transportRef.current = transport;
        const client = new ConnectClient({ transport, dapp });
        clientRef.current = client;
        const result = await client.connect();
        setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
      } else if (hasExtension()) {
        // P2: Extension
        await connectViaExtension();
      } else {
        // P3: Popup (must stay open for the connection to work)
        await connectViaPopup();
      }
    } catch (err) {
      setState(s => ({ ...s, isConnecting: false, error: err instanceof Error ? err.message : 'Connection failed' }));
    }
  }, [dapp, connectViaExtension, connectViaPopup]);

  const disconnect = useCallback(async () => {
    try { await clientRef.current?.disconnect(); } catch { /* ignore */ }
    transportRef.current?.destroy();
    clientRef.current = null;
    transportRef.current = null;
    popupRef.current?.close();
    popupRef.current = null;
    popupMode.current = false;
    sessionStorage.removeItem(SESSION_KEY);
    setState({ isConnected: false, isConnecting: false, identity: null, permissions: [], error: null });
  }, []);

  const query = useCallback(async <T = unknown>(method: RpcMethod | string, params?: Record<string, unknown>): Promise<T> => {
    if (!clientRef.current) throw new Error('Not connected');
    return clientRef.current.query<T>(method, params);
  }, []);

  const intent = useCallback(async <T = unknown>(action: IntentAction | string, params: Record<string, unknown>): Promise<T> => {
    if (!clientRef.current) throw new Error('Not connected');
    return clientRef.current.intent<T>(action, params);
  }, []);

  const on = useCallback((event: string, handler: (data: unknown) => void): (() => void) => {
    if (!clientRef.current) throw new Error('Not connected');
    return clientRef.current.on(event, handler);
  }, []);

  // Poll for popup close — reset connection state when detected
  useEffect(() => {
    if (!state.isConnected || !popupMode.current) return;
    const interval = setInterval(() => {
      if (popupRef.current && popupRef.current.closed) {
        clearInterval(interval);
        transportRef.current?.destroy();
        clientRef.current = null;
        transportRef.current = null;
        popupRef.current = null;
        popupMode.current = false;
        sessionStorage.removeItem(SESSION_KEY);
        setState({ isConnected: false, isConnecting: false, identity: null, permissions: [], error: null });
      }
    }, 1000);
    return () => clearInterval(interval);
  }, [state.isConnected]);

  // Silent auto-connect on mount
  useEffect(() => {
    const silentConnect = async () => {
      // P1: Iframe — wait for parent to signal ready
      if (isInIframe()) {
        await new Promise<void>((resolve, reject) => {
          const timer = setTimeout(() => { window.removeEventListener('message', h); reject(new Error('timeout')); }, 5000);
          function h(e: MessageEvent) {
            if (e.data?.type === HOST_READY_TYPE) { clearTimeout(timer); window.removeEventListener('message', h); resolve(); }
          }
          window.addEventListener('message', h);
        });
        const transport = PostMessageTransport.forClient();
        transportRef.current = transport;
        const client = new ConnectClient({ transport, dapp, silent: true });
        clientRef.current = client;
        const result = await client.connect();
        setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
        return;
      }

      // P2: Extension — silent check if origin is already approved
      if (hasExtension()) {
        const transport = ExtensionTransport.forClient();
        transportRef.current = transport;
        const client = new ConnectClient({ transport, dapp, silent: true });
        clientRef.current = client;
        const result = await client.connect();
        setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
        return;
      }

      // P3: Popup session resume — if popup is still open, reconnect
      const saved = sessionStorage.getItem(SESSION_KEY);
      if (saved) {
        popupMode.current = true;
        await openPopupAndConnect();
      }
    };

    silentConnect()
      .catch(() => {
        transportRef.current?.destroy();
        clientRef.current = null;
        transportRef.current = null;
      })
      .finally(() => setIsAutoConnecting(false));
  }, []); // eslint-disable-line react-hooks/exhaustive-deps

  return { ...state, isAutoConnecting, connect, connectViaExtension, connectViaPopup, disconnect, query, intent, on, extensionInstalled: hasExtension() };
}
```

## Connection modes

| Priority | Mode | Persistent? | Notes |
|----------|------|-------------|-------|
| P1 | Embedded iframe | Yes (parent keeps running) | dApp runs inside Sphere's own iframe |
| P2 | Browser extension | Yes (service worker) | Best UX — no popup needed after first approval |
| P3 | Popup window | **No** — popup must stay open | Fallback when no extension installed |

> **Why not a hidden bridge iframe?** Cross-origin iframes cannot access the wallet's IndexedDB in modern Chrome (third-party storage partitioning since v115). `BroadcastChannel` is also partitioned. `requestStorageAccess()` requires a user gesture inside the iframe. For persistent connections without the extension, deploy wallet and dApp on the same origin.

## Usage notes

- Replace the `dapp` metadata with the actual app name and description
- The `VITE_WALLET_URL` env var defaults to `https://sphere.unicity.network`
- `isAutoConnecting` is `true` during the initial silent check — use it to show a loading state instead of flashing the Connect button
- **P2 (extension)** is the recommended mode for production — persistent, no popup needed
- **P3 (popup)** — closing the popup terminates the connection. Session IDs are saved to `sessionStorage` so page reloads can resume without re-approval (the popup re-opens automatically)
