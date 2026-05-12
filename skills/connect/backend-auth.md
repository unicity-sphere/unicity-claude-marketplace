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
import { verifySphereAuth, AuthVerificationError } from '@unicitylabs/sphere-sdk';

const challenges = new Map<string, { challenge: string; expiresAt: number }>();

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
    challenges.delete(chainPubkey); // one-time use

    try {
      // verifySphereAuth combines signature verification AND directAddress
      // derivation into a single misuse-resistant call. Notice it takes no
      // `directAddress` parameter — that's by design, see Security notes below.
      const { chainPubkey: pk, directAddress } = await verifySphereAuth({
        challenge: stored.challenge,
        signature,
        chainPubkey,
      });

      // Issue JWT — `pk` and `directAddress` are both safe identifiers.
      // The example below uses `pk` (chainPubkey) as the JWT subject because
      // it's directly self-authenticating via signature, but you can use
      // `directAddress` instead if your domain prefers it.
      const token = server.jwt.sign({ sub: pk, addr: directAddress }, { expiresIn: '24h' });
      return { token };
    } catch (err) {
      if (err instanceof AuthVerificationError) {
        return reply.status(401).send({ error: err.message, code: err.code });
      }
      throw err;
    }
  });
}
```

Install the SDK (already a dependency if you're using `@unicitylabs/sphere-sdk` elsewhere — `verifySphereAuth` is included):
```bash
npm install @unicitylabs/sphere-sdk
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

- **`verifySphereAuth` is the canonical primitive for backend wallet auth.** Don't reimplement signature verification yourself, and don't manually compose `verifySignedMessage` + `computeDirectAddressFromChainPubkey` unless you have a strong reason — the recipe handles ordering, format validation, and error categorization correctly.
- **Don't accept identifier claims you can derive.** `verifySphereAuth` deliberately has no `directAddress` parameter. If your design uses `directAddress` (or `l1Address`, or `nametag`) as a primary user identifier, never accept that value from the request body and use it directly as a database key — that's a **pre-bind hijack** (an attacker registers someone else's address under their own pubkey, locking the owner out forever). Let `verifySphereAuth` derive identifiers from the proven `chainPubkey` instead.
- **Challenge is single-use** — delete it after verify to prevent replay attacks.
- **Challenge expires in 5 minutes** — prevents stale signature reuse.
- **chainPubkey is included in the challenge text** — links the signature to a specific key.
- **Never store signatures** — only the JWT after successful verification.
- **Use `sessionStorage` for JWT** — cleared on tab close. Use `localStorage` only if you want persistence across sessions.
- The `sign_message` intent requires `sign:request` permission scope — request it during `autoConnect()`.

### Need just the address derivation?

For custom flows that don't fit the `verifySphereAuth` recipe (e.g., escrow services, batch verification, custom session models), use the primitive directly:

```typescript
import { computeDirectAddressFromChainPubkey } from '@unicitylabs/sphere-sdk';
const addr = await computeDirectAddressFromChainPubkey(chainPubkey);  // 'DIRECT://...'
```

The primitive is deterministic, requires no network round-trip, and produces the exact same address the wallet itself would compute for that pubkey.
