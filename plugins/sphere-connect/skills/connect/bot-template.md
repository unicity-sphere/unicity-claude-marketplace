# Bot Template: Own-Wallet Node Bot

This template creates a Node bot that runs **its own** Sphere wallet — its own keys, its own local
token storage, direct SDK usage. There is **no Connect protocol involved anywhere in this
template.**

> **This is not [nodejs-template.md](nodejs-template.md).** `nodejs-template.md` builds a Node
> **dApp** that connects *to* someone else's already-running wallet over Connect (`ConnectClient` +
> `WebSocketTransport`). This template builds a bot that **is** the wallet: it calls `Sphere.init()`
> directly and drives `sphere.payments` / `sphere.communications` itself. Use this template when the
> user asks to "build a bot", "give an agent its own wallet", or "run a wallet headlessly" — use
> `nodejs-template.md` when the user asks to "connect to a wallet" or "talk to Sphere over Connect".

## Dependencies

```bash
npm install @unicitylabs/sphere-sdk dotenv ws
```

`ws` is a required dependency here too, even though this template never touches the Connect
protocol — the SDK's Node transport (Nostr relays) needs a WebSocket implementation on Node, and
`ws` is what provides it. Node.js **>= 22** is required.

## ⚠️ Receiving tokens: the wallet-api rail (read this first)

A bot built from `createNodeProviders()` alone can **send** L3 tokens and use DMs out of the box,
but it **cannot receive tokens sent from a hosted Sphere wallet** (e.g. `sphere.unicity.network`).
The hosted wallet delivers sends through the **wallet-api mailbox**, not over Nostr. If the bot
never composes that mailbox delivery rail, a transfer sent to it from the hosted wallet **silently
never arrives — no error, nothing to catch.**

To receive those transfers, compose in the wallet-api rail with `createOwnStorageWalletApiProviders`
(see the template below) and set a `WALLET_API_URL`. If the bot only needs to send + DM, this step
can be skipped — but always tell the user explicitly that skipping it means the bot cannot receive
hosted-wallet payments, and log that fact at boot the way the template does.

Do **not** reach for `createWalletApiProviders` (no `OwnStorage` prefix) for this — that is a
different preset ("External Custody") that **replaces** the bot's own `tokenStorage` with a thin
provider backed by the wallet-api server's encrypted archive. This template wants the opposite:
the bot keeps owning its keys and its local token storage, and only *adds* the mailbox delivery
rail on top. That's exactly what `createOwnStorageWalletApiProviders` does — it takes the
already-built `base` providers and returns a new bundle with `base.tokenStorage` untouched, plus
`delivery` (wallet-api mailbox, custody `'external'` — zero server inventory writes on claim) and
`walletApi` (the auth-session client) added.

## Template

### `src/sphere.ts` — own-wallet init

```typescript
import 'dotenv/config';
import { Sphere, type Identity } from '@unicitylabs/sphere-sdk';
import { createNodeProviders } from '@unicitylabs/sphere-sdk/impl/nodejs';
import { createOwnStorageWalletApiProviders } from '@unicitylabs/sphere-sdk/impl/shared/wallet-api';

// `network` here is a plain string ('testnet2'), NOT the SPHERE_NETWORKS.testnet2
// OBJECT used by the Connect protocol (ConnectClient / autoConnect). Passing the
// object here would hand a string-typed field an object — this is a Node-SDK
// config value, not a Connect handshake field.
const TESTNET2 = 'testnet2' as const;

export interface BotSphere {
  sphere: Sphere;
  identity: Identity;
  /** True when the wallet-api mailbox delivery rail was composed in (WALLET_API_URL set). */
  receivesPayments: boolean;
}

/**
 * Boot the bot's own Sphere wallet against testnet2.
 *
 * - `AGGREGATOR_API_KEY` — testnet2 aggregator key (required).
 * - `BOT_MNEMONIC` — persists the bot's identity across runs. Leave empty to
 *   auto-generate a fresh one (printed once — save it to persist).
 * - `BOT_DATA_DIR` — local file storage root for wallet + token data
 *   (default `./.bot-data`); split into `<dir>/wallet` and `<dir>/tokens`.
 * - `WALLET_API_URL` — optional. When set, composes the wallet-api mailbox
 *   delivery rail (own-storage custody preset) so the bot can RECEIVE tokens
 *   sent from a hosted Sphere wallet. Left unset, the bot can still send + DM
 *   over Nostr, just not receive mailbox-delivered payments (see warning above).
 */
export async function createBotSphere(): Promise<BotSphere> {
  const aggregatorApiKey = process.env.AGGREGATOR_API_KEY;
  if (!aggregatorApiKey) {
    throw new Error('AGGREGATOR_API_KEY is required');
  }
  const botMnemonic = process.env.BOT_MNEMONIC || undefined;
  const botDataDir = process.env.BOT_DATA_DIR || './.bot-data';
  const walletApiUrl = process.env.WALLET_API_URL || undefined;

  const base = createNodeProviders({
    network: TESTNET2,
    dataDir: `${botDataDir}/wallet`,
    tokensDir: `${botDataDir}/tokens`,
    oracle: { apiKey: aggregatorApiKey },
  });

  const receivesPayments = Boolean(walletApiUrl);
  const providers = walletApiUrl
    ? createOwnStorageWalletApiProviders(base, { baseUrl: walletApiUrl, network: TESTNET2 })
    : base;

  const { sphere, generatedMnemonic } = await Sphere.init({
    ...providers,
    network: TESTNET2,
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

  return { sphere, identity, receivesPayments };
}
```

### `src/coins.ts` — base-unit + coinId helpers

`payments.send` and `payments.mintFungibleToken` both take amounts in **base units** (the smallest
indivisible unit) and a canonical **lowercase 64-hex `coinId`** — never a human decimal amount or a
bare symbol. Convert at the edge, exactly, with string arithmetic (never `Number`/`parseFloat` —
that silently loses precision on high-decimal coins):

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
  const { sphere, identity, receivesPayments } = await createBotSphere();
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
    const amount = BigInt(toBaseUnits('100', decimals)); // mintFungibleToken takes a BIGINT, not a string
    const result = await sphere.payments.mintFungibleToken(coinId, amount);
    if (result.success) {
      console.log(`Self-minted 100 UCT (tokenId ${result.tokenId})`);
    } else {
      console.log(`Self-mint unavailable: ${result.error}`); // does NOT throw on ordinary failure
    }
  } catch (err) {
    console.error('Self-mint threw:', err instanceof Error ? err.message : err);
  }

  // --- Receive: only fires if the wallet-api mailbox rail was composed in ---
  if (receivesPayments) {
    sphere.on('transfer:incoming', (t: IncomingTransfer) => {
      console.log(`Incoming transfer ${t.id} from ${t.senderNametag ?? t.senderPubkey}`);
    });
  } else {
    console.log('WALLET_API_URL not set — this bot will not receive tokens from the hosted wallet.');
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

  // ... wire sendTokens() up to whatever triggers this bot (stdin, DM commands, an API route)
}

main().catch((err) => {
  console.error('Bot failed to start:', err instanceof Error ? err.message : err);
  process.exit(1);
});
```

### `.env`

```bash
# testnet2 aggregator key (required)
AGGREGATOR_API_KEY=
# The bot's own wallet. Leave empty on first run to auto-generate (the bot
# prints the generated mnemonic — save it). Set it to persist identity.
BOT_MNEMONIC=
# Local file storage for the bot's wallet + tokens
BOT_DATA_DIR=./.bot-data
# OPTIONAL: only needed to RECEIVE tokens from the hosted Sphere wallet (see
# the wallet-api warning above). Leave empty to run DM + send only.
WALLET_API_URL=
```

## Key API surface used here

| Call | Signature | Notes |
|------|-----------|-------|
| `createNodeProviders(config)` | `impl/nodejs` | `config.network` is a required plain string (`'testnet2'`), throws `INVALID_CONFIG` if missing |
| `createOwnStorageWalletApiProviders(base, config)` | `impl/shared/wallet-api` | Adds the mailbox `delivery` + `walletApi` rail on top of `base`, keeps `base.tokenStorage` (own-storage custody) |
| `Sphere.init(options)` | SDK root | `{ ...providers, mnemonic?, autoGenerate?, network? }` → `{ sphere, created, generatedMnemonic? }` |
| `sphere.identity` | getter | `Identity \| null` — `{ chainPubkey, directAddress?, ipnsName?, nametag? }` |
| `sphere.communications.onDirectMessage(handler)` | returns unsubscribe fn | `handler: (m: DirectMessage) => void` |
| `sphere.communications.sendDM(recipient, content)` | `Promise<DirectMessage>` | |
| `sphere.payments.getBalance(coinId?)` | synchronous | `Asset[]` |
| `sphere.payments.mintFungibleToken(coinIdHex, amount)` | `Promise<{success:true,...} \| {success:false,error}>` | **`amount` is a `bigint`**, not a base-unit string — wrap `toBaseUnits()`'s string result in `BigInt(...)` |
| `sphere.payments.send(request)` | `Promise<TransferResult>` | `{ coinId, amount, recipient, memo? }`; `result.deliveryPending` true = success, never resend |
| `sphere.on('transfer:incoming', handler)` | returns unsubscribe fn | Only fires with a delivery rail composed in (mailbox or Nostr-native) |
| `isPossiblyCommittedSendOutcome(err)` | SDK root | Guards a `send` catch block — true means "do not retry" |
| `sphere.destroy()` | `Promise<void>` | Call on shutdown |

## DO NOT

- Do not pass `SPHERE_NETWORKS.testnet2` (the Connect network object) to `createNodeProviders` or
  `createOwnStorageWalletApiProviders` — both take the plain string `'testnet2'`.
- Do not pass a base-unit string to `mintFungibleToken` — it takes a `bigint`.
- Do not resend on a possibly-committed `send` failure — check `isPossiblyCommittedSendOutcome(err)`
  first and stop if it's true.
- Do not silently drop the wallet-api receive rail — if the user's bot needs to receive hosted-wallet
  payments, `WALLET_API_URL` (and `createOwnStorageWalletApiProviders`) is not optional; say so.
- Do not reach for `createWalletApiProviders` (no `OwnStorage` prefix) by mistake — see the warning
  above for why that preset is the wrong one for an own-keys bot.
