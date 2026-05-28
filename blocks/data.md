# Data

Use this guide when an agent needs resilient on-chain data reads for a dapp. The 1001 data block is `dapp-query`.

## What To Use This For

- Frontends that need to read chain-derived data from more than one source.
- Indexer-first reads with RPC fallback.
- RPC-first reads with cache and live refresh behavior.
- Browser-persistent query caches.
- Vue dapps that need reactive query state.
- Queries where a single GraphQL endpoint, REST endpoint, or RPC provider would be too fragile.

## When Not To Use It

- Do not use `dapp-query` for a one-off direct contract read where `readContract` is enough.
- Do not use it as a full indexing framework. Use `simple-indexer` or a Ponder app when you need to build and own derived tables.
- Do not put incompatible data shapes in different sources for the same query. Every source in a query must return the same domain type.
- Do not use the `race` strategy by default. It spends more requests; reserve it for latency-sensitive reads.

## Packages/Repos Involved

- Source repo: <https://github.com/1001-digital/dapp-query>
- `@1001-digital/dapp-query-core`: source definitions, query client, strategies, cache implementations.
- `@1001-digital/dapp-query-vue`: Vue plugin and `useQuery` composable.
- Peer dependency: `viem` for RPC sources.

## Install Commands

Core only:

```sh
pnpm add @1001-digital/dapp-query-core viem
```

Vue integration:

```sh
pnpm add @1001-digital/dapp-query-core @1001-digital/dapp-query-vue viem
```

## Minimal Setup

Create one query client per app.

```ts
import {
  createQueryClient,
  idbCache,
} from '@1001-digital/dapp-query-core'

export const queryClient = createQueryClient({
  cache: idbCache('my-dapp'),
  defaultStaleTime: 5 * 60_000,
  defaultStaleWhileRevalidate: true,
})
```

Use `idbCache` in browsers when cached data should survive reloads. Use `memoryCache` for tests, server processes, or pages where persistence is unnecessary.

Vue app setup:

```ts
import { createApp } from 'vue'
import { dappQueryPlugin } from '@1001-digital/dapp-query-vue'
import { queryClient } from './query-client'
import App from './App.vue'

createApp(App)
  .use(dappQueryPlugin, queryClient)
  .mount('#app')
```

In Nuxt/layers apps, register the plugin in a client plugin if IndexedDB is used:

```ts
// plugins/dapp-query.client.ts
import { createQueryClient, idbCache } from '@1001-digital/dapp-query-core'
import { dappQueryPlugin } from '@1001-digital/dapp-query-vue'

export default defineNuxtPlugin((nuxtApp) => {
  const client = createQueryClient({
    cache: idbCache('my-dapp'),
  })

  nuxtApp.vueApp.use(dappQueryPlugin, client)
})
```

## Core APIs/Components/Contracts/Config

### Query Client

```ts
const client = createQueryClient({
  cache: idbCache('my-app'),
  defaultStaleTime: 5 * 60_000,
  defaultStaleWhileRevalidate: true,
})
```

Client methods agents should use:

| API | Use |
| --- | --- |
| `client.fetch(query, ...args)` | One-shot fetch. Returns fresh cache when possible, otherwise fetches. |
| `client.subscribe(query, args, callback)` | Reactive subscription with cache, background revalidation, and live watchers. |
| `client.invalidate(query, ...args)` | Clears a cache entry and revalidates active subscribers. |
| `client.waitForChange(query, args, predicate, options?)` | Poll fresh source data until a post-write predicate passes, updating cache/subscribers when it does. |
| `client.getSourceHealth(sourceId)` | Inspect failures, latency, and backoff state. |
| `client.reset()` | Clear cache, health state, active watchers, and in-flight dedupe state. |

### Query Definition

A query describes cache key, sources, strategy, TTL, and optional final transform.

```ts
const tokenTransfersQuery = {
  key: (collection: string, tokenId: bigint) =>
    `transfers:${collection.toLowerCase()}:${tokenId}`,
  sources: [indexedTransfers, rpcTransfers],
  strategy: 'fallback' as const,
  staleTime: 5 * 60_000,
  staleWhileRevalidate: true,
  transform: (rows: Transfer[]) =>
    rows.sort((a, b) => Number(b.block - a.block)),
}
```

Rules:

- `key` must include every argument that changes the result.
- Keep keys stable and string-only.
- Put source-specific parsing inside the source `transform`, not after the query.
- Use final `transform` only for source-independent shaping such as sorting or dedupe.

### Source Types

`dapp-query` source callbacks are typed as `(...args: unknown[])` in the published package. In strict TypeScript, keep callback parameters broad and cast the argument tuple inside the callback. Do not narrow the callback signature itself.

#### GraphQL Source

Use for Ponder, The Graph, or any GraphQL indexer.

```ts
import { graphqlSource } from '@1001-digital/dapp-query-core'

const indexedTransfers = graphqlSource({
  endpoints: [
    'https://indexer-primary.example/graphql',
    'https://indexer-backup.example/graphql',
  ],
  query: `query($collection: String!, $tokenId: BigInt!) {
    transfers(where: { collection: $collection, tokenId: $tokenId }) {
      items { from to tokenId block transactionHash }
    }
  }`,
  variables: (...args: unknown[]) => {
    const [collection, tokenId] = args as [string, bigint]
    return {
      collection: collection.toLowerCase(),
      tokenId: tokenId.toString(),
    }
  },
  transform: (data: any) => data.transfers.items.map((row: any) => ({
    from: row.from,
    to: row.to,
    tokenId: BigInt(row.tokenId),
    block: BigInt(row.block),
    transactionHash: row.transactionHash,
  })),
})
```

Use multiple endpoints for failover. The source tries them in order.

#### RPC Source

Use for direct event-log reads through viem.

```ts
import { rpcSource } from '@1001-digital/dapp-query-core'
import { createPublicClient, http, parseAbiItem } from 'viem'
import { mainnet } from 'viem/chains'

const publicClient = createPublicClient({
  chain: mainnet,
  transport: http('https://eth.llamarpc.com'),
})

const rpcTransfers = rpcSource({
  client: publicClient,
  event: parseAbiItem(
    'event Transfer(address indexed from, address indexed to, uint256 indexed tokenId)'
  ),
  address: '0x0000000000000000000000000000000000000000',
  fromBlock: 18_000_000n,
  maxBlockRange: 2_000,
  filter: (...args: unknown[]) => {
    const [, tokenId] = args as [string, bigint]
    return { tokenId }
  },
  transform: (logs: any[]) => logs.map((log) => ({
    from: log.args.from,
    to: log.args.to,
    tokenId: log.args.tokenId,
    block: log.blockNumber,
    transactionHash: log.transactionHash,
  })),
})
```

`rpcSource` automatically chunks large block ranges so the app does not exceed provider `getLogs` limits.

#### HTTP Source

Use for REST APIs. Use `sseUrl` when the source can notify clients about live updates.

```ts
import { httpSource } from '@1001-digital/dapp-query-core'

const httpTransfers = httpSource({
  url: 'https://api.example.com/transfers',
  request: (...args: unknown[]) => {
    const [address] = args as [string]
    return { params: { address } }
  },
  transform: (data: any[]) => data.map(parseTransfer),
  sseUrl: 'https://api.example.com/transfers/stream',
})
```

#### Custom Source

Use for any async function that can return the same domain type as the other sources.

```ts
import { customSource } from '@1001-digital/dapp-query-core'

const localSource = customSource({
  id: 'local-owner-cache',
  fetch: async (...args: unknown[]) => {
    const [collection, tokenId] = args as [string, bigint]
    const res = await fetch(`/api/owner/${collection}/${tokenId}`)
    if (!res.ok) throw new Error('Local owner cache failed')
    return res.json()
  },
})
```

### Strategies

| Strategy | Behavior | Use when |
| --- | --- | --- |
| `fallback` | Try sources in order. Skip temporarily unhealthy sources. | Default for most dapp reads. |
| `race` | Fire all sources and use the first successful response. | User-facing latency matters more than request cost. |

Fallback source health:

- Sources with 3 or more consecutive failures are skipped for a short backoff window.
- Successful fetches reduce the failure count.
- Health includes failures, last failure time, average latency, and sample count.

```ts
const health = client.getSourceHealth('graphql:https://indexer.example.com')
```

### Caches

| Cache | Use |
| --- | --- |
| `memoryCache(maxSize?)` | Tests, server processes, temporary pages, no persistence. |
| `idbCache(dbName?)` | Browser persistence across page reloads. Handles BigInt serialization. |
| Custom cache | Shared storage, encrypted storage, app-specific eviction. |

Custom cache shape:

```ts
interface Cache {
  get<T>(key: string): Promise<CacheEntry<T> | undefined>
  set<T>(key: string, entry: CacheEntry<T>): Promise<void>
  delete(key: string): Promise<void>
  clear(): Promise<void>
}
```

### Vue `useQuery`

```ts
import { useQuery } from '@1001-digital/dapp-query-vue'

const { data, pending, error, revalidating, refresh } = useQuery(
  tokenTransfersQuery,
  () => [collection.value, tokenId.value] as [string, bigint],
)
```

Returned state:

| Field | Meaning |
| --- | --- |
| `data` | `Ref<T | undefined>` with the latest result. |
| `pending` | `true` only while initial data is missing. |
| `error` | Latest fetch error. |
| `revalidating` | `true` when stale data is visible while refresh runs. |
| `refresh()` | Invalidate and refetch. |

## Common Pairings With Other 1001 Blocks

- Pair with `layers.evm` for Nuxt/Vue UI and transaction flows.
- Pair with `simple-indexer` when one source should be a local browser/server index.
- Pair with Ponder and `ponder-artifacts` or `ponder-ens` when the primary source is an indexer API.
- Pair with `dweb-fetch` and `resolve-metadata` for queries that return token URI or contract URI data.
- Pair with transaction flows by invalidating relevant queries after a write completes.

## Practical Implementation Patterns

### Indexer First, RPC Fallback

```ts
const ownersQuery = {
  key: (collection: string, tokenId: bigint) =>
    `owner:${collection.toLowerCase()}:${tokenId}`,
  sources: [indexedOwner, rpcOwner],
  strategy: 'fallback' as const,
  staleTime: 60_000,
}

const owner = await queryClient.fetch(ownersQuery, collection, tokenId)
```

Use this when the indexer is fast but cannot be trusted as the only source.

### Cache Then Revalidate In Vue

```ts
const { data: transfers, pending, revalidating, refresh } = useQuery(
  tokenTransfersQuery,
  () => [props.collection, BigInt(props.tokenId)] as [string, bigint],
)
```

UI behavior:

- Show skeleton only when `pending` is true.
- Show existing data with a subtle refreshing state when `revalidating` is true.
- Provide a manual refresh button that calls `refresh`.

### Invalidate After A Transaction

```vue
<EvmTransactionFlowDialog
  :request="mint"
  @complete="refreshMintData"
/>
```

```ts
async function refreshMintData() {
  await queryClient.invalidate(tokenTransfersQuery, collection.value, tokenId.value)
}
```

When a write settles before an indexer or RPC read reflects the new state, poll deliberately instead of spinning in the component:

```ts
await queryClient.waitForChange(
  tokenTransfersQuery,
  [collection.value, tokenId.value] as [string, bigint],
  (current, previous) => current.length > (previous?.length ?? 0),
  { interval: 3_000, maxAttempts: 10 },
)
```

### Source Type Discipline

Define a domain type and force every source transform to return that type.

```ts
type Transfer = {
  from: `0x${string}`
  to: `0x${string}`
  tokenId: bigint
  block: bigint
  transactionHash: `0x${string}`
}
```

Do not return GraphQL-native strings from one source and viem-native `bigint` values from another. Convert inside each source.

## CSS/Config/Runtime/Env Details

- `idbCache` requires a browser context. Register it in a client-only Nuxt plugin.
- Query clients should be singleton-like per app runtime. Creating a new client inside every component splits cache and subscription state.
- Endpoint URLs belong in runtime config or environment variables, not in reusable query definitions.
- For multiple chains, include the chain ID in cache keys and source IDs.
- For user-specific queries, include the user address in cache keys and normalize it to lowercase.

Example Nuxt runtime config:

```ts
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      indexers: {
        mainnet: '',
        sepolia: '',
      },
    },
  },
})
```

## Gotchas And Failure Modes

- Mismatched source outputs cause subtle UI bugs. Normalize every source to the same type.
- Missing cache-key args cause data leaks across pages/users/tokens.
- RPC backfills over wide block ranges can be slow. Set `fromBlock` narrowly and keep `maxBlockRange` provider-safe.
- `race` can double or triple request volume. Use it intentionally.
- IndexedDB is unavailable during SSR. Use client plugins or memory cache on the server.
- Stale data is not an error. Show `revalidating` separately from `pending`.
- A failing primary source can be skipped temporarily. Do not assume source order means the first source always runs.

## Agent Checklist

- Decide whether the read needs resilience. If not, use direct wagmi/viem.
- Define the domain type before writing sources.
- Create one query client for the app.
- Choose `idbCache` for browser persistence or `memoryCache` for tests/server.
- Put indexer sources before RPC sources for normal dapp pages.
- Add RPC fallback when the indexer is not a hard dependency.
- Include chain, address, token ID, filters, and user scope in cache keys.
- Use `useQuery` in Vue and handle `pending`, `revalidating`, and `error` distinctly.
- Invalidate related queries after successful writes.
- Inspect source health before blaming the UI for stale data.
