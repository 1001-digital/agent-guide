# Metadata

Use the metadata stack when a dapp needs to fetch, normalize, store, or render decentralized-web and NFT metadata.

The normal flow is:

1. Normalize the identifier.
2. Fetch through a dweb-aware client.
3. Normalize the JSON into renderable metadata.

## Use This For

- Token URI and contract URI flows.
- IPFS, IPNS, Arweave, HTTP(S), raw CIDs, and `eip155:` NFT references.
- Converting gateway URLs into canonical protocol URIs.
- Normalizing ERC-721, ERC-1155, and contract-level metadata before rendering.
- Feeding metadata into `EvmArtifact`, galleries, search indexes, and caches.

## Do Not Use This For

- Contract reads by themselves. Use viem, wagmi, `layers`, or `dapp-query`.
- A canonical metadata store without persistence of your own.
- Assuming every NFT returns valid JSON. Always handle missing media and malformed fields.

## Packages

- Source repo: <https://github.com/1001-digital/dweb-fetch>
- Source repo: <https://github.com/1001-digital/normalize-dweb-url>
- Source repo: <https://github.com/1001-digital/resolve-metadata>
- `@1001-digital/dweb-fetch`: protocol-aware fetch for IPFS, IPNS, Arweave, HTTPS, and EIP-155 NFT references.
- `@1001-digital/normalize-dweb-url`: canonical URI normalization and resolvable URI checks.
- `@1001-digital/resolve-metadata`: token and contract metadata normalization.

## Install

```sh
pnpm add @1001-digital/dweb-fetch @1001-digital/normalize-dweb-url @1001-digital/resolve-metadata
```

If the app resolves `eip155:` token references, also add/configure the chain client dependencies used by the app, usually `viem`.

## Minimal Flow

```ts
import { createDwebFetch } from '@1001-digital/dweb-fetch'
import { normalizeUri } from '@1001-digital/normalize-dweb-url'
import { resolveTokenMetadata } from '@1001-digital/resolve-metadata'

const dweb = createDwebFetch({
  ipfs: { mode: 'gateway' },
})

export async function loadTokenMetadata(tokenUri: string) {
  const canonicalUri = normalizeUri(tokenUri)
  const response = await dweb.fetch(canonicalUri)
  const raw = await response.json()

  return resolveTokenMetadata(raw)
}
```

Render with the layer:

```vue
<template>
  <EvmArtifact :metadata="metadata" />
</template>
```

## Normalize URLs

Use `normalize-dweb-url` before storing identifiers, comparing URLs, creating cache keys, or deciding whether something can be fetched.

```ts
import {
  isResolvableUri,
  normalizeUri,
} from '@1001-digital/normalize-dweb-url'

normalizeUri('https://gateway.pinata.cloud/ipfs/QmXxx')
// 'ipfs://QmXxx'

normalizeUri('https://arweave.net/txId123')
// 'ar://txId123'

normalizeUri('QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG')
// 'ipfs://QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG'

isResolvableUri('ipfs://QmXxx')
// true
```

Store canonical protocol URIs such as `ipfs://...` or `ar://...` when possible. Do not store a gateway URL as the canonical ID unless the original content is truly just HTTP.

## Fetch Dweb Content

Create one dweb client near the app boundary:

```ts
import { createDwebFetch } from '@1001-digital/dweb-fetch'

export const dweb = createDwebFetch({
  ipfs: {
    mode: 'gateway',
    gateways: ['https://ipfs.io'],
  },
  arweave: {
    gateways: ['https://arweave.net'],
    routingStrategy: 'fastest',
  },
})
```

Use gateway origins such as `https://ipfs.io` in config. The client appends `/ipfs/<cid>` or `/ipns/<name>` internally.

Common fetches:

```ts
await dweb.fetch('ipfs://bafy...')
await dweb.fetch('ipns://example.eth')
await dweb.fetch('ar://txId')
await dweb.fetch('https://example.com/metadata.json')
```

Resolve a renderable HTTP URL without fetching the bytes:

```ts
const imageUrl = await dweb.resolveUrl('ipfs://bafy.../image.png')
```

## EIP-155 NFT References

Use `eip155:` when the metadata reference is a chain, token standard, contract, and token ID rather than a direct URI.

```ts
const dweb = createDwebFetch({
  eip155: {
    rpcUrls: {
      1: 'https://eth.llamarpc.com',
      8453: 'https://mainnet.base.org',
    },
  },
})

await dweb.fetch(
  'eip155:1/erc721:0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D/1234',
)
```

Use this only when the app has reliable RPCs for the referenced chains.

## Normalize Metadata

Use `resolve-metadata` after fetching JSON:

```ts
import {
  resolveContractMetadata,
  resolveTokenMetadata,
} from '@1001-digital/resolve-metadata'

const token = resolveTokenMetadata(rawTokenJson)
const collection = resolveContractMetadata(rawContractJson)
```

The normalizers handle common NFT field variants, normalize media URLs, preserve useful unknown fields in `extra`, and return predictable shapes for app code. Inspect the package source/types when exact output fields matter for a component or database schema.

## Cache Shape

For app caches or database rows, store:

- Canonical metadata URI.
- Fetch timestamp and source.
- Normalized token or contract metadata.
- Raw JSON only when debugging, provenance, or reprocessing requires it.

## Pair With

- [`layers.md`](layers.md) for `EvmArtifact` and dweb helpers in Nuxt apps.
- [`data.md`](data.md) when metadata is part of cached app reads.
- [`indexing.md`](indexing.md) for Ponder artifact caches.
- [`infra.md`](infra.md) when the project owns its IPFS gateway.

## Agent Checklist

- Normalize URI before storage or cache keys.
- Fetch with `dweb-fetch`, not plain `fetch`, for dweb protocols.
- Normalize raw JSON before rendering.
- Use canonical protocol URIs internally.
- Configure RPCs before using `eip155:` references.
- Handle missing images, broken JSON, and unknown media gracefully.
