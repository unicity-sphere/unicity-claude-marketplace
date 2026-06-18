# Node.js Template: WebSocket Client

This template creates a Sphere Connect client for Node.js applications using WebSocket transport.

## Dependencies

```bash
npm install @unicitylabs/sphere-sdk ws
```

## Template

```typescript
// src/lib/sphere-client.ts

import { ConnectClient, SPHERE_NETWORKS, ERROR_CODES, RPC_METHODS, INTENT_ACTIONS } from '@unicitylabs/sphere-sdk/connect';
import { WebSocketTransport } from '@unicitylabs/sphere-sdk/connect/nodejs';
import type { RpcMethod, IntentAction, PublicIdentity, PermissionScope } from '@unicitylabs/sphere-sdk/connect';
import WebSocket from 'ws';

export interface SphereClientConfig {
  /** WebSocket URL of the wallet host (e.g., 'ws://localhost:8765') */
  url: string;
  /** dApp metadata shown to the user during connection approval */
  dapp: { name: string; description: string; url: string };
}

export interface SphereClient {
  identity: PublicIdentity;
  permissions: readonly PermissionScope[];
  query: <T = unknown>(method: RpcMethod | string, params?: Record<string, unknown>) => Promise<T>;
  intent: <T = unknown>(action: IntentAction | string, params: Record<string, unknown>) => Promise<T>;
  on: (event: string, handler: (data: unknown) => void) => () => void;
  disconnect: () => Promise<void>;
}

/**
 * Connect to a Sphere wallet via WebSocket.
 *
 * Usage:
 *   const client = await connectToSphere({
 *     url: 'ws://localhost:8765',
 *     dapp: { name: 'My App', description: 'My server app', url: 'https://myapp.com' },
 *   });
 *
 *   const balance = await client.query('sphere_getBalance');
 *   await client.intent('send', { to: '@alice', amount: '10', coinId: '<lowercase 64-hex coin id>' });
 *   await client.disconnect();
 */
export async function connectToSphere(config: SphereClientConfig): Promise<SphereClient> {
  const transport = WebSocketTransport.createClient({
    url: config.url,
    createWebSocket: (url: string) => {
      const ws = new WebSocket(url);
      // Guard against writes to a closing/closed socket — prevents
      // "WebSocket is not open" errors during disconnect races.
      const origSend = ws.send.bind(ws);
      ws.send = (data: any) => {
        if (ws.readyState === WebSocket.OPEN) origSend(data);
      };
      return ws as any;
    },
    autoReconnect: false,
  });

  await transport.connect();

  // network is required by the v2 compatibility gate; wallet rejects with
  // INCOMPATIBLE_NETWORK (4008) if this does not match its active network.
  const client = new ConnectClient({
    transport,
    dapp: config.dapp,
    network: SPHERE_NETWORKS.testnet2,
  });

  let result;
  try {
    result = await client.connect();
  } catch (err) {
    transport.destroy();
    const code = (err as { code?: number })?.code;
    if (code === ERROR_CODES.INCOMPATIBLE_NETWORK) {
      throw new Error('Wallet is on a different network — ensure the wallet uses testnet2.');
    }
    if (code === ERROR_CODES.UNSUPPORTED_PROTOCOL_VERSION) {
      throw new Error('Protocol version mismatch — update @unicitylabs/sphere-sdk to the latest version.');
    }
    throw err;
  }

  return {
    identity: result.identity,
    permissions: result.permissions,
    query: (method, params) => client.query(method, params),
    intent: (action, params) => client.intent(action, params),
    on: (event, handler) => client.on(event, handler),
    disconnect: async () => {
      await client.disconnect();
      transport.destroy();
    },
  };
}

// Re-export constants for convenience
export { RPC_METHODS, INTENT_ACTIONS };
```

## Usage example

```typescript
import { connectToSphere, RPC_METHODS } from './lib/sphere-client';

const client = await connectToSphere({
  url: 'ws://localhost:8765',
  dapp: { name: 'My Server', description: 'Backend service', url: 'https://myapp.com' },
});

console.log('Connected as:', client.identity.nametag);

// Query
const balance = await client.query(RPC_METHODS.GET_BALANCE);
console.log('Balance:', balance);

// Intent
await client.intent('send', { to: '@bob', amount: '50', coinId: '<lowercase 64-hex coin id>' });

// Events
client.on('transfer:incoming', (data) => console.log('Received:', data));

// Cleanup
await client.disconnect();
```

## safeSend pattern

When working with raw WebSocket connections (e.g., custom transport wrappers or direct `ws` usage), always guard against writes to a closing or closed socket. The template above patches `ws.send` automatically, but if you manage the WebSocket yourself:

```typescript
const safeSend = (data: string) => {
  if (ws.readyState === WebSocket.OPEN) ws.send(data);
};
```

This prevents `"WebSocket is not open"` errors that commonly occur during disconnect races or when the remote host drops the connection.
