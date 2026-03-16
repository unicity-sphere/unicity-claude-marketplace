# Backend Authentication via Wallet Signature

How to authenticate users to a backend server using their Sphere wallet — challenge-response pattern.

## Why signing is needed

`autoConnect()` gives the **frontend** proof of wallet ownership. The **backend** has no way to verify this — any HTTP request could claim any wallet address. The solution: the backend issues a random challenge, the wallet signs it, the backend verifies the signature and issues a JWT.

```
Frontend          Wallet           Backend
   |                 |                |
   |── autoConnect ─►|                |
   |◄─ identity ─────|                |
   |                 |                |
   |── POST /auth/challenge ─────────►|
   |◄── { challenge } ───────────────|
   |                 |                |
   |── intent('sign_message') ───────►|
   |◄── { signature } ───────────────|
   |                 |                |
   |── POST /auth/verify ────────────►|
   |   { chainPubkey, challenge,      |
   |     signature }                  |
   |◄── { token: JWT } ──────────────|
   |                 |                |
   |── GET /api/... (Bearer JWT) ────►|
```

## Backend implementation (Fastify + TypeScript)

```typescript
// src/routes/auth.ts
import { randomBytes } from 'crypto';
import { FastifyInstance } from 'fastify';
import { secp256k1 } from '@noble/curves/secp256k1';
import { sha256 } from '@noble/hashes/sha256';

const SIGN_MESSAGE_PREFIX = 'Sphere Signed Message:\n';
const challenges = new Map<string, { challenge: string; expiresAt: number }>();

function hashSignMessage(message: string): Uint8Array {
  const prefixed = `${SIGN_MESSAGE_PREFIX}${message}`;
  const bytes = new TextEncoder().encode(prefixed);
  return sha256(sha256(bytes));
}

function verifySignedMessage(message: string, signature: string, chainPubkey: string): boolean {
  try {
    const msgHash = hashSignMessage(message);
    const sigBytes = hexToBytes(signature);
    // Recoverable signature: first byte is v (31 or 32), then r (32), s (32)
    const v = sigBytes[0] - 31;
    const rs = sigBytes.slice(1);
    const sig = secp256k1.Signature.fromCompact(bytesToHex(rs)).addRecoveryBit(v);
    const recovered = sig.recoverPublicKey(msgHash);
    return recovered.toHex(true) === chainPubkey;
  } catch {
    return false;
  }
}

function hexToBytes(hex: string): Uint8Array {
  const bytes = new Uint8Array(hex.length / 2);
  for (let i = 0; i < bytes.length; i++) {
    bytes[i] = parseInt(hex.slice(i * 2, i * 2 + 2), 16);
  }
  return bytes;
}

function bytesToHex(bytes: Uint8Array): string {
  return Array.from(bytes).map(b => b.toString(16).padStart(2, '0')).join('');
}

export async function authRoutes(server: FastifyInstance) {
  // Step 1: Get challenge
  server.post<{ Body: { chainPubkey: string } }>('/auth/challenge', async (req, reply) => {
    const { chainPubkey } = req.body;
    if (!chainPubkey) return reply.status(400).send({ error: 'chainPubkey required' });

    const nonce = randomBytes(16).toString('hex');
    const challenge = [
      'Sign in to My App',
      `Chain Public Key: ${chainPubkey}`,
      `Nonce: ${nonce}`,
      `Issued At: ${new Date().toISOString()}`,
    ].join('\n');

    // Store with 5-minute expiry
    challenges.set(chainPubkey, { challenge, expiresAt: Date.now() + 5 * 60_000 });

    return { challenge };
  });

  // Step 2: Verify signature → issue JWT
  server.post<{
    Body: { chainPubkey: string; signature: string };
  }>('/auth/verify', async (req, reply) => {
    const { chainPubkey, signature } = req.body;

    const stored = challenges.get(chainPubkey);
    if (!stored) return reply.status(401).send({ error: 'No challenge found — request a new one' });
    if (Date.now() > stored.expiresAt) {
      challenges.delete(chainPubkey);
      return reply.status(401).send({ error: 'Challenge expired' });
    }

    const valid = verifySignedMessage(stored.challenge, signature, chainPubkey);
    challenges.delete(chainPubkey); // one-time use

    if (!valid) return reply.status(401).send({ error: 'Invalid signature' });

    // Issue JWT — sub = chainPubkey (wallet identifier)
    const token = server.jwt.sign({ sub: chainPubkey }, { expiresIn: '24h' });
    return { token };
  });
}
```

Install verification dependency:
```bash
npm install @noble/curves @noble/hashes
```

## Frontend implementation (React hook)

```typescript
// src/hooks/useWalletAuth.ts
import { useState, useCallback } from 'react';

const API_URL = import.meta.env.VITE_API_URL ?? '';

export interface WalletAuthState {
  token: string | null;
  isAuthenticating: boolean;
  error: string | null;
  authenticate: (chainPubkey: string, signMessage: (msg: string) => Promise<string>) => Promise<string>;
  logout: () => void;
}

export function useWalletAuth(): WalletAuthState {
  const [token, setToken] = useState<string | null>(() => sessionStorage.getItem('wallet_token'));
  const [isAuthenticating, setIsAuthenticating] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const authenticate = useCallback(async (
    chainPubkey: string,
    signMessage: (msg: string) => Promise<string>,
  ): Promise<string> => {
    setIsAuthenticating(true);
    setError(null);
    try {
      // 1. Get challenge
      const challengeRes = await fetch(`${API_URL}/auth/challenge`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ chainPubkey }),
      });
      const { challenge } = await challengeRes.json();

      // 2. Sign challenge with wallet
      const signature = await signMessage(challenge);

      // 3. Verify signature → get JWT
      const verifyRes = await fetch(`${API_URL}/auth/verify`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ chainPubkey, signature }),
      });
      if (!verifyRes.ok) throw new Error((await verifyRes.json()).error);
      const { token } = await verifyRes.json();

      sessionStorage.setItem('wallet_token', token);
      setToken(token);
      return token;
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Authentication failed';
      setError(message);
      throw err;
    } finally {
      setIsAuthenticating(false);
    }
  }, []);

  const logout = useCallback(() => {
    sessionStorage.removeItem('wallet_token');
    setToken(null);
  }, []);

  return { token, isAuthenticating, error, authenticate, logout };
}
```

## Wiring it together in a component

```tsx
import { useWalletConnect } from './hooks/useWalletConnect';
import { useWalletAuth } from './hooks/useWalletAuth';

function App() {
  const wallet = useWalletConnect();
  const auth = useWalletAuth();

  const handleLogin = async () => {
    if (!wallet.isConnected || !wallet.identity?.chainPubkey) return;

    await auth.authenticate(
      wallet.identity.chainPubkey,
      // signMessage: calls sign_message intent on wallet
      (message) => wallet.intent('sign_message', { message }) as Promise<string>,
    );
  };

  if (wallet.isAutoConnecting) return <div>Connecting...</div>;

  if (!wallet.isConnected) return <button onClick={wallet.connect}>Connect Wallet</button>;

  if (!auth.token) {
    return (
      <div>
        <p>Connected: {wallet.identity?.nametag}</p>
        <button onClick={handleLogin} disabled={auth.isAuthenticating}>
          {auth.isAuthenticating ? 'Signing...' : 'Sign In'}
        </button>
        {auth.error && <p style={{ color: 'red' }}>{auth.error}</p>}
      </div>
    );
  }

  return <div>Authenticated ✓ — JWT stored in sessionStorage</div>;
}
```

## Making authenticated API requests

```typescript
// src/lib/api.ts
const API_URL = import.meta.env.VITE_API_URL ?? '';

export async function apiFetch<T>(path: string, init?: RequestInit): Promise<T> {
  const token = sessionStorage.getItem('wallet_token');
  const res = await fetch(`${API_URL}${path}`, {
    ...init,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...init?.headers,
    },
  });
  if (!res.ok) throw new Error(`${res.status} ${res.statusText}`);
  return res.json();
}

// Usage:
const quests = await apiFetch<Quest[]>('/api/quests');
```

## Protecting routes on the backend (Fastify)

```typescript
// Authenticate middleware — add to protected routes
server.addHook('onRequest', async (req, reply) => {
  try {
    await req.jwtVerify();
  } catch {
    reply.status(401).send({ error: 'Unauthorized' });
  }
});

// Access the wallet address in a route handler:
server.get('/api/profile', async (req) => {
  const { sub: chainPubkey } = req.user as { sub: string };
  return { chainPubkey };
});
```

## Security notes

- **Challenge is single-use** — delete after verify to prevent replay attacks
- **Challenge expires in 5 minutes** — prevents stale signature reuse
- **chainPubkey is included in the challenge message** — links signature to a specific key
- **Never store signatures** — only the JWT after successful verification
- **Use `sessionStorage` for JWT** — cleared on tab close; use `localStorage` only if you want persistence across sessions
- The `sign_message` intent requires `sign:request` permission scope — request it during `autoConnect()`
