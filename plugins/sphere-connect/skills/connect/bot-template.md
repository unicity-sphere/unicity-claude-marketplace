# Bot Template: Own-Wallet Node Bot

This template creates a Node bot that runs **its own** Sphere wallet — its own keys, direct SDK
usage. There is **no Connect protocol involved anywhere in this template.**

> **This is not [nodejs-template.md](nodejs-template.md).** `nodejs-template.md` builds a Node
> **dApp** that connects *to* someone else's already-running wallet over Connect (`ConnectClient` +
> `WebSocketTransport`). This template builds a bot that **is** the wallet: it calls `Sphere.init()`
> directly and drives `sphere.payments` / `sphere.communications` itself. Use this template when the
> user asks to "build a bot", "give an agent its own wallet", or "run a wallet headlessly" — use
> `nodejs-template.md` when the user asks to "connect to a wallet" or "talk to Sphere over Connect".

## Dependencies

```bash
npm install @unicitylabs/sphere-sdk dotenv ws
npm install -D @types/ws   # TypeScript projects: `ws` ships no declarations
```

`ws` is a required dependency here too, even though this template never touches the Connect
protocol — the SDK's Node transport (Nostr relays) needs a WebSocket implementation on Node, and
`ws` is what provides it. Node.js **>= 22** is required.

## ⚠️ `WALLET_API_URL` is required (read this first)

The money side of the SDK is composed from a **wallet-api transport config**. `Sphere.init` without
it fails closed:

```
Sphere requires a wallet-api composition for money: pass `walletApi`
({ network, baseUrl, deviceId? } — createWalletApiProviders builds it) to Sphere.init.
```

— a `SphereError` with code `INVALID_CONFIG`, thrown before the bot ever runs. There is no
"send + DM only" mode to fall back to: without `walletApi` there is no payments vertical at all,
so nothing sends, nothing receives, and nothing reads a balance. Ask the user for a wallet-api base
URL up front and fail loudly when it is missing, exactly as the template does.

`createWalletApiProviders(base, config)` takes the already-built platform providers and returns them
with the `walletApi` config attached. Its `network` **must equal** the network `Sphere.init` runs on:
a payments composition is single-network and a mismatch is rejected with `INVALID_CONFIG`.

## Template

### `src/sphere.ts` — own-wallet init

```typescript
import 'dotenv/config';
import { Sphere, type Identity } from '@unicitylabs/sphere-sdk';
import { createNodeProviders } from '@unicitylabs/sphere-sdk/impl/nodejs';
import { createWalletApiProviders } from '@unicitylabs/sphere-sdk/impl/shared/wallet-api';

// `network` here is a plain string ('testnet2' | 'mainnet'), NOT the SPHERE_NETWORKS.testnet2
// OBJECT used by the Connect protocol (ConnectClient / autoConnect). Passing the object here
// would hand a string-typed field an object — this is a Node-SDK config value, not a Connect
// handshake field.
const NETWORK = (process.env.SPHERE_NETWORK ?? 'testnet2') as 'mainnet' | 'testnet2';

export interface BotSphere {
  sphere: Sphere;
  identity: Identity;
}

/**
 * Boot the bot's own Sphere wallet.
 *
 * - `AGGREGATOR_API_KEY` — aggregator key for the target network (required).
 *   On mainnet this key is a SECRET — never commit one.
 * - `WALLET_API_URL` — wallet-api base URL (required). Without it Sphere.init throws
 *   INVALID_CONFIG: the payments vertical is composed from this config.
 * - `SPHERE_NETWORK` — 'testnet2' (default) or 'mainnet'. Must match the wallet-api's network.
 * - `BOT_MNEMONIC` — persists the bot's identity across runs. Leave empty to
 *   auto-generate a fresh one (printed once — save it to persist).
 * - `BOT_DATA_DIR` — local file storage root for wallet data (default `./.bot-data`).
 * - `BOT_DEVICE_ID` — optional stable device id for the wallet-api session.
 */
export async function createBotSphere(): Promise<BotSphere> {
  const aggregatorApiKey = process.env.AGGREGATOR_API_KEY;
  if (!aggregatorApiKey) throw new Error('AGGREGATOR_API_KEY is required');

  const walletApiUrl = process.env.WALLET_API_URL;
  if (!walletApiUrl) {
    throw new Error('WALLET_API_URL is required — Sphere.init has no payments vertical without it');
  }

  const botMnemonic = process.env.BOT_MNEMONIC || undefined;
  const botDataDir = process.env.BOT_DATA_DIR || './.bot-data';

  const base = createNodeProviders({
    network: NETWORK,
    dataDir: botDataDir,
    oracle: { apiKey: aggregatorApiKey },
  });

  const providers = createWalletApiProviders(base, {
    baseUrl: walletApiUrl,
    network: NETWORK, // must equal the Sphere network below
    ...(process.env.BOT_DEVICE_ID ? { deviceId: process.env.BOT_DEVICE_ID } : {}),
  });

  const { sphere, generatedMnemonic } = await Sphere.init({
    ...providers,
    network: NETWORK,
    mnemonic: botMnemonic,
    autoGenerate: !botMnemonic,
  });

  if (generatedMnemonic) {
    console.log(
      '\n=== GENERATED A NEW BOT MNEMONIC ===\n' +
        generatedMnemonic +
        '\nSAVE THIS — set BOT_MNEMONIC to persist this identity across runs.\n' +
        '=====================================\n',
    );
  }

  const identity = sphere.identity;
  if (!identity) {
    throw new Error('Sphere.init succeeded but sphere.identity is null');
  }

  return { sphere, identity };
}
```

### `src/coins.ts` — base-unit + coinId helpers

`payments.send` takes an amount in **base units** (the smallest indivisible unit) as a string and a
canonical **lowercase 64-hex `coinId`**; `payments.mint` takes the same `coinId` and a **`bigint`**.
Neither takes a human decimal amount or a bare symbol. Convert at the edge, exactly, with string
arithmetic (never `Number`/`parseFloat` — that silently loses precision on high-decimal coins):

```typescript
import { TokenRegistry } from '@unicitylabs/sphere-sdk';

const HEX_COIN_ID_RE = /^[0-9a-f]{64,}$/i;

/** Convert a human decimal amount (e.g. "1.5") to an integer base-unit string. */
export function toBaseUnits(human: string, decimals: number): string {
  if (!/^\d+(\.\d+)?$/.test(human)) {
    throw new Error(`Invalid amount "${human}": expected a non-negative decimal string`);
  }
  const [intPart, fracPart = ''] = human.split('.');
  if (fracPart.length > decimals) {
    throw new Error(`Amount "${human}" has more fractional digits than the coin's ${decimals} decimals`);
  }
  const combined = intPart + fracPart.padEnd(decimals, '0');
  return combined.replace(/^0+(?=\d)/, '');
}

/** Resolve a coin symbol or hex coinId to { coinId, decimals } via the TokenRegistry singleton. */
export function resolveCoin(symbolOrId: string): { coinId: string; decimals: number } {
  const registry = TokenRegistry.getInstance();
  if (HEX_COIN_ID_RE.test(symbolOrId)) {
    const def = registry.getDefinition(symbolOrId);
    if (!def) throw new Error(`Unknown coinId "${symbolOrId}"`);
    return { coinId: symbolOrId, decimals: def.decimals ?? 0 };
  }
  const coinId = registry.getCoinIdBySymbol(symbolOrId);
  const def = registry.getDefinitionBySymbol(symbolOrId);
  if (!coinId || !def) throw new Error(`Unknown coin symbol "${symbolOrId}"`);
  return { coinId, decimals: def.decimals ?? 0 };
}
```

### `src/index.ts` — runtime: DM auto-reply, self-mint, money-safe send

```typescript
import { isPossiblyCommittedSendOutcome, type DirectMessage, type IncomingTransfer } from '@unicitylabs/sphere-sdk';
import { createBotSphere } from './sphere';
import { resolveCoin, toBaseUnits } from './coins';

async function main() {
  const { sphere, identity } = await createBotSphere();
  console.log('Bot identity:', identity);

  // --- DMs: reply to anything sent to the bot, skip self-DMs to avoid an echo loop ---
  sphere.communications.onDirectMessage(async (m: DirectMessage) => {
    if (m.senderPubkey === identity.chainPubkey) return;
    try {
      await sphere.communications.sendDM(m.senderPubkey, `echo: ${m.content}`);
    } catch (err) {
      console.error('DM reply failed:', err instanceof Error ? err.message : err);
    }
  });

  // --- Self-mint a starting float (best-effort — may be unavailable on some networks) ---
  try {
    const { coinId, decimals } = resolveCoin('UCT');
    const amount = BigInt(toBaseUnits('100', decimals)); // payments.mint takes a BIGINT, not a string
    const result = await sphere.payments.mint(coinId, amount);
    if (result.success) {
      console.log(`Self-minted 100 UCT (tokenId ${result.tokenId})`);
    } else {
      console.log(`Self-mint unavailable: ${result.error}`); // does NOT throw on ordinary failure
    }
  } catch (err) {
    console.error('Self-mint threw:', err instanceof Error ? err.message : err);
  }

  // --- Receive ---
  sphere.on('transfer:incoming', (t: IncomingTransfer) => {
    console.log(`Incoming transfer ${t.id} from ${t.senderNametag ?? t.senderPubkey}`);
  });

  // --- Balances (async: assets() reads through the wallet-api inventory) ---
  async function balances() {
    const assets = await sphere.payments.assets();
    for (const asset of assets) {
      console.log(asset.symbol, asset.totalAmount, `(confirmed ${asset.confirmedAmount})`);
    }
  }

  // --- Money-safe send ---
  async function sendTokens(to: string, humanAmount: string, symbolOrId: string) {
    const { coinId, decimals } = resolveCoin(symbolOrId);
    const amount = toBaseUnits(humanAmount, decimals);
    try {
      const result = await sphere.payments.send({ coinId, amount, recipient: to });
      if (result.deliveryPending) {
        console.log('sent, delivery pending'); // SUCCESS — source is spent, delivery will land later
      } else {
        console.log('Send result:', result);
      }
    } catch (err) {
      // Money safety: a possibly-committed outcome means the spend may already have
      // gone through. NEVER auto-resend — that spends a different source token and
      // can pay the recipient twice. Surface it to a human / reconciliation step.
      if (isPossiblyCommittedSendOutcome(err)) {
        console.log('spend may already be committed — do NOT retry');
        return;
      }
      console.error('Send failed:', err instanceof Error ? err.message : err);
    }
  }

  // ... wire sendTokens() / balances() up to whatever triggers this bot (stdin, DM commands, an API route)
}

main().catch((err) => {
  console.error('Bot failed to start:', err instanceof Error ? err.message : err);
  process.exit(1);
});
```

### `.env`

```bash
# Aggregator key for the target network (required). A mainnet key is a SECRET.
AGGREGATOR_API_KEY=
# Wallet-api base URL (REQUIRED — Sphere.init throws INVALID_CONFIG without it)
WALLET_API_URL=
# 'testnet2' (default) or 'mainnet'. Must match the wallet-api's network.
SPHERE_NETWORK=testnet2
# The bot's own wallet. Leave empty on first run to auto-generate (the bot
# prints the generated mnemonic — save it). Set it to persist identity.
BOT_MNEMONIC=
# Local file storage for the bot's wallet data
BOT_DATA_DIR=./.bot-data
# Optional stable device id for the wallet-api session
BOT_DEVICE_ID=
```

## Key API surface used here

| Call | Signature | Notes |
|------|-----------|-------|
| `createNodeProviders(config)` | `impl/nodejs` | `config.network` is a required plain string, throws `INVALID_CONFIG` if missing. Storage is `dataDir` only — there is **no `tokensDir`** |
| `createWalletApiProviders(base, config)` | `impl/shared/wallet-api` | `{ baseUrl, network, deviceId? }` → `base & { walletApi }`. **Required**: `Sphere.init` has no payments vertical without it |
| `Sphere.init(options)` | SDK root | `{ ...providers, mnemonic?, autoGenerate?, network? }` → `{ sphere, created, generatedMnemonic? }` |
| `sphere.identity` | getter | `Identity \| null` — `{ chainPubkey, directAddress?, ipnsName?, nametag? }` |
| `sphere.communications.onDirectMessage(handler)` | returns unsubscribe fn | `handler: (m: DirectMessage) => void` |
| `sphere.communications.sendDM(recipient, content)` | `Promise<DirectMessage>` | |
| `sphere.payments.assets(coinId?)` | `Promise<Asset[]>` | **Async.** Fields are `symbol` / `totalAmount` / `confirmedAmount` / `unconfirmedAmount` — there is no `amount` |
| `sphere.payments.tokens(filter?)` | synchronous | `Token[]` |
| `sphere.payments.mint(coinIdHex, amount)` | `Promise<MintResult>` | **`amount` is a `bigint`**, not a base-unit string — wrap `toBaseUnits()`'s string result in `BigInt(...)` |
| `sphere.payments.send(request)` | `Promise<TransferResult>` | `{ coinId, amount, recipient, memo? }`; `result.deliveryPending` true = success, never resend |
| `sphere.payments.pendingTransfers()` / `.resumeNow()` | `Promise<…>` | The retry surface. A retry button calls `resumeNow()` — **never** `send()` again |
| `sphere.on('transfer:incoming', handler)` | returns unsubscribe fn | |
| `isPossiblyCommittedSendOutcome(err)` | SDK root | Guards a `send` catch block — true means "do not retry" |
| `sphere.destroy()` | `Promise<void>` | Call on shutdown |

## DO NOT

- Do not pass `SPHERE_NETWORKS.testnet2` (the Connect network object) to `createNodeProviders` or
  `createWalletApiProviders` — both take the plain string `'testnet2'` / `'mainnet'`.
- Do not pass a base-unit string to `payments.mint` — it takes a `bigint`.
- Do not call `payments.mintFungibleToken` or `payments.getBalance` — both are gone. They are
  `payments.mint(coinId, bigint)` and `payments.assets()` (async) now.
- Do not pass `tokensDir` to `createNodeProviders` — the option does not exist; it is a type error.
- Do not reach for `createOwnStorageWalletApiProviders` — that export no longer exists.
  `createWalletApiProviders` is the one composition.
- Do not treat `WALLET_API_URL` as optional, and do not offer a "send + DM only" fallback — there
  is no payments vertical without it.
- Do not resend on a possibly-committed `send` failure — check `isPossiblyCommittedSendOutcome(err)`
  first and stop if it's true. To retry a stuck transfer, call `payments.resumeNow()`, never `send()`.
