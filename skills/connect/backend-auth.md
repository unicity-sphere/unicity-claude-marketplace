# Backend Authentication via Wallet Signature

How to authenticate users to a backend server using their Sphere wallet — challenge-response with **signature recovery** and **Nostr identity binding resolution**.

## Why signing is needed

`autoConnect()` gives the **frontend** proof of wallet ownership. The **backend** has no way to verify this — any HTTP request could claim any wallet address. The solution: the backend issues a random challenge, the wallet signs it, the backend **recovers** the signer's public key from the signature and looks up the corresponding identity on Nostr.

## Critical: don't trust any identifier from the request body

A naive backend reads `{ directAddress, chainPubkey, signature }` from the body and trusts the claimed addresses. This is broken — an attacker can sign with their own key while claiming someone else's `directAddress`. Past versions of this skill recommended an SDK helper (`verifySphereAuth` / `computeDirectAddressFromChainPubkey`) for this; both are **removed** because `directAddress` cannot in fact be derived from `chainPubkey` — the SDK derives it from `SHA256(privkey)`, which is private.

The correct flow:

```
Frontend          Wallet           Backend
   |                 |                |
   |── autoConnect ─►|                |
   |◄─ identity ─────|                |
   |                 |                |
   |── POST /auth/challenge ─────────►|  ← no body claims
   |◄── { nonce } ───────────────────|
   |                 |                |
   |── intent('sign_message', msg) ─►|
   |◄── { signature } ───────────────|
   |                 |                |
   |── POST /auth/verify ────────────►|
   |   { nonce, signature }           |  ← that's it
   |                                  |  recoverPubkey(challenge, sig)
   |                                  |  → chainPubkey
   |                                  |  sphere.resolve(chainPubkey)
   |                                  |  → { directAddress, nametag, ... }
   |◄── { token: JWT } ──────────────|
   |                 |                |
   |── GET /api/... (Bearer JWT) ────►|
```

The backend **never reads** `directAddress` or `chainPubkey` from the request — both come from cryptographic recovery and Nostr resolution respectively.

## Backend implementation (Fastify + TypeScript)

```typescript
// src/routes/auth.ts
import { randomBytes } from 'node:crypto';
import type { FastifyInstance } from 'fastify';
import { recoverPubkeyFromSignature } from '@unicitylabs/sphere-sdk';
import { Sphere } from '@unicitylabs/sphere-sdk';
import { createNodeProviders } from '@unicitylabs/sphere-sdk/impl/nodejs';

// Singleton Sphere instance used only for resolving — not for signing/sending.
// Initialize once at boot.
let sphere: Sphere;
export async function initSphere() {
  const providers = createNodeProviders({ network: 'testnet', dataDir: './.sphere-resolver' });
  const { sphere: instance } = await Sphere.init({ ...providers, autoGenerate: true });
  sphere = instance;
}

const CHALLENGE_TTL_MS = 5 * 60_000;
const nonces = new Map<string, { challenge: string; expiresAt: number }>();

export async function authRoutes(server: FastifyInstance) {
  // Step 1: Issue a challenge (no claims from the client)
  server.post('/auth/challenge', async (req) => {
    const nonce = randomBytes(16).toString('hex');
    const issuedAt = new Date().toISOString();
    const challenge = [
      'Sign in to My App',
      `Domain: ${req.headers.host ?? 'localhost'}`,
      `Nonce: ${nonce}`,
      `Issued At: ${issuedAt}`,
    ].join('\n');
    nonces.set(nonce, { challenge, expiresAt: Date.now() + CHALLENGE_TTL_MS });
    return { nonce };
  });

  // Step 2: Verify signature → resolve identity → issue JWT
  server.post<{ Body: { nonce: string; signature: string } }>(
    '/auth/verify',
    async (req, reply) => {
      const { nonce, signature } = req.body;

      const stored = nonces.get(nonce);
      if (!stored) return reply.status(401).send({ error: 'unknown_nonce' });
      if (Date.now() > stored.expiresAt) {
        nonces.delete(nonce);
        return reply.status(401).send({ error: 'nonce_expired' });
      }
      nonces.delete(nonce); // one-time use

      // 1. Recover the signer's pubkey from the signature — no claims needed.
      let chainPubkey: string;
      try {
        chainPubkey = recoverPubkeyFromSignature(stored.challenge, signature);
      } catch {
        return reply.status(401).send({ error: 'invalid_signature' });
      }

      // 2. Look up the Nostr-bound identity for that pubkey.
      const peer = await sphere.resolve(chainPubkey);
      if (!peer || !peer.directAddress) {
        // Nostr has no binding for this pubkey. The wallet must publish
        // an identity binding event before logging in. Surface this as a
        // dedicated error so the frontend can guide the user.
        return reply.status(401).send({ error: 'no_identity_binding' });
      }

      // 3. (Optional) Detect pre-bind hijacks — see Security notes below.

      // 4. Issue JWT with the trusted, resolved identity.
      const token = server.jwt.sign(
        { sub: chainPubkey, addr: peer.directAddress, name: peer.nametag ?? null },
        { expiresIn: '24h' },
      );
      return { token };
    },
  );
}
```

Install the SDK (any 0.7.2+ release):
```bash
npm install @unicitylabs/sphere-sdk
```

## Frontend implementation (React hook)

```typescript
// src/hooks/useWalletAuth.ts
import { useState, useCallback } from 'react';

const API_URL = import.meta.env.VITE_API_URL ?? '';

export function useWalletAuth() {
  const [token, setToken] = useState<string | null>(() => sessionStorage.getItem('wallet_token'));
  const [isAuthenticating, setIsAuthenticating] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const authenticate = useCallback(async (
    signMessage: (msg: string) => Promise<string>,
  ): Promise<string> => {
    setIsAuthenticating(true);
    setError(null);
    try {
      // 1. Ask backend for a nonce-bound challenge. Send no claims.
      const challengeRes = await fetch(`${API_URL}/auth/challenge`, { method: 'POST' });
      const { nonce } = await challengeRes.json();

      // 2. Reconstruct the canonical challenge text. It must byte-match
      //    what the backend stored. Keep this string in sync with the
      //    server format. The wallet sees the exact text it signs.
      const challenge = [
        'Sign in to My App',
        `Domain: ${window.location.host}`,
        `Nonce: ${nonce}`,
        // Note: Issued At is the only field that differs — we let the
        // server inject it and don't include it client-side. Use server
        // echo if you need to display the exact signed text.
      ].join('\n');

      // 3. Sign via the wallet (sign_message intent).
      const signature = await signMessage(challenge);

      // 4. Send only { nonce, signature } — never the address or pubkey.
      const verifyRes = await fetch(`${API_URL}/auth/verify`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ nonce, signature }),
      });
      if (!verifyRes.ok) {
        const { error: code } = await verifyRes.json();
        throw new Error(code);
      }
      const { token } = await verifyRes.json();

      sessionStorage.setItem('wallet_token', token);
      setToken(token);
      return token;
    } catch (err) {
      const message = err instanceof Error ? err.message : 'auth_failed';
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

> **Practical tip:** to keep the challenge text in sync between client and server, have the backend return the full challenge string instead of just the nonce. The client signs whatever the server sends. This avoids per-deploy format drift.

## Wiring it together in a component

```tsx
import { useWalletConnect } from './hooks/useWalletConnect';
import { useWalletAuth } from './hooks/useWalletAuth';

function App() {
  const wallet = useWalletConnect();
  const auth = useWalletAuth();

  const handleLogin = async () => {
    if (!wallet.isConnected) return;
    await auth.authenticate(
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
        {auth.error === 'no_identity_binding' && (
          <p style={{ color: 'orange' }}>
            Wallet has no public identity binding yet. Register a nametag in the wallet
            (which publishes the binding to Nostr), then retry.
          </p>
        )}
        {auth.error && auth.error !== 'no_identity_binding' && (
          <p style={{ color: 'red' }}>{auth.error}</p>
        )}
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
```

## Protecting routes on the backend (Fastify)

```typescript
server.addHook('onRequest', async (req, reply) => {
  try {
    await req.jwtVerify();
  } catch {
    reply.status(401).send({ error: 'Unauthorized' });
  }
});

server.get('/api/profile', async (req) => {
  const { sub: chainPubkey, addr, name } = req.user as {
    sub: string; addr: string; name: string | null;
  };
  return { chainPubkey, directAddress: addr, nametag: name };
});
```

## Security notes

### Pre-bind hijack — what to watch for

Even with signature recovery + Nostr resolution, you must still defend against the case where a record was created under a hijacked address earlier (before this auth flow was deployed, or because of a different broken endpoint). When a verified signer logs in, compare the resolved identity against your stored user record:

```typescript
const user = await db.users.findOne({ directAddress: peer.directAddress });

if (user && user.chainPubkey !== chainPubkey) {
  // The stored pubkey for this directAddress disagrees with the one
  // that just signed. The record is suspect — refuse to issue a JWT
  // and emit an admin alert. Do NOT auto-rewrite the user record.
  await emitAdminAlert({
    type: 'auth.login_blocked_pubkey_mismatch',
    severity: 'warning',
    subject: { type: 'wallet', identifier: peer.directAddress },
    context: { recoveredChainPubkey: chainPubkey, storedChainPubkey: user.chainPubkey },
  });
  return reply.status(401).send({ error: 'identity_mismatch' });
}
```

Also scan for foreign records claimed under the same `chainPubkey`:

```typescript
const foreign = await db.users.find({
  chainPubkey,
  walletAddress: { $ne: peer.directAddress },
}).toArray();
for (const hijacked of foreign) {
  await emitAdminAlert({
    type: 'auth.pre_bind_hijack_detected',
    severity: 'critical',
    subject: { type: 'wallet', identifier: hijacked.walletAddress },
    context: { attackerChainPubkey: chainPubkey, victimWalletAddress: hijacked.walletAddress },
  });
}
```

The reference implementation of this pattern lives in [`sphere-api/src/services/auth.service.ts`](https://github.com/unicity-sphere/sphere-api/blob/main/src/services/auth.service.ts).

### General

- **Never derive `directAddress` from `chainPubkey` on the backend.** It cannot be done — the SDK derives `directAddress` from `SHA256(privkey)`, not from `chainPubkey`. Always resolve via `sphere.resolve(chainPubkey)` (Nostr binding event).
- **Body-claimed identifiers are advisory at best.** Old API consumers may still send `directAddress` / `chainPubkey` in the body; accept them only for backward-compat parsing and ignore them when establishing identity.
- **Challenge is single-use** — delete after verify.
- **Challenge expires in 5 minutes** — prevents stale signature reuse.
- **Bind the nonce to the request, not to the user.** Don't key the nonce store by `chainPubkey` — that lets an attacker overwrite a victim's pending nonce. Key by the nonce itself.
- **Use `sessionStorage` for JWT** — cleared on tab close. Use `localStorage` only if you want persistence.
- **Never store signatures** — only the JWT after successful verification.
- The `sign_message` intent requires `sign:request` permission scope — request it during `autoConnect()`.

### What if Nostr resolution fails?

`sphere.resolve()` can return `null` if:
- The wallet has never published an identity binding event (i.e. has no nametag and has not done a full key-binding publish).
- The configured Nostr relays are unreachable.

For login UX, return a specific `no_identity_binding` error so the frontend can suggest registering a nametag (which publishes the binding). For ops, emit an admin alert (e.g. `type: 'auth.no_binding'`, severity `info`) so you can see how often it happens and decide whether to relax the policy for specific cases.
