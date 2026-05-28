# Metadata

Use this guide when an agent needs to fetch, normalize, store, or render decentralized-web and NFT metadata.

## What To Use This For

- Token URI and contract URI flows.
- IPFS, IPNS, Arweave, HTTP, HTTPS, and `eip155:` NFT references.
- Converting gateway URLs and raw CIDs into canonical protocol URIs.
- Normalizing ERC-721, ERC-1155, and contract-level metadata into predictable typed shapes.
- Preparing metadata for `EvmArtifact`, galleries, collection pages, search indexes, or caches.

## When Not To Use It

- Do not use `resolve-metadata` to fetch content. It normalizes JSON and URI strings; use `dweb-fetch` for retrieval.
- Do not store gateway URLs as canonical identifiers when a protocol URI can be stored instead.
- Do not assume metadata from NFT contracts is valid JSON or has consistent field names.
- Do not use verified IPFS fetching for high-volume server jobs if gateway-only fetching is an acceptable trust tradeoff.

## Packages/Repos Involved

- Source repo: <https://github.com/1001-digital/dweb-fetch>
- Source repo: <https://github.com/1001-digital/normalize-dweb-url>
- Source repo: <https://github.com/1001-digital/resolve-metadata>
- `@1001-digital/dweb-fetch`: protocol-aware fetch for IPFS, IPNS, Arweave, HTTPS, and EIP-155 NFT references.
- `@1001-digital/normalize-dweb-url`: canonical URI normalization and resolvable URI checks.
- `@1001-digital/resolve-metadata`: strict token and contract metadata normalization.

## Install Commands

```sh
pnpm add @1001-digital/dweb-fetch @1001-digital/normalize-dweb-url @1001-digital/resolve-metadata
```

If resolving `eip155:` token references, also install/configure the chain client dependencies used by your app, typically `viem`:

```sh
pnpm add viem
```

## Minimal Setup

Default client:

```ts
import { createDwebFetch } from '@1001-digital/dweb-fetch'

export const dweb = createDwebFetch()
```

Server-heavy gateway-only IPFS mode:

```ts
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

EIP-155 NFT reference support:

```ts
export const dweb = createDwebFetch({
  eip155: {
    rpcUrls: {
      1: 'https://eth.llamarpc.com',
      8453: 'https://mainnet.base.org',
    },
  },
})
```

Canonical NFT metadata flow:

```ts
import { normalizeUri } from '@1001-digital/normalize-dweb-url'
import { createDwebFetch } from '@1001-digital/dweb-fetch'
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

## Core APIs/Components/Contracts/Config

### `normalize-dweb-url`

Use this package before storing metadata URLs, comparing identifiers, creating cache keys, or deciding if a field can be fetched.

```ts
import {
  normalizeUri,
  isResolvableUri,
} from '@1001-digital/normalize-dweb-url'

normalizeUri('https://gateway.pinata.cloud/ipfs/QmXxx')
// 'ipfs://QmXxx'

normalizeUri('https://bafyABC.ipfs.nftstorage.link/meta.json')
// 'ipfs://bafyABC/meta.json'

normalizeUri('https://arweave.net/txId123')
// 'ar://txId123'

normalizeUri('QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG')
// 'ipfs://QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG'

isResolvableUri('ipfs://QmXxx')
// true
```

Normalization handles:

- IPFS path gateways: `https://ipfs.io/ipfs/<cid>`
- IPFS subdomain gateways: `https://<cid>.ipfs.<gateway>/path`
- IPNS path and subdomain gateways.
- Arweave gateways.
- Redundant path segments such as `ipfs://ipfs/<cid>`.
- Embedded gateway URLs such as `ipfs://https://gateway.pinata.cloud/ipfs/<cid>`.
- Raw CIDv0 and CIDv1 strings.
- `data:` and normal HTTPS passthrough.

### `dweb-fetch`

Create a dweb client that routes by URL scheme.

```ts
import { createDwebFetch } from '@1001-digital/dweb-fetch'

const dweb = createDwebFetch({
  ipfs: {
    gateways: ['https://ipfs.io'],
    mode: 'verified',
  },
  arweave: {
    gateways: ['https://arweave.net', 'https://ar-io.dev'],
    routingStrategy: 'fastest',
    useNetworkDiscovery: true,
  },
  eip155: {
    rpcUrls: {
      1: 'https://eth.llamarpc.com',
    },
  },
})
```

Use gateway origins such as `https://ipfs.io` in `dweb-fetch` config. The client appends `/ipfs/<cid>` or `/ipns/<name>` internally; do not pass a path gateway such as `https://ipfs.io/ipfs/` here.

Supported fetch inputs:

```ts
await dweb.fetch('ipfs://bafy...')
await dweb.fetch('ipns://example.eth')
await dweb.fetch('ar://txId')
await dweb.fetch('https://example.com/metadata.json')
await dweb.fetch('eip155:1/erc721:0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D/1234')
```

Resolve a renderable HTTP URL without fetching the bytes:

```ts
const imageUrl = await dweb.resolveUrl('ipfs://bafy.../image.png')
await dweb.destroy()
```

Protocol behavior:

| Protocol | Behavior |
| --- | --- |
| IPFS/IPNS | Uses verified fetching by default, with HTTPS gateway fallback. |
| IPFS gateway mode | Uses configured gateways only and avoids Helia initialization. |
| Arweave | Tries static gateways and can use network discovery fallback. |
| HTTP/HTTPS | Native `fetch` passthrough. |
| EIP-155 | Resolves ERC-721 `tokenURI` or ERC-1155 `uri`, then fetches the resulting URI. |

Helpful APIs:

```ts
import {
  extractScheme,
  parseDwebUrl,
  parseEip155Uri,
  resolveEip155TokenUri,
} from '@1001-digital/dweb-fetch'
```

Resolve an EIP-155 token URI without fetching metadata:

```ts
const tokenUri = await resolveEip155TokenUri({
  chainId: 1,
  standard: 'erc721',
  contract: '0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D',
  tokenId: '1234',
  rpcUrl: 'https://eth.llamarpc.com',
})
```

For ERC-1155, `{id}` placeholders are replaced with the lower-case 64-character hex token ID expected by the metadata spec.

Error classes:

- `DwebFetchError`: base error.
- `DwebUnsupportedProtocolError`: unknown scheme.
- `Eip155ResolutionError`: token URI resolution failed, missing RPC, empty URI, or unsupported standard.

### `resolve-metadata`

Use after fetching JSON. It accepts unknown input and returns typed metadata with null/default fields and unknown fields preserved in `extra`.

```ts
import {
  resolveTokenMetadata,
  resolveContractMetadata,
} from '@1001-digital/resolve-metadata'

const token = resolveTokenMetadata(rawTokenJson)
const collection = resolveContractMetadata(rawContractJson)
```

Token metadata normalization:

| Input | Output behavior |
| --- | --- |
| `image`, `image_url`, `image_data` | Canonical image fields where possible. |
| `animation_url`, `animation` | Canonical animation field. |
| Gateway URLs | Protocol URIs such as `ipfs://` and `ar://`. |
| `background_color` | Hex normalized without `#`, 3-character expanded to 6-character. |
| `attributes` array | Normalized to consistent trait/value objects. |
| Numeric string attributes | Coerced to numbers. |
| ERC-1155 fields | `decimals`, `properties`, and `localization` preserved. |
| Unknown fields | Preserved in `extra`. |

Contract metadata normalization:

| Input | Output behavior |
| --- | --- |
| `image`, `banner_image`, `featured_image` | URI-normalized media fields. |
| `external_url` | Mapped to `external_link` when appropriate. |
| `collaborators` | Validated as string array. |
| Unknown fields | Preserved in `extra`. |

Attribute formats handled:

```ts
resolveTokenMetadata({
  attributes: [
    { trait_type: 'Color', value: 'Blue' },
    { name: 'Level', value: '7' },
  ],
})

resolveTokenMetadata({
  attributes: {
    Color: 'Blue',
    Rarity: 'Legendary',
  },
})
```

## Common Pairings With Other 1001 Blocks

- Pair with `layers.evm` and `EvmArtifact` for display.
- Pair with `dapp-query` when metadata URLs are returned from resilient chain/indexer reads.
- Pair with `simple-indexer` or Ponder helpers to cache metadata outside the render path.
- Pair with `ipfs.server` when the app owns a gateway or pinning service.
- Pair with `proxies` and `natspec` when metadata is shown on contract explorer pages.

## Practical Implementation Patterns

### Token URI To Renderable Artifact

```ts
import { normalizeUri } from '@1001-digital/normalize-dweb-url'
import { createDwebFetch } from '@1001-digital/dweb-fetch'
import { resolveTokenMetadata } from '@1001-digital/resolve-metadata'

const dweb = createDwebFetch({
  ipfs: { mode: 'gateway' },
})

export async function resolveArtifact(tokenUri: string) {
  const uri = normalizeUri(tokenUri)
  const response = await dweb.fetch(uri)

  if (!response.ok) {
    throw new Error(`metadata fetch failed: ${response.status}`)
  }

  return resolveTokenMetadata(await response.json())
}
```

```vue
<template>
  <EvmArtifact
    v-if="metadata"
    :metadata="metadata"
    aspect-ratio="1 / 1"
    use-background-color
  />
</template>
```

### Contract URI To Collection Header

```ts
import { resolveContractMetadata } from '@1001-digital/resolve-metadata'

export async function resolveCollection(contractUri: string) {
  const response = await dweb.fetch(normalizeUri(contractUri))
  const metadata = resolveContractMetadata(await response.json())

  return {
    name: metadata.name,
    description: metadata.description,
    image: metadata.image,
    banner: metadata.banner_image,
    link: metadata.external_link,
  }
}
```

### EIP-155 Reference

Use `eip155:` when you want a single portable reference to an NFT token rather than separately storing chain, standard, contract, and token ID.

```ts
const uri =
  'eip155:1/erc721:0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D/1234'

const metadata = resolveTokenMetadata(await dweb.fetch(uri).then((res) => res.json()))
```

Configure RPC URLs for every chain ID you will resolve:

```ts
createDwebFetch({
  eip155: {
    rpcUrls: {
      1: process.env.MAINNET_RPC_URL!,
      8453: process.env.BASE_RPC_URL!,
    },
  },
})
```

### Metadata Cache Key

Use canonical URIs for cache keys:

```ts
const key = `metadata:${normalizeUri(tokenUri)}`
```

Do not use gateway URLs as cache keys. The same CID can appear through many gateways.

### Server Batch Job

Use gateway mode for predictable server resource usage:

```ts
const metadataClient = createDwebFetch({
  ipfs: {
    mode: 'gateway',
    gateways: ['https://ipfs.io', 'https://cloudflare-ipfs.com'],
  },
  arweave: {
    gateways: ['https://arweave.net'],
    useNetworkDiscovery: false,
  },
})
```

Gateway mode is a trust/performance tradeoff: it avoids running verified IPFS retrieval internals in a long-lived server process.

## CSS/Config/Runtime/Env Details

- Keep gateway and RPC endpoints in runtime config or server environment variables.
- Do not expose private RPC keys in public Nuxt runtime config. Use public RPCs client-side or proxy through server routes.
- In Nuxt apps, build the `dweb` client in a plugin or composable. Use separate server/client configs if server has private endpoints.
- In browser apps, avoid fetching many large media files through JavaScript when a normalized media URI can be handed to an `<img>`, `<video>`, or `EvmArtifact`.
- If using `ipfs.server`, configure app gateways to point at the project-owned gateway for pinned content.

Nuxt runtime config sketch:

```ts
export default defineNuxtConfig({
  runtimeConfig: {
    dweb: {
      mainnetRpcUrl: '',
    },
    public: {
      dweb: {
        ipfsGateway: 'https://ipfs.io/ipfs/',
        arweaveGateway: 'https://arweave.net/',
      },
    },
  },
})
```

## Gotchas And Failure Modes

- Metadata can be non-JSON, unavailable, slow, or too large. Always handle fetch and parse errors.
- Gateway URLs are not canonical. Normalize before storage, cache keys, and dedupe.
- `resolve-metadata` does not fetch linked image/media files.
- `resolve-metadata` preserves unknown fields in `extra`; do not throw them away if the app needs marketplace-specific data.
- ERC-1155 `{id}` replacement must use 64-character lower-case hex. Let `dweb-fetch` handle this for EIP-155 references.
- IPNS can be slower and less stable than IPFS CIDs. Consider caching results.
- `data:` URIs can be valid but large. Be careful when storing or logging them.
- Private RPC URLs should not be configured in client-visible env vars.

## Agent Checklist

- Normalize every incoming metadata URI with `normalizeUri`.
- Use canonical protocol URIs for database/cache identifiers.
- Use `dweb-fetch` to retrieve IPFS/IPNS/Arweave/EIP-155 resources.
- Choose IPFS `verified` mode for content verification; choose `gateway` mode for trusted high-volume gateway workloads.
- Use `resolveTokenMetadata` for token-level JSON.
- Use `resolveContractMetadata` for collection/contract-level JSON.
- Preserve `extra` if downstream UI may need marketplace-specific fields.
- Render normalized metadata through `EvmArtifact` when using `layers.evm`.
- Add error states for missing JSON, unsupported protocol, bad RPC config, stale gateway, and missing media.
