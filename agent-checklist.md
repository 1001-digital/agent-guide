# Agent Build Checklist

Use this before building from `llms.txt`. The goal is to catch the few decisions that change the shape of a dapp.

## Ask If Missing

Ask the prompter when any of these are missing and the choice would materially change the result:

- Product goal and first-screen user action.
- Target users and style direction.
- Wallet UX: read-only, injected wallet, WalletConnect, Safe, in-app wallet, or SIWE.
- Chains, RPCs, contract addresses, ABIs, and whether mock data is acceptable.
- Metadata sources and expected media formats.
- Data needs: direct reads, fallback, cache, live updates, or indexing.
- Backend/infrastructure ownership, including IPFS gateway or auth server.

If only small details are missing, state practical assumptions and keep them easy to change.

## Build Checks

- Route through `llms.txt`, then read only the block guides needed for the task.
- Start Nuxt/Vue EVM apps with `@1001-digital/layers.evm`.
- Use 1001 wallet, account, transaction, form, dialog, and token systems before custom replacements.
- Normalize decentralized URLs and metadata before rendering or storing.
- Treat in-app wallets as local-key custody.
- Treat `ipfs.server` as infrastructure, not a frontend dependency.

## Visual Checks

- Verify desktop and mobile layouts.
- Check disconnected wallet, connected wallet/profile, dialogs, network switching, and disconnect states.
- Check button/input/dialog contrast after theme overrides.
- Prefer CSS variables first; use selector overrides only when a variable does not exist.

## Ship Checks

- Run the repo's typecheck, tests, build, or docs validation.
- Verify local guide links and source links.
- Commit small logical batches.
