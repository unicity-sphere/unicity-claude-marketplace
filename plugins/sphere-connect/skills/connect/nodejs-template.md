# Node.js Template: WebSocket Client

This template creates a Sphere Connect client for Node.js applications using WebSocket transport.

## Dependencies

```bash
npm install @unicitylabs/sphere-sdk ws
```

## Template

```typescript
// src/lib/sphere-client.ts

import { ConnectClient, RPC_METHODS, INTENT_ACTIONS } from '@unicitylabs/sphere-sdk/connect';
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
 *   await client.intent('send', { recipient: '@alice', amount: '1000000', coinId: 'UCT' });
 *   await client.disconnect();
 */
export async function connectToSphere(config: SphereClientConfig): Promise<SphereClient> {
  const transport = WebSocketTransport.createClient({
    url: config.url,
    createWebSocket: (url: string) => new WebSocket(url) as any,
    autoReconnect: false,
  });

  await transport.connect();

  const client = new ConnectClient({ transport, dapp: config.dapp });
  const result = await client.connect();

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
await client.intent('send', { recipient: '@bob', amount: '500000', coinId: 'UCT' });

// Events
client.on('transfer:incoming', (data) => console.log('Received:', data));

// Cleanup
await client.disconnect();
```
