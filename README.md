# Sphere Connect Plugin for Claude Code

A Claude Code plugin that helps developers integrate the [Sphere wallet](https://sphere.unicity.network) Connect protocol into their dApps.

## What it does

This plugin teaches Claude how to generate correct Sphere Connect integration code, including:

- **Transport detection** — iframe vs extension vs bridge iframe vs popup, with correct priority
- **Silent auto-connect** — no button flash on page load
- **Bridge iframe mode** — persistent session via hidden iframe, popup only for intent approval
- **Popup lifecycle** — HOST_READY handshake, close polling, session resume, bridge takeover
- **Full type safety** — correct imports, RPC methods, intents, permissions, events
- **Error handling** — all 15 error codes with appropriate responses

Supports **React**, **Node.js**, and **vanilla JavaScript** projects.

## Installation

### From marketplace

```
/plugin install sphere-connect
```

### Local development

```bash
claude --plugin-dir ./sphere-connect-plugin
```

## Usage

### Automatic (recommended)

Just ask Claude to add wallet integration:

- "Add Sphere wallet connect to my React app"
- "Integrate Sphere SDK Connect"
- "Connect my dApp to Sphere extension"

Claude automatically detects the skill and generates the integration.

### Explicit command

```
/sphere-connect:integrate react
/sphere-connect:integrate nodejs
/sphere-connect:integrate vanilla
```

## What gets generated

### React projects

| File | Purpose |
|------|---------|
| `src/lib/sphere-detection.ts` | `isInIframe()`, `hasExtension()` helpers |
| `src/hooks/useWalletConnect.ts` | Main hook: connect, disconnect, query, intent, events |
| `.env` | `VITE_WALLET_URL` for popup mode |

### Node.js projects

| File | Purpose |
|------|---------|
| `src/lib/sphere-client.ts` | `connectToSphere()` wrapper over WebSocketTransport |

## Sphere Connect Protocol

The Connect protocol enables dApps to interact with Sphere wallets through a typed RPC layer:

- **16 RPC methods** — identity, balance, assets, tokens, history, L1, resolve, DMs, events
- **6 intent actions** — send, l1_send, dm, payment_request, receive, sign_message
- **13 permission scopes** — granular access control per method/intent
- **34 real-time events** — transfers, payments, messages, sync, identity changes
- **15 error codes** — structured error handling

Full reference: [Sphere SDK Connect docs](https://github.com/unicitylabs/sphere-sdk/blob/main/docs/CONNECT.md)

## Requirements

- Claude Code v1.0.33+
- `@unicitylabs/sphere-sdk` npm package (installed automatically if missing)
- For Node.js: `ws` package

## License

MIT
