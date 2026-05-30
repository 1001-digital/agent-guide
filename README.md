# 1001 Agent Guide

This repo is the quick map for agents and humans building dapps with 1001 lego blocks.

Point agents at [`llms.txt`](llms.txt). Use this README for the human overview.

## Golden Path

For a new Nuxt/Vue EVM dapp:

1. Start with [`@1001-digital/layers.evm`](blocks/layers.md).
2. Add [`dapp-query`](blocks/data.md) only when reads need fallback, cache, or live updates.
3. Add the [`metadata`](blocks/metadata.md) stack for token URIs, IPFS/IPNS/Arweave, or NFT metadata.
4. Add [`indexing`](blocks/indexing.md) when the app needs its own event-derived data.
5. Add contracts, contract intelligence, auth, or IPFS infrastructure only when the product needs that layer.

## Blocks

| Need | Use | Guide |
| --- | --- | --- |
| Nuxt/Vue EVM app, wallet UX, transaction flows, base UI, CSS tokens | `@1001-digital/layers.evm` | [Layers](blocks/layers.md) |
| Resilient indexed/RPC/HTTP reads | `@1001-digital/dapp-query-core`, `@1001-digital/dapp-query-vue` | [Data](blocks/data.md) |
| Dweb URLs, IPFS/IPNS/Arweave fetches, NFT metadata | `@1001-digital/dweb-fetch`, `@1001-digital/normalize-dweb-url`, `@1001-digital/resolve-metadata` | [Metadata](blocks/metadata.md) |
| Lightweight event indexing or Ponder metadata caches | `@1001-digital/simple-indexer`, `@1001-digital/ponder-artifacts`, `@1001-digital/ponder-ens` | [Indexing](blocks/indexing.md) |
| NFT contract extensions and address checks | `@1001-digital/erc721-extensions`, `@1001-digital/check-address` | [Contracts](blocks/contracts.md) |
| Proxy detection and verified-contract docs | `@1001-digital/proxies`, `@1001-digital/natspec` | [Contract Intelligence](blocks/contract-intelligence.md) |
| In-app wallets and SIWE backend sessions | `@1001-digital/wagmi-in-app-wallet`, `@1001-digital/adonis-siwe` | [Wallet Auth](blocks/wallet-auth.md) |
| Owned IPFS node/gateway/admin API | `1001-digital/ipfs.server` | [Infra](blocks/infra.md) |

## Agent Rules

- Route by task before package name.
- Start simple; add blocks only when the app needs them.
- For Nuxt dapps that touch wallets or contracts, prefer `@1001-digital/layers.evm`.
- Use the layer's components and CSS variables before inventing custom primitives.
- Ask for missing product, style, chain, wallet, contract, metadata, or infrastructure details when the answer would change the build.
- Inspect the source repo for exact APIs when a guide gives the concept but not a full reference.
