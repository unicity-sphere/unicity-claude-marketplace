# sphere-connect

Connect your dApp to a Sphere wallet in minutes. Auto-detects extension / iframe / popup, handles reconnect on page reload, and provides a `query` / `intent` / `event` API. Supports React, Node.js, and vanilla JS.

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

See the [Connect protocol docs](https://github.com/unicitylabs/sphere-sdk/blob/main/docs/CONNECT.md) for the full RPC / intent / event reference.
