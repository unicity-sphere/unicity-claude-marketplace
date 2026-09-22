---
description: Generate Sphere wallet Connect integration for your project
disable-model-invocation: true
argument-hint: [react|nodejs|vanilla]
---

# Sphere Connect Integration

Generate a complete Sphere wallet Connect integration for this project.

Target framework: **$ARGUMENTS** (if not specified, auto-detect from package.json)

## Instructions

1. Detect the project framework if not specified:
   - Check `package.json` for `react`, `vue`, `svelte`, or Node.js
   - Check the bundler: `next` / `next.config.*` (Next.js) vs `vite.config.*` (Vite) vs anything else — it decides the env-var prefix and whether browser globals may be touched at module scope
   - Check for `tsconfig.json` (TypeScript vs JavaScript)
   - Check for the package manager lockfile

2. Check if `@unicitylabs/sphere-sdk` is already installed. If not, ask the user before installing.
   Install `0.17.x`: the wallet refuses a handshake from a dApp below `0.14.1` with
   `UNSUPPORTED_PROTOCOL_VERSION` (4007). On a TypeScript Node.js project also add
   `npm install -D @types/ws`.

3. Generate integration files using the sphere-connect skill templates:
   - **React**: One hook file using `autoConnect()` from SDK (see react-template.md)
   - **Vue / Svelte**: the same logic as a **framework-neutral module** (a factory or class over `autoConnect()`), wired into that framework's own reactivity. Do not hand a Vue or Svelte project the React hook.
   - **Node.js**: One client wrapper using `WebSocketTransport` (see nodejs-template.md)
   - **Vanilla JS**: Use `autoConnect()` directly (see SKILL.md vanilla example). No separate detection file — `autoConnect()` detects the transport itself.
   - **All templates**: always pass `network` in the `autoConnect()` / `ConnectClient` config, read from an env var (`VITE_SPHERE_NETWORK` / `NEXT_PUBLIC_SPHERE_NETWORK`) rather than hardcoded — `mainnet` and `testnet2` are both live, and the wallet rejects a mismatched or missing network with `INCOMPATIBLE_NETWORK` (4008).

4. For TypeScript projects, **check `compilerOptions.moduleResolution`** instead of adding path
   mappings: `bundler` / `node16` / `nodenext` already resolve the `./connect` subpaths, so do
   nothing; otherwise recommend switching to one of them. Keep the `paths` mapping only as a
   documented fallback for an application stuck on `node10` resolution — never for a library that
   emits declarations. See SKILL.md step 4.

5. Add the wallet URL and network to `.env` if missing (browser projects only), using the detected
   bundler's prefix — Vite: `VITE_WALLET_URL=https://sphere.unicity.network`,
   `VITE_SPHERE_NETWORK=testnet2`; Next.js: the same two as `NEXT_PUBLIC_*`.

6. Show a concise usage example after generating files, and tell the developer how to try it against
   a real wallet: serve the dApp over **https on a publicly reachable host** (an https tunnel such as
   `cloudflared` / `ngrok` in front of the dev server) and load it as a custom agent at
   `https://sphere.unicity.network/agents/custom?url=<url-encoded https url>`. A `localhost` /
   `127.0.0.1` URL in that query is answered **403** by the CDN before the wallet ever sees it
   (measured with curl — the same route with an https URL returns 200), and a plain `http://` URL
   would not be framed anyway: the wallet gates a custom tab on `https:` and otherwise falls back
   silently to its "Load Custom URL" prompt. Popup mode and localhost stay fine against a wallet the
   developer runs themselves. See SKILL.md, "Testing a local dApp against the hosted wallet".

Do NOT overwrite existing files without asking the user first.
