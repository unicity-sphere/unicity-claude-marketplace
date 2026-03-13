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

## Available Plugins

| Plugin | Category | Description |
|--------|----------|-------------|
| [sphere-connect](plugins/sphere-connect/) | development | Integrate Sphere wallet Connect protocol into your dApp |

## For Plugin Developers

To add a plugin to this marketplace, submit a PR:
1. Create a directory under `plugins/your-plugin-name/`
2. Add `.claude-plugin/plugin.json`, `skills/`, and/or `commands/`
3. Add an entry to `.claude-plugin/marketplace.json`
