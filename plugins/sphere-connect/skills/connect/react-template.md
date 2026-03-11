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
  disconnect: () => Promise<void>;
  query: <T = unknown>(method: RpcMethod | string, params?: Record<string, unknown>) => Promise<T>;
  intent: <T = unknown>(action: IntentAction | string, params: Record<string, unknown>) => Promise<T>;
  on: (event: string, handler: (data: unknown) => void) => () => void;
  /** True when the bridge iframe needs a popup for intent approval. */
  needsPopup: boolean;
  /** Open the approval popup (bridge mode only). */
  openApprovalPopup: () => void;
}

const WALLET_URL = import.meta.env.VITE_WALLET_URL || 'https://sphere.unicity.network';
const SESSION_KEY = 'sphere-connect-session';
const BRIDGE_APPROVED_KEY = 'sphere-connect-bridge-approved';
const OPEN_POPUP_MSG = 'sphere-connect:open-popup';

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
  const [isAutoConnecting, setIsAutoConnecting] = useState(true);
  const [needsPopup, setNeedsPopup] = useState(false);
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
  const iframeRef = useRef<HTMLIFrameElement | null>(null);
  const popupMode = useRef(false);
  const bridgeMode = useRef(false);
  const bridgeReadyRef = useRef(false);

  // TODO: Replace with your app's metadata
  const dapp = { name: 'My App', description: 'My dApp', url: location.origin } as const;

  // ---------------------------------------------------------------------------
  // Bridge iframe helpers
  // ---------------------------------------------------------------------------

  const createBridgeIframe = useCallback((): HTMLIFrameElement => {
    const existing = document.getElementById('sphere-connect-bridge') as HTMLIFrameElement | null;
    if (existing) return existing;
    const iframe = document.createElement('iframe');
    iframe.id = 'sphere-connect-bridge';
    iframe.src = WALLET_URL + '/connect-bridge?origin=' + encodeURIComponent(location.origin);
    iframe.style.display = 'none';
    iframe.setAttribute('aria-hidden', 'true');
    document.body.appendChild(iframe);
    return iframe;
  }, []);

  const waitForBridgeReady = useCallback((): Promise<void> => {
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        window.removeEventListener('message', handler);
        reject(new Error('Bridge iframe did not become ready in time'));
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
  }, []);

  /** Connect via hidden bridge iframe (always silent). */
  const connectViaBridge = useCallback(async (): Promise<ConnectClient | null> => {
    const iframe = createBridgeIframe();
    iframeRef.current = iframe;
    if (!bridgeReadyRef.current) {
      await waitForBridgeReady();
      bridgeReadyRef.current = true;
    }
    const transport = PostMessageTransport.forClient({
      target: iframe.contentWindow!,
      targetOrigin: WALLET_URL,
    });
    transportRef.current = transport;
    const client = new ConnectClient({ transport, dapp, silent: true });
    clientRef.current = client;
    try {
      const result = await client.connect();
      bridgeMode.current = true;
      popupMode.current = false;
      localStorage.setItem(BRIDGE_APPROVED_KEY, '1');
      setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
      return client;
    } catch {
      transportRef.current?.destroy();
      clientRef.current = null;
      transportRef.current = null;
      return null;
    }
  }, [createBridgeIframe, waitForBridgeReady, dapp]);

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

  const openApprovalPopup = useCallback(() => {
    const popup = window.open(
      WALLET_URL + '/connect?origin=' + encodeURIComponent(location.origin) + '&bridge',
      'sphere-wallet', 'width=420,height=650',
    );
    if (popup) popupRef.current = popup;
    setNeedsPopup(false);
  }, []);

  // ---------------------------------------------------------------------------
  // Public connect / disconnect
  // ---------------------------------------------------------------------------

  const connect = useCallback(async () => {
    setState(s => ({ ...s, isConnecting: true, error: null }));
    try {
      if (isInIframe()) {
        // P1: Inside Sphere iframe
        popupMode.current = false;
        bridgeMode.current = false;
        const transport = PostMessageTransport.forClient();
        transportRef.current = transport;
        const client = new ConnectClient({ transport, dapp });
        clientRef.current = client;
        const result = await client.connect();
        setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
        return;
      }

      if (hasExtension()) {
        // P2: Extension installed
        try {
          popupMode.current = false;
          bridgeMode.current = false;
          const transport = ExtensionTransport.forClient();
          transportRef.current = transport;
          const client = new ConnectClient({ transport, dapp });
          clientRef.current = client;
          const result = await client.connect();
          setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
          return;
        } catch {
          transportRef.current?.destroy();
          clientRef.current = null;
          transportRef.current = null;
        }
      }

      // P3: Popup for first-time approval, then bridge takeover
      popupMode.current = true;
      bridgeMode.current = false;
      const popupClient = await openPopupAndConnect();

      // Disarm popup-close detector before bridge takeover
      const popupTransport = transportRef.current;
      const popupWindow = popupRef.current;
      popupRef.current = null;
      popupMode.current = false;

      // Switch to bridge iframe for persistent session
      const bridgeClient = await connectViaBridge();
      if (bridgeClient) {
        popupClient.disconnect().catch(() => {});
        popupTransport?.destroy();
        popupWindow?.close();
      } else {
        // Bridge failed — restore popup
        popupMode.current = true;
        popupRef.current = popupWindow;
        transportRef.current = popupTransport;
        clientRef.current = popupClient;
      }
    } catch (err) {
      setState(s => ({ ...s, isConnecting: false, error: err instanceof Error ? err.message : 'Connection failed' }));
    }
  }, [dapp, openPopupAndConnect, connectViaBridge]);

  const disconnect = useCallback(async () => {
    try { await clientRef.current?.disconnect(); } catch { /* ignore */ }
    transportRef.current?.destroy();
    clientRef.current = null;
    transportRef.current = null;
    popupRef.current?.close();
    popupRef.current = null;
    popupMode.current = false;
    if (iframeRef.current) {
      iframeRef.current.remove();
      iframeRef.current = null;
    }
    bridgeMode.current = false;
    bridgeReadyRef.current = false;
    sessionStorage.removeItem(SESSION_KEY);
    localStorage.removeItem(BRIDGE_APPROVED_KEY);
    setNeedsPopup(false);
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

  // ---------------------------------------------------------------------------
  // Listen for "open-popup" messages from bridge iframe (intent approval)
  // ---------------------------------------------------------------------------
  useEffect(() => {
    function handleMessage(event: MessageEvent) {
      if (event.data?.type === OPEN_POPUP_MSG && bridgeMode.current) {
        setNeedsPopup(true);
      }
    }
    window.addEventListener('message', handleMessage);
    return () => window.removeEventListener('message', handleMessage);
  }, []);

  // Poll for popup close (popup-only mode)
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

      // P2: Extension
      if (hasExtension()) {
        try {
          const transport = ExtensionTransport.forClient();
          transportRef.current = transport;
          const client = new ConnectClient({ transport, dapp, silent: true });
          clientRef.current = client;
          const result = await client.connect();
          setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
          return;
        } catch {
          transportRef.current?.destroy();
          clientRef.current = null;
          transportRef.current = null;
        }
      }

      // P3: Bridge iframe (only if previously approved)
      if (localStorage.getItem(BRIDGE_APPROVED_KEY)) {
        const client = await connectViaBridge();
        if (client) return;
      }

      // P4: Popup session resume
      const saved = sessionStorage.getItem(SESSION_KEY);
      if (saved) {
        popupMode.current = true;
        const popup = window.open(WALLET_URL + '/connect?origin=' + encodeURIComponent(location.origin), 'sphere-wallet', 'width=420,height=650');
        if (!popup) return;
        popupRef.current = popup;
        const transport = PostMessageTransport.forClient({ target: popup, targetOrigin: WALLET_URL });
        transportRef.current = transport;
        await waitForHostReady();
        const client = new ConnectClient({ transport, dapp, resumeSessionId: saved });
        clientRef.current = client;
        const result = await client.connect();
        sessionStorage.setItem(SESSION_KEY, result.sessionId);
        setState({ isConnected: true, isConnecting: false, identity: result.identity, permissions: result.permissions, error: null });
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

  return { ...state, isAutoConnecting, connect, disconnect, query, intent, on, needsPopup, openApprovalPopup };
}
```

## Usage notes

- Replace the `dapp` metadata with the actual app name and description
- The `VITE_WALLET_URL` env var defaults to `https://sphere.unicity.network`
- `isAutoConnecting` is `true` during the initial silent check — use it to show a loading state instead of flashing the Connect button
- The hook manages the full lifecycle: transport selection, connect, disconnect, bridge iframe, popup polling, session resume
- **Bridge mode:** After first popup approval, the session switches to a hidden iframe. Closing the popup no longer disconnects. On page reload, the bridge auto-reconnects without any popup.
- **`needsPopup`:** When `true`, the bridge needs user interaction for an intent. Show a prompt and call `openApprovalPopup()` on click.
