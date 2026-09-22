# sphere-connect

Connect your dApp to a Sphere wallet in minutes. Targets the iframe transport the wallet embeds dApps with, handles reconnect on page reload, and provides a `query` / `intent` / `event` API. Supports React, Node.js, and vanilla JS.

The plugin ships a `connect` skill plus an `/integrate` command that scaffolds the wallet-connection code (transport detection, connect hook, RPC/intent/event wiring) for your project.

## Install

```bash
# Add the marketplace, then install the plugin
/plugin marketplace add https://github.com/unicity-sphere/unicity-claude-marketplace
/plugin install sphere-connect@unicity-sphere
```

Or browse it in the `/plugin` interactive UI under the **Discover** tab.

## Updating

Plugins from third-party marketplaces do **not** auto-update by default — you decide when to pull new versions:

```bash
# 1. Refresh the marketplace catalog from GitHub
/plugin marketplace update unicity-sphere

# 2. Update the plugin to the latest version
/plugin update sphere-connect@unicity-sphere
```

Then run `/reload-plugins` (or restart Claude Code) to apply the new version in the current session.

To receive future updates automatically, enable it once: `/plugin` → **Marketplaces** → `unicity-sphere` → **Enable auto-update**.

## Network

When connecting, your dApp must declare the target network or the wallet rejects the handshake with `INCOMPATIBLE_NETWORK` (4008). Use the canonical constant:

```typescript
import { ConnectClient, SPHERE_NETWORKS } from '@unicitylabs/sphere-sdk/connect';

const client = new ConnectClient({
  // ...
  network: SPHERE_NETWORKS.testnet2, // required by the v2 compatibility gate
});
```

`SPHERE_NETWORKS` has two live entries — `mainnet` (id 1) and `testnet2` (id 4) — so read the target
from configuration rather than hardcoding it.

## Requirements

- `@unicitylabs/sphere-sdk` **0.17.x**. The wallet refuses a handshake from a dApp reporting an SDK
  below `0.14.1` with `UNSUPPORTED_PROTOCOL_VERSION` (4007).
- A wallet speaking **Connect 2.3** for the `mint_nft` intent.
- The production transport is the **iframe**: the Sphere wallet embeds the dApp. The Chrome
  extension wallet is discontinued — do not build on `ExtensionTransport`.

See the [Connect protocol docs](https://github.com/unicity-sphere/sphere-sdk/blob/main/docs/CONNECT.md) for the full RPC / intent / event reference.
