# Unicity Claude Marketplace

Official Claude Code plugin marketplace for the Unicity Sphere wallet ecosystem.

## Installation

### VS Code Extension

1. Open **Manage Plugins** → **Marketplaces** tab
2. Add: `https://github.com/unicity-sphere/unicity-claude-marketplace`
3. Switch to **Plugins** tab and enable the plugins you need

### CLI (Terminal)

In Claude Code interactive mode, use the `/plugin` command:

```bash
# 1. Add the marketplace
/plugin marketplace add https://github.com/unicity-sphere/unicity-claude-marketplace

# 2. Browse and install plugins via interactive UI
/plugin
```

In the interactive UI, switch to the **Discover** tab to find and install `sphere-connect`.

## Updating

Plugins from third-party marketplaces (like this one) do **not** auto-update by default — you control when new versions are pulled.

```bash
# 1. Refresh the marketplace catalog from GitHub
/plugin marketplace update unicity-sphere

# 2. Update an installed plugin to the latest version
/plugin update sphere-connect@unicity-sphere
```

Then run `/reload-plugins` (or restart Claude Code) to apply it in your current session.

To get future updates automatically, enable it once per marketplace: `/plugin` → **Marketplaces** → `unicity-sphere` → **Enable auto-update**.

## Available Plugins

| Plugin | Category | Description |
|--------|----------|-------------|
| [sphere-connect](plugins/sphere-connect/) | development | Integrate Sphere wallet Connect protocol into your dApp |

## For Plugin Developers

To add a plugin to this marketplace, submit a PR:
1. Create a directory under `plugins/your-plugin-name/`
2. Add `.claude-plugin/plugin.json`, `skills/`, and/or `commands/`
3. Add an entry to `.claude-plugin/marketplace.json`
