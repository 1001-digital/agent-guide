# Indexing

Use the indexing blocks when an app needs event-derived data or Ponder-side offchain caches.

Choose the smallest block that fits:

- `simple-indexer` for a lightweight browser/server event index owned by the app.
- `ponder-artifacts` for NFT metadata caches inside a Ponder app.
- `ponder-ens` for ENS profile caches inside a Ponder app.

## Use This For

- Owners, transfers, balances, activity feeds, votes, annotations, or other event-derived app data.
- Browser-local or server-local indexes where a full hosted indexer is too much.
- Ponder apps that need offchain artifact or ENS caches.

## Do Not Use This For

- A simple one-off contract read.
- Canonical backend data stored only in a browser index.
- High-volume production indexing without a real block range, persistence, and finality plan.
- NFT metadata or ENS resolution inside deterministic Ponder event handlers. Use the helper caches/routes instead.

## Packages

- Source repo: <https://github.com/1001-digital/simple-indexer>
- Source repo: <https://github.com/1001-digital/ponder-artifacts>
- Source repo: <https://github.com/1001-digital/ponder-ens>
- `simple-indexer` source checkout: EVM event indexer with memory, IndexedDB, and SQLite stores.
- `@1001-digital/ponder-artifacts`: NFT collection/token metadata cache and Hono routes for Ponder.
- `@1001-digital/ponder-ens`: ENS profile cache and Hono routes for Ponder.

## Install

For `simple-indexer`, use a local built source checkout unless the app's configured registry already exposes the package:

```sh
git clone https://github.com/1001-digital/simple-indexer ../simple-indexer
pnpm --dir ../simple-indexer install
pnpm --dir ../simple-indexer build
pnpm add ../simple-indexer viem
```

Ponder artifact cache:

```sh
pnpm add @1001-digital/ponder-artifacts drizzle-orm hono viem pg @electric-sql/pglite
```

Ponder ENS cache:

```sh
pnpm add @1001-digital/ponder-ens drizzle-orm hono viem pg @electric-sql/pglite
```

## Simple Indexer Basics

Use `simple-indexer` when the app owns a small event index and wants to replay or update local derived state.

```ts
import {
  createIndexer,
  createMemoryStore,
} from '@1001-digital/simple-indexer'
import { createPublicClient, http, parseAbi } from 'viem'
import { mainnet } from 'viem/chains'

const indexer = createIndexer({
  client: createPublicClient({
    chain: mainnet,
    transport: http('https://eth.llamarpc.com'),
  }),
  store: createMemoryStore(),
  version: 1,
  contracts: {
    NFT: {
      abi: parseAbi([
        'event Transfer(address indexed from, address indexed to, uint256 indexed tokenId)',
      ]),
      address: '0x0000000000000000000000000000000000000000',
      startBlock: 18_000_000n,
      events: {
        async Transfer({ event, store }) {
          await store.set('owners', `${event.args.tokenId}`, {
            tokenId: event.args.tokenId,
            owner: event.args.to,
            block: event.block,
          })
        },
      },
    },
  },
})

await indexer.start()
```

Store choice:

- `createMemoryStore`: tests, examples, temporary server work.
- `createIdbStore`: browser persistence.
- `createSqliteStore`: small server persistence. Add `better-sqlite3`.
- Custom store: use when the app already has a persistence layer.

Bump `version` when handler logic changes and the derived data should replay.

## Ponder Artifact Cache

Use `ponder-artifacts` inside a Ponder app when token or contract metadata should be fetched and cached outside deterministic event handlers.

```ts
// src/api/index.ts
import { db, publicClients } from 'ponder:api'
import schema from 'ponder:schema'
import { Hono } from 'hono'
import { client, graphql } from 'ponder'
import {
  createArtifactRoutes,
  createOffchainDb,
} from '@1001-digital/ponder-artifacts'

const { db: artifactDb } = await createOffchainDb()
const app = new Hono()

app.route('/artifacts', createArtifactRoutes({
  client: publicClients.ethereum,
  db: artifactDb,
}))

app.use('/sql/*', client({ db, schema }))
app.use('/', graphql({ db, schema }))

export default app
```

## Ponder ENS Cache

Use `ponder-ens` inside a Ponder app when the frontend needs cached ENS names, avatars, or profile data.

```ts
import { createEnsRoutes, createOffchainDb } from '@1001-digital/ponder-ens'

const { db: ensDb } = await createOffchainDb()

app.route('/ens', createEnsRoutes({
  client: publicClients.ethereum,
  db: ensDb,
}))
```

ENS resolution needs an Ethereum mainnet public client in Ponder config.

## Common Pairings

- [`data.md`](data.md): expose an index as one dapp-query source with RPC fallback.
- [`layers.md`](layers.md): render owners, feeds, and transaction-triggered invalidation in Nuxt.
- [`metadata.md`](metadata.md): normalize token metadata before caching or rendering.
- [`contract-intelligence.md`](contract-intelligence.md): inspect proxy implementations for contract-aware indexing tools.

## Agent Checklist

- Use indexing only when the app needs derived event data or Ponder offchain caches.
- Choose browser, server, or Ponder ownership before coding.
- Set a realistic `startBlock`.
- Keep event handlers deterministic.
- Use Ponder helpers for metadata/ENS caches instead of resolving them in event handlers.
- Pair with dapp-query when the frontend needs fallback or reactive reads.
