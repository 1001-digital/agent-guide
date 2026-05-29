# Agent Build Checklist

Use this before building a 1001 dapp from `llms.txt`. The goal is to avoid plausible but wrong assumptions, especially around product shape, chain setup, wallet UX, and visual customization.

## Ask Before Building

Ask the prompter when any of these are missing and the choice would materially change the result:

- Product goal: what the user should be able to do on the first screen.
- Target users: collectors, creators, admins, developers, operators, or a mixed audience.
- Style direction: brand references, mood, density, palette, typography, light/dark mode, and whether the app should feel editorial, operational, playful, gallery-like, or protocol-native.
- Wallet UX: injected wallet only, WalletConnect, Safe App support, in-app wallet, SIWE, or guest/read-only mode.
- Chain setup: target chains, default chain, RPC/indexer endpoints, block ranges, contract addresses, ABIs, and whether mock/local data is acceptable.
- Metadata setup: expected token URI formats, IPFS/IPNS/Arweave gateways, media types, and fallback behavior.
- Data freshness: live updates, cache lifetime, offline mode, optimistic updates, and whether direct RPC fallback is required.
- Indexing scope: browser-local index, server process, Ponder app, reorg tolerance, and persistence requirements.
- Deployment/infrastructure: whether the app needs its own IPFS gateway, auth server, or backend.

If only one or two details are missing, ask focused questions. If many are missing and the user wants momentum, state the assumptions before coding and keep them easy to change.

## Build Checks

- Route by task through `llms.txt`, then read the relevant `blocks/*.md` guide before implementation.
- Prefer `@1001-digital/layers.evm` for Nuxt/Vue apps that touch wallets or contracts.
- Use the layer's wallet, account, transaction, form, dialog, and CSS token systems before creating app-specific primitives.
- Normalize decentralized URLs before storing or rendering metadata.
- Use `dapp-query` when a read needs fallback, cache, live updates, or multiple sources.
- Use `simple-indexer` for small owned indexes; use Ponder helpers only inside Ponder apps.
- Treat in-app wallets as local-key custody and explain that in product UI.
- Treat `ipfs.server` as deployable infrastructure, not a frontend package.

## Visual QA Checks

- Verify the first viewport is the actual app experience, not a generic landing page.
- Check desktop and mobile layouts for overflow, overlap, cramped labels, and text clipped inside buttons/cards.
- Check disconnected wallet trigger, wallet selection dialog, in-app wallet setup, connected profile trigger, profile dialog, network switcher, and disconnect flow.
- Check computed `color`, `background-color`, and disabled states for wallet buttons, primary buttons, inputs, dialogs, and toasts.
- If the app customizes the 1001 design system, verify `color-scheme`, `--background`, `--color`, button tokens, input tokens, card tokens, and dialog tokens together.
- Confirm hover/focus states are readable and do not shift layout.
- Confirm custom CSS is global when it needs to affect dialogs, toasts, or wallet overlays.

## Ship Checks

- Run the repo's typecheck/test/build commands.
- Verify links, env variable names, and local guide references.
- Browser-test the primary user journey end to end.
- Commit small logical batches so later agents can see why each change exists.
