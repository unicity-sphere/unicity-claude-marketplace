# React Template: useWalletConnect Hook

Uses SDK's `autoConnect()` for automatic transport detection, silent reconnect, and full lifecycle management.

## Dependencies

Requires `@unicitylabs/sphere-sdk` installed. No separate detection file needed.

## Template

```typescript
// src/hooks/useWalletConnect.ts

import { useState, useRef, useCallback, useEffect } from 'react';
import { autoConnect, isInIframe, hasExtension } from '@unicitylabs/sphere-sdk/connect/browser';
import { WALLET_EVENTS } from '@unicitylabs/sphere-sdk/connect';
import type { AutoConnectResult, DetectedTransport } from '@unicitylabs/sphere-sdk/connect/browser';
import type { PublicIdentity, RpcMethod, IntentAction, PermissionScope } from '@unicitylabs/sphere-sdk/connect';

export interface UseWalletConnect {
  isConnected: boolean;
  isConnecting: boolean;
  isAutoConnecting: boolean;
  isWalletLocked: boolean;
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
  transportType: DetectedTransport | null;
}

const DISCONNECTED = {
  isConnected: false,
  isConnecting: false,
  isWalletLocked: false,
  identity: null as PublicIdentity | null,
  permissions: [] as readonly PermissionScope[],
  error: null as string | null,
};

const WALLET_URL = import.meta.env.VITE_WALLET_URL || 'https://sphere.unicity.network';

// TODO: Replace with your app's metadata
const DAPP_META = {
  name: 'My App',
  description: 'My dApp',
  url: typeof location !== 'undefined' ? location.origin : '',
} as const;

export function useWalletConnect(): UseWalletConnect {
  // Silent auto-connect for iframe and extension (both have persistent hosts).
  const willSilentCheck = isInIframe() || hasExtension();

  const [isAutoConnecting, setIsAutoConnecting] = useState(willSilentCheck);
  const [transportType, setTransportType] = useState<DetectedTransport | null>(null);
  const [state, setState] = useState({ ...DISCONNECTED });

  const resultRef = useRef<AutoConnectResult | null>(null);

  // Full cleanup — resets all state, usable from any disconnect path
  const fullDisconnect = useCallback(() => {
    if (resultRef.current) {
      resultRef.current.disconnect().catch(() => {});
      resultRef.current = null;
    }
    setTransportType(null);
    setState({ ...DISCONNECTED });
  }, []);

  const doConnect = useCallback(async (forceTransport?: DetectedTransport, silent?: boolean) => {
    setState(s => ({ ...s, isConnecting: true, error: null }));
    try {
      const result = await autoConnect({
        dapp: DAPP_META,
        walletUrl: WALLET_URL,
        forceTransport,
        silent,
      });
      resultRef.current = result;
      setTransportType(result.transport);
      setState({
        ...DISCONNECTED,
        isConnected: true,
        identity: result.connection.identity,
        permissions: result.connection.permissions,
      });
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Connection failed';
      setState(s => ({
        ...s,
        isConnecting: false,
        error: silent ? null : message,
      }));
    }
  }, []);

  const connect = useCallback(() => doConnect(), [doConnect]);
  const connectViaExtension = useCallback(() => doConnect('extension'), [doConnect]);
  const connectViaPopup = useCallback(() => {
    return isInIframe() ? doConnect('iframe') : doConnect('popup');
  }, [doConnect]);

  const disconnect = useCallback(async () => {
    fullDisconnect();
  }, [fullDisconnect]);

  // Auto-disconnect on transport/session errors (popup closed, refreshed, logged out)
  const handleRequestError = useCallback((err: unknown) => {
    const msg = err instanceof Error ? err.message : String(err);
    if (/not.connected|timeout|transport|closed|session/i.test(msg)) {
      fullDisconnect();
    }
    throw err;
  }, [fullDisconnect]);

  const query = useCallback(async <T = unknown>(method: RpcMethod | string, params?: Record<string, unknown>): Promise<T> => {
    if (!resultRef.current) throw new Error('Not connected');
    try {
      return await resultRef.current.client.query<T>(method, params);
    } catch (err) {
      return handleRequestError(err) as never;
    }
  }, [handleRequestError]);

  const intent = useCallback(async <T = unknown>(action: IntentAction | string, params: Record<string, unknown>): Promise<T> => {
    if (!resultRef.current) throw new Error('Not connected');
    try {
      return await resultRef.current.client.intent<T>(action, params);
    } catch (err) {
      return handleRequestError(err) as never;
    }
  }, [handleRequestError]);

  const on = useCallback((event: string, handler: (data: unknown) => void): (() => void) => {
    if (!resultRef.current) throw new Error('Not connected');
    return resultRef.current.client.on(event, handler);
  }, []);

  // Handle wallet-initiated events (auto-pushed by ConnectHost, no sphere_subscribe needed)
  useEffect(() => {
    if (!state.isConnected || !resultRef.current) return;
    const client = resultRef.current.client;

    // wallet:locked — wallet logged out or popup navigated away
    const unsubLocked = client.on(WALLET_EVENTS.LOCKED, () => {
      fullDisconnect();
    });

    // identity:changed — user switched address in wallet
    const unsubIdentity = client.on(WALLET_EVENTS.IDENTITY_CHANGED, (data) => {
      setState(s => ({ ...s, isWalletLocked: false, identity: data as PublicIdentity }));
    });

    return () => {
      unsubLocked();
      unsubIdentity();
    };
  }, [state.isConnected, fullDisconnect]);

  // Silent auto-connect on mount
  useEffect(() => {
    if (!willSilentCheck) return;
    doConnect(undefined, true)
      .catch(() => { /* silent check failed — show Connect button */ })
      .finally(() => setIsAutoConnecting(false));
  }, []); // eslint-disable-line react-hooks/exhaustive-deps

  return {
    ...state,
    isAutoConnecting,
    connect,
    connectViaExtension,
    connectViaPopup,
    disconnect,
    query,
    intent,
    on,
    extensionInstalled: hasExtension(),
    transportType,
  };
}
```

## Connection modes

| Priority | Mode | Persistent? | Notes |
|----------|------|-------------|-------|
| P1 | Embedded iframe | Yes (parent keeps running) | dApp runs inside Sphere's own iframe |
| P2 | Browser extension | Yes (service worker) | Best UX — auto-reconnects on page reload |
| P3 | Popup window | **No** — popup must stay open | Fallback when no extension installed |

## Wallet events (handled automatically)

The hook automatically handles two wallet-initiated events pushed by `ConnectHost`:

| Event | Behavior |
|-------|----------|
| `wallet:locked` | Wallet logged out or popup closed — full disconnect, shows Connect button |
| `identity:changed` | User switched address — updates `identity` in state, UI re-renders |

These events require **no `sphere_subscribe`** call — they are auto-pushed by the wallet.

### Error-based auto-disconnect

If any `query()` or `intent()` call fails with a transport/session error (e.g., popup was closed or refreshed), the hook automatically disconnects and resets state. This serves as a fallback when the `wallet:locked` event doesn't arrive.

## Usage

```tsx
import { useWalletConnect } from './hooks/useWalletConnect';

function App() {
  const wallet = useWalletConnect();

  // Loading state during auto-reconnect — prevents Connect button flash
  if (wallet.isAutoConnecting) return <div>Connecting...</div>;

  // Not connected — show connect options
  if (!wallet.isConnected) {
    return (
      <div>
        <button onClick={wallet.connect}>Connect Wallet</button>
        {/* Or specific transport: */}
        {wallet.extensionInstalled && (
          <button onClick={wallet.connectViaExtension}>Connect via Extension</button>
        )}
        <button onClick={wallet.connectViaPopup}>Connect via Popup</button>
        {wallet.error && <p style={{ color: 'red' }}>{wallet.error}</p>}
      </div>
    );
  }

  // Connected — identity updates automatically when user switches address
  return (
    <div>
      <p>Connected as {wallet.identity?.nametag} via {wallet.transportType}</p>
      <button onClick={wallet.disconnect}>Disconnect</button>
    </div>
  );
}
```

## Notes

- Replace `DAPP_META` with your actual app name and description
- `VITE_WALLET_URL` defaults to `https://sphere.unicity.network`
- `isAutoConnecting` prevents flashing the Connect button on page reload
- Extension mode auto-reconnects instantly on reload (background service worker checks approved origins)
- Popup mode requires the popup to stay open — closing it disconnects
- `autoConnect()` handles all transport detection internally — no separate detection file needed
- `identity` updates in real-time when the user switches addresses in the wallet
- Connection auto-resets on wallet logout, popup close, or transport errors
