# React Template: useWalletConnect Hook

Uses SDK's `autoConnect()` for automatic transport detection, silent reconnect, and full lifecycle management.

## Dependencies

Requires `@unicitylabs/sphere-sdk` installed. No separate detection file needed.

## Template

```typescript
// src/hooks/useWalletConnect.ts

import { useState, useRef, useCallback, useEffect } from 'react';
import { autoConnect, isInIframe, hasExtension } from '@unicitylabs/sphere-sdk/connect/browser';
import { WALLET_EVENTS, SPHERE_NETWORKS, ERROR_CODES } from '@unicitylabs/sphere-sdk/connect';
import type { AutoConnectResult, DetectedTransport } from '@unicitylabs/sphere-sdk/connect/browser';
import type { PublicIdentity, RpcMethod, IntentAction, PermissionScope } from '@unicitylabs/sphere-sdk/connect';

export interface UseWalletConnect {
  isConnected: boolean;
  isConnecting: boolean;
  isAutoConnecting: boolean;
  /** Wallet locked. Session, permissions and transport are ALIVE (Connect >= 2.1). */
  isWalletLocked: boolean;
  /** A lock ended with a DIFFERENT wallet than this session was approved for. */
  walletChanged: boolean;
  /** Bumps once per unlock that returned the SAME wallet — the retry signal for reads. */
  unlockEpoch: number;
  /** Connect version the WALLET reported at handshake ('2.1', '2.0', …) or null. */
  walletProtocol: string | null;
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
  walletChanged: false,
  unlockEpoch: 0,
  walletProtocol: null as string | null,
  identity: null as PublicIdentity | null,
  permissions: [] as readonly PermissionScope[],
  error: null as string | null,
};

/** Connect errors carry a numeric `.code`; SphereError.code is a string, so it is ignored here. */
function connectErrorCode(err: unknown): number | undefined {
  if (typeof err !== 'object' || err === null || !('code' in err)) return undefined;
  const code = (err as { code: unknown }).code;
  return typeof code === 'number' ? code : undefined;
}

/** SDK failures that carry no code at all but do mean the connection is gone. Deliberately
 *  matches neither "session" nor a bare "closed": a typed refusal that merely mentions a
 *  session must not force a disconnect. Never match on a 4009's text — it is a documented
 *  recommendation, not a wire contract. */
const CODELESS_TEARDOWN =
  /\b(not connected|disconnected|connection timeout|query timeout|intent timeout|popup was closed)\b/i;

/** True when the wallet preserves the session across a lock (Connect >= 2.1). Unknown = legacy:
 *  assuming otherwise leaves the dApp waiting forever for a wallet:unlocked that cannot arrive. */
function supportsGracefulLock(walletProtocol: string | null): boolean {
  const minor = Number(/^\d+\.(\d+)$/.exec(walletProtocol ?? '')?.[1] ?? NaN);
  return Number.isFinite(minor) && minor >= 1;
}

const WALLET_URL = import.meta.env.VITE_WALLET_URL || 'https://sphere.unicity.network';

// TODO: Replace with your app's metadata
const DAPP_META = {
  name: 'My App',
  description: 'My dApp',
  url: typeof location !== 'undefined' ? location.origin : '',
  icon: typeof location !== 'undefined' ? `${location.origin}/icon.svg` : '',
} as const;

const SESSION_KEY = 'sphere_connect_session';

export function useWalletConnect(): UseWalletConnect {
  // Silent auto-connect for iframe and extension (both have persistent hosts).
  // Also check for a saved popup session — allows resuming after page reload.
  const hasSavedSession = typeof sessionStorage !== 'undefined' && !!sessionStorage.getItem(SESSION_KEY);
  const willSilentCheck = isInIframe() || hasExtension() || hasSavedSession;

  const [isAutoConnecting, setIsAutoConnecting] = useState(willSilentCheck);
  const [transportType, setTransportType] = useState<DetectedTransport | null>(null);
  const [state, setState] = useState({ ...DISCONNECTED });

  const resultRef = useRef<AutoConnectResult | null>(null);
  // Mirrors state.identity so the event handlers, registered once per connection, compare
  // against the CURRENT identity instead of a stale closure copy.
  const identityRef = useRef<PublicIdentity | null>(null);
  useEffect(() => {
    identityRef.current = state.identity;
  }, [state.identity]);

  // Reset local state WITHOUT calling AutoConnectResult.disconnect(): that closes the popup,
  // and after a logout the user may be onboarding a new wallet in exactly that window.
  const localReset = useCallback(() => {
    resultRef.current = null;
    sessionStorage.removeItem(SESSION_KEY);
    setTransportType(null);
    setState({ ...DISCONNECTED });
  }, []);

  // Full cleanup — resets all state, usable from any disconnect path
  const fullDisconnect = useCallback(() => {
    if (resultRef.current) {
      resultRef.current.disconnect().catch(() => {});
      resultRef.current = null;
    }
    sessionStorage.removeItem(SESSION_KEY);
    setTransportType(null);
    setState({ ...DISCONNECTED });
  }, []);

  const doConnect = useCallback(async (forceTransport?: DetectedTransport, silent?: boolean) => {
    setState(s => ({ ...s, isConnecting: true, error: null }));
    try {
      const savedSession = sessionStorage.getItem(SESSION_KEY);
      const result = await autoConnect({
        dapp: DAPP_META,
        walletUrl: WALLET_URL,
        network: SPHERE_NETWORKS.testnet2, // required by the v2 compatibility gate
        forceTransport,
        silent,
        resumeSessionId: savedSession || undefined,
      });
      resultRef.current = result;
      setTransportType(result.transport);

      // Persist session for popup mode — allows resuming after page reload
      if (result.connection.sessionId) {
        sessionStorage.setItem(SESSION_KEY, result.connection.sessionId);
      }

      setState({
        ...DISCONNECTED,
        isConnected: true,
        // A resume that lands on a LOCKED wallet succeeds and reports it: connected AND locked.
        isWalletLocked: result.connection.locked === true,
        identity: result.connection.identity,
        permissions: result.connection.permissions,
        walletProtocol: result.client.walletProtocol,
      });
    } catch (err) {
      sessionStorage.removeItem(SESSION_KEY);
      // Handle v2 compatibility gate rejections with user-facing messages
      const code = connectErrorCode(err);
      let message: string;
      if (code === ERROR_CODES.INCOMPATIBLE_NETWORK) {
        message = 'Wrong network — please switch your wallet to testnet2.';
      } else if (code === ERROR_CODES.UNSUPPORTED_PROTOCOL_VERSION) {
        message = 'This app needs to be updated to connect to your wallet.';
      } else if (code === ERROR_CODES.WALLET_LOCKED) {
        // A resume with a MATCHING sessionId succeeds while locked, so this is the other case:
        // a locked wallet with nothing to resume. Nothing is broken — say so and wait.
        message = 'Your wallet is locked. Unlock it and try again.';
      } else {
        message = err instanceof Error ? err.message : 'Connection failed';
      }
      setState(s => ({
        ...s,
        isConnecting: false,
        isWalletLocked: code === ERROR_CODES.WALLET_LOCKED,
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

  // Classify a failed request by ERROR CODE, never by the message text.
  //   4009             → the session is ALIVE. Flag the lock, change nothing else, rethrow.
  //   4001 / 4004      → the connection is gone. Reset locally.
  //   codeless timeout → the connection is gone.
  //   anything else    → a typed refusal the session survives. Surface it untouched.
  const handleRequestError = useCallback((err: unknown) => {
    const code = connectErrorCode(err);

    if (code === ERROR_CODES.WALLET_LOCKED) {
      // Do NOT disconnect: the host preserved this session and will push wallet:unlocked on it.
      // The caller's promise still rejects — retrying is your decision, never a silent replay.
      setState(s => ({ ...s, isWalletLocked: true }));
      throw err;
    }

    const gone =
      code === ERROR_CODES.NOT_CONNECTED ||
      code === ERROR_CODES.SESSION_EXPIRED ||
      (code === undefined && CODELESS_TEARDOWN.test(err instanceof Error ? err.message : String(err)));

    if (gone) {
      resultRef.current = null;
      sessionStorage.removeItem(SESSION_KEY);
      setTransportType(null);
      // Preserve isWalletLocked: a codeless timeout raised WHILE the wallet is locked must not
      // erase the lock by assigning the whole DISCONNECTED constant over it.
      setState(s => ({ ...DISCONNECTED, isWalletLocked: s.isWalletLocked }));
    }

    throw err;
  }, []);

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

  // Auto-pushed by ConnectHost — no sphere_subscribe needed for any of these.
  useEffect(() => {
    if (!state.isConnected || !resultRef.current) return;
    const client = resultRef.current.client;

    // wallet:locked — a STATE, not a teardown. Session, permissions and transport all survive,
    // in EVERY transport mode. Disconnecting here orphans a host-side session that outlives the
    // lock, and the next silent auto-connect reconnects with no prompt.
    //
    // The one exception is a wallet still speaking Connect 2.0, where the same event name also
    // meant "session revoked" and wallet:unlocked will never arrive.
    const unsubLocked = client.on(WALLET_EVENTS.LOCKED, () => {
      if (!supportsGracefulLock(client.walletProtocol)) {
        localReset();
        return;
      }
      setState(s => ({ ...s, isWalletLocked: true }));
    });

    // wallet:unlocked — the SAME session continues. Check the identity BEFORE resuming. There is
    // nothing to re-subscribe: the host replays your subscription keys before pushing this.
    const unsubUnlocked = client.on(WALLET_EVENTS.UNLOCKED, (data) => {
      const next = (data as { identity?: PublicIdentity } | undefined)?.identity ?? null;
      const previous = identityRef.current;
      const sameWallet = !!next && !!previous && next.chainPubkey === previous.chainPubkey;

      if (!sameWallet) {
        // "Forgot password -> restore from recovery phrase" installs a different seed, and the
        // origin approval that authorises this session carries no identity binding.
        setState(s => ({ ...s, isWalletLocked: false, walletChanged: true, identity: next ?? s.identity }));
        return;
      }
      setState(s => ({ ...s, isWalletLocked: false, walletChanged: false, unlockEpoch: s.unlockEpoch + 1 }));
    });

    // wallet:disconnected — the session is GONE. The only teardown of the four.
    const unsubDisconnected = client.on(WALLET_EVENTS.DISCONNECTED, () => {
      localReset();
    });

    // identity:changed — the user switched address inside an UNLOCKED wallet.
    const unsubIdentity = client.on(WALLET_EVENTS.IDENTITY_CHANGED, (data) => {
      setState(s => ({ ...s, isWalletLocked: false, walletChanged: false, identity: data as PublicIdentity }));
    });

    return () => {
      unsubLocked();
      unsubUnlocked();
      unsubDisconnected();
      unsubIdentity();
    };
  }, [state.isConnected, localReset]);

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
| P3 | Popup window | **No** — popup must stay open | Fallback when no extension. Session persisted via `sessionStorage` for page-reload resume. |

## Wallet events (handled automatically)

The hook handles all four events `ConnectHost` pushes without a subscription. **The handling no
longer depends on the transport** — a lock preserves the session in popup, extension and iframe
mode alike.

| Event | What the hook does |
|-------|--------------------|
| `wallet:locked` | Sets `isWalletLocked`. Keeps the client, the transport and the saved `sessionId`. Never closes the popup. Against a Connect 2.0 wallet (`walletProtocol`) it resets locally instead — there the session was already revoked. |
| `wallet:unlocked` | Compares `identity.chainPubkey` with the one this session connected as. Same wallet → clears the lock and bumps `unlockEpoch`. Different wallet (or no identity in the payload) → clears the lock, raises `walletChanged`, resumes nothing. Sends nothing either way: the host re-armed your subscriptions before pushing the event. |
| `wallet:disconnected` | Local reset via `localReset()` — clears `resultRef`, `sessionStorage` and state. Deliberately **not** `AutoConnectResult.disconnect()`, which closes the popup the user may be onboarding a new wallet in. |
| `identity:changed` | Updates `identity`; clears `isWalletLocked` and `walletChanged`. |

These events require **no `sphere_subscribe`** call — they are auto-pushed by the wallet, and
routing one through `sphere_subscribe` is refused (it would silently never emit).

### Resuming onto a locked wallet

`doConnect()` reads `result.connection.locked`: a resume whose `sessionId` matches **succeeds**
while the wallet is locked, so the hook comes up connected **and** locked rather than failing.

### Retrying after unlock

`unlockEpoch` is the retry signal. Key a `useEffect` on it in your **read** panels:

```tsx
useEffect(() => {
  if (unlockEpoch === 0) return;   // no unlock yet
  void refetch();
}, [unlockEpoch]); // eslint-disable-line react-hooks/exhaustive-deps
```

**Never auto-replay an intent.** It moves money, and firing it on unlock means it executes with
no fresh user gesture, at the exact moment the wallet came back. The SDK does not queue or replay
a 4009'd request either — the original promise rejects and retrying is your decision.

### Error-based classification

A failed `query()` / `intent()` is classified by the numeric `.code`, not by the message text:
`4009` flags the lock and keeps everything (its `data` is `{ reason: 'locked' }`); `4001` / `4004`
and the codeless SDK timeouts reset locally; every other code is surfaced untouched. The old regex
(`/not.connected|timeout|transport|closed|session/i`) disconnected on any error whose text merely
mentioned a session.

### Who raises the unlock UI

The wallet does, from its own permanent chrome, only after a human clicks. **No dApp request can
raise the password field** — not a query, not an intent, not a handshake. A locked request lights a
passive badge in the wallet and nothing more. A forged consent dialog gains an attacker nothing; a
forged credential dialog harvests the seed password, so the two must never share a trigger.

## Connection modes

| Priority | Mode | Persistent? | Notes |
|----------|------|-------------|-------|
| P1 | Embedded iframe | Yes (parent keeps running) | dApp runs inside Sphere's own iframe |
| P2 | Browser extension | Yes (service worker) | Best UX — auto-reconnects on page reload |
| P3 | Popup window | **No** — popup must stay open | Fallback when no extension. Session persisted via `sessionStorage` for page-reload resume. |

## Wallet events (handled automatically)

The hook automatically handles the four wallet-initiated events pushed by `ConnectHost`. The
reaction no longer depends on the transport — a lock preserves the session in popup, extension
and iframe alike:

| Event | What the hook does |
|-------|--------------------|
| `wallet:locked` | Sets `isWalletLocked = true` and **nothing else**. Client, transport and saved session all survive; requests answer `WALLET_LOCKED` (4009) until the wallet is unlocked. Against a legacy Connect 2.0 wallet (`walletProtocol === '2.0'`) it still tears down — there the same event had already revoked the session. |
| `wallet:unlocked` | Compares the identity in the payload against the connected one. Same wallet → clears the flag and bumps `unlockEpoch` so read panels can refetch. Different wallet → sets `walletChanged` and resumes nothing. Subscriptions need no re-arming: the host replays them before pushing this. |
| `wallet:disconnected` | The only teardown signal. Clears client, transport and saved session, and shows the Connect button. |
| `identity:changed` | Updates `identity` in state, UI re-renders. |

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

- Replace `DAPP_META` with your actual app name, description, and icon (shown in wallet connect dialog)
- `VITE_WALLET_URL` defaults to `https://sphere.unicity.network`
- `isAutoConnecting` prevents flashing the Connect button on page reload
- Extension mode auto-reconnects instantly on reload (background service worker checks approved origins)
- Popup mode requires the popup to stay open — closing it disconnects. The `sessionId` is saved to `sessionStorage` so the session can resume after page reload (as long as the popup is still open).
- `autoConnect()` handles all transport detection internally — no separate detection file needed
- `identity` updates in real-time when the user switches addresses in the wallet
- A wallet **lock** never disconnects: `isWalletLocked` goes true, the session and the popup
  survive, and requests answer `WALLET_LOCKED` (4009) until the user unlocks. Only
  `wallet:disconnected` (logout, wallet deleted, session expiry, a different seed behind the lock
  screen) resets the connection — plus a `wallet:locked` from a legacy Connect 2.0 wallet, which
  had already revoked the session.
- Requires `@unicitylabs/sphere-sdk` ≥ 0.13.0 for `WALLET_EVENTS.UNLOCKED` / `.DISCONNECTED`,
  `ERROR_CODES.WALLET_LOCKED`, `ConnectResult.locked` and `ConnectClient.walletProtocol` /
  `.walletLocked`.
