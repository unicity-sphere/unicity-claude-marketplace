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
   - Check for `tsconfig.json` (TypeScript vs JavaScript)
   - Check for the package manager lockfile

2. Check if `@unicitylabs/sphere-sdk` is already installed. If not, ask the user before installing.

3. Generate the integration files using the sphere-connect skill templates.

4. For TypeScript browser projects, add path mappings to `tsconfig.json` for `@unicitylabs/sphere-sdk/connect` and `@unicitylabs/sphere-sdk/connect/browser`.

5. Add `VITE_WALLET_URL=https://sphere.unicity.network` to `.env` if it doesn't exist (browser projects only).

6. Show a concise usage example after generating files.

Do NOT overwrite existing files without asking the user first.
