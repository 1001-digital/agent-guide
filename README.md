# 1001 Agent Guide

This repository is the entrypoint for agents and humans building with the 1001 dapp lego blocks.

Point agents at [`llms.txt`](llms.txt). Use this README when you want the human-sized map first.

## Golden Path

For a new Nuxt/Vue EVM dapp, start here:

1. Build the app on [`@1001-digital/layers.evm`](blocks/layers.md) for wallet UX, EVM components, transaction flows, and the shared design system.
2. Add [`dapp-query`](blocks/data.md) when chain reads need indexed/RPC fallback, browser cache, or live updates.
3. Use the [`metadata`](blocks/metadata.md) stack for token URIs, contract URIs, IPFS, IPNS, Arweave, and NFT metadata normalization.
4. Add [`simple-indexer`](blocks/indexing.md) for local browser/server indexing, or the Ponder helpers when working inside a Ponder app.
5. Reach for [`proxies` and `natspec`](blocks/contract-intelligence.md) when the app needs to understand contracts, implementations, facets, or human-readable verified docs.

## Block Matrix

| Need | Use | Start here |
| --- | --- | --- |
| Nuxt/Vue EVM app foundation | `@1001-digital/layers.evm`, `@1001-digital/layers.base`, `@1001-digital/components.evm`, `@1001-digital/components`, `@1001-digital/styles` | [Layers guide](blocks/layers.md) |
| Wallet connection, account UI, transaction UX | `@1001-digital/layers.evm` | [Layers guide](blocks/layers.md) |
| CSS variables, component tokens, design-system overrides | `@1001-digital/styles` through `layers` | [Layers guide](blocks/layers.md) |
| Resilient on-chain data reads | `@1001-digital/dapp-query-core`, `@1001-digital/dapp-query-vue` | [Data guide](blocks/data.md) |
| IPFS/IPNS/Arweave/eip155 fetching | `@1001-digital/dweb-fetch` | [Metadata guide](blocks/metadata.md) |
| Canonical dweb URLs and raw CIDs | `@1001-digital/normalize-dweb-url` | [Metadata guide](blocks/metadata.md) |
| NFT and contract metadata normalization | `@1001-digital/resolve-metadata` | [Metadata guide](blocks/metadata.md) |
| Lightweight event indexing | `@1001-digital/simple-indexer` | [Indexing guide](blocks/indexing.md) |
| Ponder artifact and ENS caches | `@1001-digital/ponder-artifacts`, `@1001-digital/ponder-ens` | [Indexing guide](blocks/indexing.md) |
| NFT contract extensions | `@1001-digital/erc721-extensions` | [Contracts guide](blocks/contracts.md) |
| Address type helpers | `@1001-digital/check-address` | [Contracts guide](blocks/contracts.md) |
| Proxy, diamond, clone, beacon detection | `@1001-digital/proxies` | [Contract intelligence guide](blocks/contract-intelligence.md) |
| NatSpec from verified contracts | `@1001-digital/natspec` | [Contract intelligence guide](blocks/contract-intelligence.md) |
| In-app wagmi wallet | `@1001-digital/wagmi-in-app-wallet` | [Wallet/auth guide](blocks/wallet-auth.md) |
| AdonisJS SIWE sessions | `@1001-digital/adonis-siwe` | [Wallet/auth guide](blocks/wallet-auth.md) |
| Own IPFS node/gateway/admin API | `ipfs.server` | [Infra guide](blocks/infra.md) |

## Agent Rules

- Route by task first, package second.
- Prefer `layers.evm` for Nuxt dapps that touch wallets or contracts.
- Normalize decentralized URLs before storing them.
- Normalize metadata before rendering it.
- Use dapp-query when a direct RPC-only read would be fragile.
- Use `simple-indexer` for small owned indexes; use the Ponder helpers inside Ponder apps.
- Treat in-app wallets as local-key custody.
- Treat IPFS server setup as infrastructure, not an app dependency.

## Guide Template

Each deep guide follows the same structure:

- What to use this for
- When not to use it
- Packages/repos involved
- Install commands
- Minimal setup
- Core APIs/components/contracts/config
- Common pairings with other 1001 blocks
- Practical implementation patterns
- CSS/config/runtime/env details where relevant
- Gotchas and failure modes
- Agent checklist
