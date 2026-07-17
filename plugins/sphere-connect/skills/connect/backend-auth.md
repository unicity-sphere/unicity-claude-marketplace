# Backend Authentication via Wallet Signature

How to authenticate users to a backend server using their Sphere wallet — challenge-response with **signature recovery**. The core flow needs nothing but a crypto helper: no `Sphere` instance, no storage, no network call, on the verifying side.

## Why signing is needed

`autoConnect()` gives the **frontend** proof of wallet ownership. The **backend** has no way to verify this — any HTTP request could claim any wallet address. The solution: the backend issues a random challenge, the wallet signs it, the backend **recovers** the signer's public key from the signature. That recovered key — not anything the client asserted — is the identity.

## Critical: don't trust any identifier the client sends you

A naive backend reads `{ directAddress, chainPubkey, signature }` from the body and trusts the claimed addresses. This is broken — an attacker can sign with their own key while claiming someone else's `directAddress`. Trust nothing self-reported: **recover** the public key from the signature instead.

This includes the `sign_message` intent's own result. Its shape is `{ signature: string, publicKey: string }` — but `publicKey` is simply what the wallet *says* signed the message. **Never read it.** The frontend should discard it entirely; the backend should never even receive it (send only `{ nonce, signature }` to `/verify`). The only trustworthy identity is what `recoverPubkeyFromSignature` independently computes from the signature.

**Key the user on the recovered `chainPubkey`, not on `directAddress`.** Since state-transition v2 the chain public key is the identity: it is what the signature recovers to, and it is what the wallet itself now shows users (the `DIRECT://` address is deliberately hidden across the Sphere UI). A `directAddress` can be obtained, but building on it is the wrong path for the core auth decision — it adds a derivation you do not need and cannot verify from a signature alone. Past versions of this skill recommended the SDK helpers `verifySphereAuth` / `computeDirectAddressFromChainPubkey`; both have since been **removed from the SDK**. Use `recoverPubkeyFromSignature` and treat the recovered `chainPubkey` as the account key.

## The byte-exact-challenge rule (read this before writing challenge code)

`recoverPubkeyFromSignature(message, signature)` mathematically recovers *some* valid public key from any well-formed signature — for the **exact** message that was signed. If the backend reconstructs even a slightly different string on `/verify` than the one the wallet actually signed on `/challenge` (different whitespace, a different line ending, fields reordered, a re-serialized timestamp), recovery does **not** fail — it silently returns a **different, still valid-looking pubkey**. There is no exception, no mismatch error: the backend just authenticates the wrong identity.

The only reliable defense: **exactly one function assembles challenge text**, and both the issuing step and the verifying step call it.

```typescript
// src/challenge.ts — the single formatter. Nowhere else assembles challenge text.
export interface ChallengeRecord {
  chainPubkey: string;
  domain: string;
  nonce: string;
  issuedAt: number;  // epoch ms
  expiresAt: number; // epoch ms
}

function formatChallenge(fields: ChallengeRecord): string {
  return [
    'Sign in to My App',
    '',
    `Domain: ${fields.domain}`,
    `Chain Pubkey: ${fields.chainPubkey}`,
    `Nonce: ${fields.nonce}`,
    `Issued At: ${new Date(fields.issuedAt).toISOString()}`,
    `Expiration Time: ${new Date(fields.expiresAt).toISOString()}`,
  ].join('\n');
}

/** Called by /challenge with freshly-generated fields (now/nonce injected by the caller —
 *  never Date.now()/Math.random() inside here — keeps this pure and unit-testable). */
export function buildChallenge(params: { chainPubkey: string; domain: string; now: number; nonce: string; ttlSeconds: number }) {
  const { chainPubkey, domain, now, nonce, ttlSeconds } = params;
  const issuedAt = now;
  const expiresAt = now + ttlSeconds * 1000;
  return { challenge: formatChallenge({ chainPubkey, domain, nonce, issuedAt, expiresAt }), nonce, issuedAt, expiresAt };
}

/** Called by /verify with the EXACT fields stored for the nonce — never re-derived
 *  from anything in the request body. Must funnel through formatChallenge too. */
export function reconstructChallenge(stored: ChallengeRecord): string {
  return formatChallenge(stored);
}
```

Store the fields that went into the challenge (`chainPubkey`, `domain`, `nonce`, `issuedAt`, `expiresAt`) keyed by nonce, and reconstruct from those stored fields on `/verify` — never from anything the request body claims. On the frontend, sign the `challenge` string returned by `/challenge` **verbatim** — no trimming, re-encoding, or reformatting before it goes into the `sign_message` intent.

## The flow

```
Frontend                        Wallet              Backend
   |                               |                    |
   |── autoConnect ───────────────►|                    |
   |◄─ identity ────────────────────                    |
   |                               |                    |
   |── GET /challenge?chainPubkey=<hint> ───────────────►|  binds challenge text; NOT a trust input
   |◄── { nonce, challenge } ─────────────────────────────|  built via buildChallenge()
   |                               |                    |
   |── intent(sign_message, { message: challenge }) ────►|
   |◄── { signature, publicKey } ─────────────────────────  publicKey is IGNORED — never sent onward
   |                               |                    |
   |── POST /verify { nonce, signature } ────────────────►|  reconstructChallenge(stored) [same formatter]
   |                               |                    |  recoverPubkeyFromSignature(challenge, sig)
   |                               |                    |  → chainPubkey  (no Sphere instance needed)
   |◄── { jwt, chainPubkey } ─────────────────────────────|  session keyed on the RECOVERED pubkey
```

The backend **never reads** an identity claim from the request body anywhere in this flow — the only identity value that exists is the one `recoverPubkeyFromSignature` computes.

## Backend implementation (Express — a Fastify server follows the identical `/challenge` + `/verify` shape)

Verifying a signature needs only the two SDK crypto helpers below — **no `Sphere` instance, no storage, no network I/O**. This is the complete, self-sufficient core of the flow:

```typescript
// src/server.ts
import { randomBytes } from 'node:crypto';
import express from 'express';
import { recoverPubkeyFromSignature, verifySignedMessage } from '@unicitylabs/sphere-sdk';
import { buildChallenge, reconstructChallenge, type ChallengeRecord } from './challenge';

const app = express();
app.use(express.json());

const CHALLENGE_TTL_SECONDS = 5 * 60;
const AUTH_DOMAIN = process.env.AUTH_DOMAIN ?? 'localhost';

// Single-use, expiring nonce store. In production this is Redis or a TTL-indexed
// DB collection shared across instances — but keeps these same two properties:
// (1) deleted the instant /verify consumes it, so a captured (nonce, signature)
// can never be replayed; (2) rejected once past its own expiresAt even if present.
const nonceStore = new Map<string, ChallengeRecord>();

app.get('/challenge', (req, res) => {
  const chainPubkey = req.query.chainPubkey as string; // shape-check: 66-char lowercase hex
  const nonce = randomBytes(16).toString('hex');
  const built = buildChallenge({ chainPubkey, domain: AUTH_DOMAIN, now: Date.now(), nonce, ttlSeconds: CHALLENGE_TTL_SECONDS });
  nonceStore.set(nonce, { chainPubkey, domain: AUTH_DOMAIN, nonce: built.nonce, issuedAt: built.issuedAt, expiresAt: built.expiresAt });
  res.json({ nonce: built.nonce, challenge: built.challenge, expiresAt: built.expiresAt });
});

app.post('/verify', (req, res) => {
  const { nonce, signature } = req.body ?? {};

  // Single-use: look up AND delete before doing anything else with it.
  const stored = nonceStore.get(nonce);
  nonceStore.delete(nonce);
  if (!stored) return res.status(401).json({ error: 'unknown_or_used_nonce' });
  if (Date.now() > stored.expiresAt) return res.status(401).json({ error: 'challenge_expired' });

  // Reconstruct the EXACT string the wallet signed — same formatter as /challenge.
  const challenge = reconstructChallenge(stored);

  let recovered: string;
  try {
    recovered = recoverPubkeyFromSignature(challenge, signature);
  } catch (err) {
    return res.status(401).json({ error: 'malformed_signature' });
  }

  // Cheap sanity re-check via the independent verifySignedMessage path — recovery
  // already guarantees this mathematically, but this catches any future drift
  // between the two SDK implementations at near-zero cost.
  if (!verifySignedMessage(challenge, signature, recovered)) {
    return res.status(401).json({ error: 'signature_verification_failed' });
  }

  // Identity is keyed ONLY on `recovered` — never on stored.chainPubkey (the hint
  // the frontend supplied to bind the challenge) and never on anything else the
  // client sent. This recovery step is the entire proof.
  const token = signJwt({ sub: recovered }, { expiresIn: '2h' });
  res.json({ jwt: token, chainPubkey: recovered });
});
```

Install the SDK (any 0.7.2+ release):
```bash
npm install @unicitylabs/sphere-sdk
```

### Optional production step: resolving a `directAddress`

`sphere-api` (this plugin's production reference) goes one step further: after recovering `chainPubkey`, it optionally resolves that pubkey to a `directAddress` / nametag via `sphere.resolve(chainPubkey)` (a Nostr binding lookup), and may key user-facing display or lookups on the resolved address. This step is **not required for the core auth decision** — it needs a full `Sphere.init()` (storage, transport, oracle providers), so only add it if the app actually needs a resolved address/nametag, not as a prerequisite for verifying identity:

```typescript
import { Sphere } from '@unicitylabs/sphere-sdk';
import { createNodeProviders } from '@unicitylabs/sphere-sdk/impl/nodejs';

// Boot once at startup — only needed if you use sphere.resolve() below.
let sphere: Sphere;
export async function initResolverSphere() {
  const providers = createNodeProviders({ network: 'testnet2', dataDir: './.sphere-resolver', oracle: { apiKey: process.env.AGGREGATOR_API_KEY! } });
  const { sphere: instance } = await Sphere.init({ ...providers, autoGenerate: true });
  sphere = instance;
}

// Call AFTER recovery succeeds, keyed on the already-recovered chainPubkey —
// never as a substitute for the recovery step itself.
const peer = await sphere.resolve(recovered);
if (peer?.directAddress) {
  // Attach for display/lookup purposes. The session/JWT `sub` stays `recovered`.
}
```

`sphere.resolve()` returns `null` when the wallet has never published an identity binding event (no nametag, no key-binding publish) or the configured Nostr relays are unreachable — treat that as "no resolved address yet", not as an auth failure; the user is still authenticated on `recovered`.

## Frontend implementation (React + `autoConnect`)

```typescript
// src/hooks/useWalletAuth.ts
import { useCallback, useState } from 'react';
import { INTENT_ACTIONS } from '@unicitylabs/sphere-sdk/connect';

const API_URL = import.meta.env.VITE_API_URL ?? '';

interface SignMessageResult {
  signature: string;
  publicKey: string; // self-reported by the wallet — NEVER trust or forward this
}

export function useWalletAuth() {
  const [token, setToken] = useState<string | null>(() => sessionStorage.getItem('wallet_token'));
  const [isAuthenticating, setIsAuthenticating] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // `client` and `chainPubkey` come from your autoConnect()/useWalletConnect() result —
  // see react-template.md. `chainPubkey` is only a binding hint for the challenge text;
  // the backend independently recovers the real signer.
  const authenticate = useCallback(async (client: { intent: <T>(a: string, p: Record<string, unknown>) => Promise<T> }, chainPubkey: string) => {
    setIsAuthenticating(true);
    setError(null);
    try {
      // 1. Ask backend for a nonce-bound challenge, binding it to this wallet as a hint.
      const { nonce, challenge } = await fetch(`${API_URL}/challenge?chainPubkey=${encodeURIComponent(chainPubkey)}`).then((r) => r.json());

      // 2. Sign the challenge text VERBATIM — no trim/re-encode. Byte-exactness is the
      //    whole safety property the backend relies on (see "byte-exact-challenge" above).
      const { signature } = await client.intent<SignMessageResult>(INTENT_ACTIONS.SIGN_MESSAGE, { message: challenge });
      // `publicKey` on the result is intentionally discarded here — never read it.

      // 3. Send only { nonce, signature } — never an address, pubkey, or the publicKey field.
      const verifyRes = await fetch(`${API_URL}/verify`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ nonce, signature }),
      });
      if (!verifyRes.ok) throw new Error((await verifyRes.json()).error ?? 'auth_failed');
      const { jwt } = await verifyRes.json();

      sessionStorage.setItem('wallet_token', jwt);
      setToken(jwt);
      return jwt;
    } catch (err) {
      setError(err instanceof Error ? err.message : 'auth_failed');
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

The `sign_message` intent requires the `sign:request` permission scope — request it (along with `identity:read`) during `autoConnect()`.

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

## Protecting routes on the backend

```typescript
function requireAuth(req: express.Request, res: express.Response, next: express.NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) return res.status(401).json({ error: 'unauthorized' });
  try {
    const { sub: chainPubkey } = jwt.verify(header.slice(7), JWT_SECRET) as { sub: string };
    (req as any).chainPubkey = chainPubkey; // the recovered pubkey — the only trusted identity
    next();
  } catch {
    res.status(401).json({ error: 'unauthorized' });
  }
}
```

(A Fastify server does the same check with `req.jwtVerify()` in an `onRequest` hook — same identity, `req.user.sub`.)

## Security notes

### Pre-bind hijack — what to watch for (only relevant if you added the optional `directAddress` resolution step)

Even with signature recovery, if your app also resolves and stores `directAddress`, defend against the case where a record was created under a hijacked address earlier (before this auth flow was deployed, or via a different broken endpoint). When a verified signer resolves, compare against your stored user record:

```typescript
const user = await db.users.findOne({ directAddress: peer.directAddress });

if (user && user.chainPubkey !== recovered) {
  // The stored pubkey for this directAddress disagrees with the one that just
  // signed. The record is suspect — refuse to issue a JWT and emit an admin
  // alert. Do NOT auto-rewrite the user record.
  await emitAdminAlert({
    type: 'auth.login_blocked_pubkey_mismatch',
    severity: 'warning',
    subject: { type: 'wallet', identifier: peer.directAddress },
    context: { recoveredChainPubkey: recovered, storedChainPubkey: user.chainPubkey },
  });
  return reply.status(401).send({ error: 'identity_mismatch' });
}
```

Also scan for foreign records claimed under the same `chainPubkey`:

```typescript
const foreign = await db.users.find({ chainPubkey: recovered, walletAddress: { $ne: peer.directAddress } }).toArray();
for (const hijacked of foreign) {
  await emitAdminAlert({
    type: 'auth.pre_bind_hijack_detected',
    severity: 'critical',
    subject: { type: 'wallet', identifier: hijacked.walletAddress },
    context: { attackerChainPubkey: recovered, victimWalletAddress: hijacked.walletAddress },
  });
}
```

The reference implementation of this pattern lives in [`sphere-api/src/services/auth.service.ts`](https://github.com/unicity-sphere/sphere-api/blob/main/src/services/auth.service.ts).

### General

- **Verification needs only `recoverPubkeyFromSignature` (+ `verifySignedMessage` as a sanity re-check).** No `Sphere` instance, no storage, no network call is required to decide who signed in — only to optionally resolve a display address afterward.
- **Byte-exact challenge text is the whole safety property.** One formatter function, called by both the issuing and verifying steps, fed only from stored fields on `/verify` — never re-derived from the request. See "The byte-exact-challenge rule" above.
- **Never trust the `sign_message` intent's self-reported `publicKey`.** It is discarded on the frontend and never sent to the backend; only `recoverPubkeyFromSignature`'s output is trusted.
- **Key identity on the recovered `chainPubkey`, never on `directAddress`.** `directAddress` is not mathematically derivable from `chainPubkey` — the SDK derives it from `SHA256(privkey)`, which is private. It can only be *looked up*, via the optional `sphere.resolve(chainPubkey)` step. Resolve it only when you actually need to display it — never as the account key.
- **Body-claimed identifiers are advisory at best.** Old API consumers may still send `directAddress` / `chainPubkey` in the body; accept them only for backward-compat parsing and ignore them when establishing identity.
- **Challenge is single-use** — look up and delete atomically before verifying.
- **Challenge expires in ~5 minutes** — prevents stale signature reuse.
- **Bind the nonce to the request, not to the user.** Don't key the nonce store by `chainPubkey` — that lets an attacker overwrite a victim's pending nonce. Key by the nonce itself.
- **Use `sessionStorage` for the JWT** — cleared on tab close. Use `localStorage` only if you want persistence.
- **Never store signatures** — only the JWT after successful verification.
