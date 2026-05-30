# Data

Use `dapp-query` when a dapp read should not depend on one fragile endpoint. It gives agents one query interface over indexed data, RPC reads, HTTP APIs, cache, and live updates.

## Use This For

- Reads that should try an indexer first and fall back to RPC.
- Reads that need browser cache, stale-while-revalidate, or live updates.
- Vue apps that need reactive query state.
- App data that can come from multiple sources but should have one domain shape.

## Do Not Use This For

- A one-off direct contract read where `readContract` is enough.
- Building a full indexer. Use [`indexing.md`](indexing.md) when the app owns derived event tables.
- Combining sources that return different shapes for the same query.

## Packages

- Source repo: <https://github.com/1001-digital/dapp-query>
- `@1001-digital/dapp-query-core`: query client, sources, strategies, and caches.
- `@1001-digital/dapp-query-vue`: Vue plugin and `useQuery`.
- `viem`: needed when using RPC sources.

## Install

```sh
pnpm add @1001-digital/dapp-query-core @1001-digital/dapp-query-vue viem
```

Core-only usage:

```sh
pnpm add @1001-digital/dapp-query-core viem
```

## Minimal Setup

Create one query client per app:

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

Register it in Vue:

```ts
import { dappQueryPlugin } from '@1001-digital/dapp-query-vue'
import { queryClient } from './query-client'

app.use(dappQueryPlugin, queryClient)
```

In Nuxt, use a client plugin if the cache uses IndexedDB:

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

## Query Shape

A query defines a stable cache key, compatible sources, a strategy, cache timing, and optional final transform:

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

Keep every source returning the same domain type. Put source-specific parsing inside the source transform, then use query-level `transform` only for shared shaping such as sorting or dedupe.

## Sources

Use `graphqlSource` for Ponder, The Graph, or another GraphQL indexer:

```ts
import { graphqlSource } from '@1001-digital/dapp-query-core'

const indexedTransfers = graphqlSource({
  endpoints: ['https://indexer.example/graphql'],
  query: `query($collection: String!, $tokenId: BigInt!) {
    transfers(where: { collection: $collection, tokenId: $tokenId }) {
      items { from to tokenId block transactionHash }
    }
  }`,
  variables: (...args: unknown[]) => {
    const [collection, tokenId] = args as [string, bigint]
    return { collection: collection.toLowerCase(), tokenId: tokenId.toString() }
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

Use `rpcSource` for direct event-log reads through viem:

```ts
import { rpcSource } from '@1001-digital/dapp-query-core'

const rpcTransfers = rpcSource({
  client: publicClient,
  event: transferEvent,
  address: collectionAddress,
  fromBlock: 18_000_000n,
  filter: (...args: unknown[]) => {
    const [, tokenId] = args as [string, bigint]
    return { tokenId }
  },
  transform: (logs: any[]) => logs.map(parseTransferLog),
})
```

Use `httpSource` for REST APIs and `customSource` for app-specific async fetchers.

In strict TypeScript, keep source callback parameters broad (`...args: unknown[]`) and cast the tuple inside the callback.

## Reading Data

One-shot fetch:

```ts
const transfers = await queryClient.fetch(
  tokenTransfersQuery,
  collection,
  tokenId,
)
```

Vue reactive read:

```ts
import { useQuery } from '@1001-digital/dapp-query-vue'

const { data, pending, error, revalidating, refresh } = useQuery(
  tokenTransfersQuery,
  () => [collection.value, tokenId.value] as [string, bigint],
)
```

After a write, invalidate the query or wait until the expected change is visible:

```ts
await queryClient.invalidate(tokenTransfersQuery, collection, tokenId)
```

## Basics To Choose

- Strategy: use `fallback` by default. Use `race` only when latency matters more than request cost.
- Cache: use `idbCache` for browser persistence and `memoryCache` for tests, scripts, or server-only temporary data.
- Sources: order sources from most preferred to least preferred.
- Keys: include every argument that changes the result.

## Pair With

- [`layers.md`](layers.md) for Nuxt UI and transaction flows.
- [`indexing.md`](indexing.md) when the app needs its own event-derived source.
- [`metadata.md`](metadata.md) when query results include token URI or media fields.

## Agent Checklist

- Use dapp-query only when fallback, cache, live updates, or multiple sources matter.
- Define one domain type per query.
- Keep cache keys stable and complete.
- Use source transforms for source-specific parsing.
- Register the Vue plugin once.
- In Nuxt, use a client plugin for IndexedDB-backed cache.
