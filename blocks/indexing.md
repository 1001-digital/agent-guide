# Indexing

Use this guide when an agent needs chain-event indexing, local derived state, or Ponder offchain caches.

## What To Use This For

- Lightweight EVM event indexes in a browser or server process.
- Derived state such as owners, balances, transfers, mints, votes, annotations, and activity feeds.
- Local replay after handler changes without re-fetching all logs.
- Reorg-aware event handling without running a full indexer stack.
- Ponder apps that need persistent offchain NFT metadata or ENS profile caches.

## When Not To Use It

- Do not use `simple-indexer` when you need a hosted multi-chain production indexer with GraphQL/SQL API out of the box. Use a full indexer stack for that.
- Do not resolve NFT metadata or ENS records inside Ponder event handlers. Use the Ponder helper packages to keep expensive offchain lookups separate from deterministic indexing.
- Do not use browser IndexedDB indexes as canonical backend data.
- Do not scan unbounded block ranges without a realistic `startBlock`, finality policy, and chunk size.

## Packages/Repos Involved

- Source repo: <https://github.com/1001-digital/simple-indexer>
- Source repo: <https://github.com/1001-digital/ponder-artifacts>
- Source repo: <https://github.com/1001-digital/ponder-ens>
- `@1001-digital/simple-indexer`: browser/server EVM event indexer with memory, IndexedDB, and SQLite stores.
- `@1001-digital/ponder-artifacts`: NFT collection/token metadata cache and Hono routes for Ponder.
- `@1001-digital/ponder-ens`: ENS profile cache and Hono routes for Ponder.

## Install Commands

Lightweight indexer:

```sh
pnpm add @1001-digital/simple-indexer viem
```

SQLite server store:

```sh
pnpm add better-sqlite3
```

Ponder artifact cache:

```sh
pnpm add @1001-digital/ponder-artifacts drizzle-orm hono viem pg @electric-sql/pglite
```

Ponder ENS cache:

```sh
pnpm add @1001-digital/ponder-ens drizzle-orm hono viem pg @electric-sql/pglite
```

## Minimal Setup

### Simple Indexer

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

### Ponder Artifacts

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

### Ponder ENS

ENS resolution requires an Ethereum mainnet client in Ponder config.

```ts
// ponder.config.ts
export default createConfig({
  chains: {
    ethereum: {
      id: 1,
      rpc: process.env.PONDER_RPC_URL_1!,
    },
  },
})
```

```ts
// src/api/index.ts
import { createEnsRoutes, createOffchainDb } from '@1001-digital/ponder-ens'

const { db: ensDb } = await createOffchainDb()

app.route('/ens', createEnsRoutes({
  client: publicClients.ethereum,
  db: ensDb,
}))
```

## Core APIs/Components/Contracts/Config

### Choosing The Indexing Block

| Need | Choose |
| --- | --- |
| Browser-local event index | `simple-indexer` with `createIdbStore`. |
| Server-local event index | `simple-indexer` with `createSqliteStore`. |
| Tests or temporary scripts | `simple-indexer` with `createMemoryStore`. |
| NFT metadata cache in Ponder | `ponder-artifacts`. |
| ENS profile cache in Ponder | `ponder-ens`. |
| Full production indexer API | Use a full Ponder app, then add 1001 Ponder helpers as needed. |

### Simple Indexer Config

```ts
createIndexer({
  client,
  store,
  contracts: {},
  schema: {},
  version: 1,
  pollingInterval: 12_000,
  finalityDepth: 0,
  maxChunkSize: 2_000,
})
```

| Config | Meaning |
| --- | --- |
| `client` | viem `PublicClient`. |
| `store` | Memory, IndexedDB, SQLite, or custom store. |
| `contracts` | Contract definitions and event handlers. |
| `schema` | Secondary indexes on derived tables. |
| `version` | Bump when handler logic changes to trigger replay. |
| `pollingInterval` | Live polling interval in milliseconds. |
| `finalityDepth` | Blocks to stay behind head for safer indexing. |
| `maxChunkSize` | Maximum block span per backfill request. |

### Stores

Memory:

```ts
import { createMemoryStore } from '@1001-digital/simple-indexer'

const store = createMemoryStore({
  schema: {
    owners: {
      indexes: [{ name: 'by_owner', fields: ['owner'] }],
    },
  },
})
```

IndexedDB:

```ts
import { createIdbStore } from '@1001-digital/simple-indexer'

const store = createIdbStore('my-indexer-db', {
  schema: {
    owners: {
      indexes: [{ name: 'by_owner', fields: ['owner'] }],
    },
  },
})
```

SQLite:

```ts
import { createSqliteStore } from '@1001-digital/simple-indexer/sqlite'

const store = createSqliteStore('./data.db', {
  schema: {
    owners: {
      indexes: [{ name: 'by_owner', fields: ['owner'] }],
    },
  },
})
```

Import SQLite from the `/sqlite` entrypoint so `better-sqlite3` is not bundled into browser builds.

### Contract Definition

```ts
const contracts = {
  NFT: {
    abi,
    address: '0x0000000000000000000000000000000000000000',
    startBlock: 18_000_000n,
    events: {
      async Transfer({ event, store }) {
        await store.set('transfers', `${event.block}:${event.logIndex}`, {
          from: event.args.from,
          to: event.args.to,
          tokenId: event.args.tokenId,
          block: event.block,
        })
      },
    },
  },
}
```

Handler rules:

- Store deterministic derived state only.
- Include enough data in keys to avoid collisions.
- Use string keys.
- Normalize addresses to lowercase when using them for lookup keys.
- Avoid offchain fetches inside handlers; keep handlers replayable.

### Store API

Available inside handlers and at `indexer.store`:

| API | Use |
| --- | --- |
| `get(table, key)` | Read one row. |
| `getAll(table, filter?)` | Read rows with optional `where`, `limit`, `offset`, and `index`. |
| `set(table, key, value)` | Create or replace a row. |
| `update(table, key, partial)` | Merge partial data. |
| `delete(table, key)` | Remove a row. |

Indexed query:

```ts
const rows = await indexer.store.getAll('transfers', {
  index: 'by_token',
  where: { tokenId: 42n },
  limit: 50,
})
```

Index rules:

- `index` must name an index declared for that table.
- `where` must include exactly the fields for that index.
- Exact-match lookup only, not range scans.
- Without `index`, `getAll` scans and filters in memory.

### Lifecycle And Reorgs

Simple-indexer has two layers:

- Raw event cache: decoded events fetched from chain.
- Derived state: app tables produced by handlers.

Sync lifecycle:

1. Check stored version and schema.
2. Reindex from cached events if version/schema changed.
3. Backfill historical events in chunks.
4. Poll live blocks.
5. Detect reorgs by comparing block hashes.
6. Reverse mutation logs for reorged blocks and resume from fork point.

Use:

```ts
indexer.onStatus((status) => {
  console.log(status.phase, status.currentBlock, status.latestBlock)
})

indexer.onChange((table, key) => {
  console.log(`${table}/${key} changed`)
})

await indexer.reindex()
indexer.stop()
```

### Ponder Artifacts API

Mounted routes:

| Method | Path | Behavior |
| --- | --- | --- |
| `GET` | `/artifacts/:address` | Returns cached collection info and refreshes if stale. |
| `POST` | `/artifacts/:address` | Force-refresh collection metadata. |
| `GET` | `/artifacts/:collection/:tokenId` | Returns cached token metadata and refreshes if stale. |
| `POST` | `/artifacts/:collection/:tokenId` | Force-refresh token metadata. |

Direct service:

```ts
import {
  createArtifactService,
  createOffchainDb,
} from '@1001-digital/ponder-artifacts'

const { db } = await createOffchainDb()
const artifacts = createArtifactService({ client, db })

const token = await artifacts.fetchToken(collection, tokenId)
await artifacts.updateToken(collection, tokenId)

const collectionInfo = await artifacts.fetchCollection(collection)
await artifacts.updateCollection(collection)
```

Token standard detection:

- ERC-721 via ERC-165 `0x80ac58cd`, then `tokenURI(tokenId)`.
- ERC-1155 via ERC-165 `0xd9b67a26`, then `uri(tokenId)` and `{id}` replacement.
- Unknown fallback to ERC-721 `tokenURI`.

### Ponder ENS API

Mounted routes:

| Method | Path | Behavior |
| --- | --- | --- |
| `GET` | `/ens/:id` | Returns cached profile and refreshes if stale. |
| `POST` | `/ens/:id` | Force-refresh profile. |

The `:id` can be an Ethereum address or ENS name.

Direct service:

```ts
import {
  createEnsService,
  createOffchainDb,
} from '@1001-digital/ponder-ens'

const { db } = await createOffchainDb()
const ens = createEnsService({ client, db })

const result = await ens.resolveProfile('vitalik.eth')
const profile = await ens.fetchProfile('0xd8da...')
await ens.updateProfile('0xd8da...' as `0x${string}`, 'vitalik.eth')
```

`ponder-ens` stores address, ENS name, avatar, header, description, url, email, Twitter, GitHub, and updated timestamp.

### Offchain DB Auto-Detection

Both Ponder helpers provide `createOffchainDb()`.

Behavior:

- With `DATABASE_URL` or `DATABASE_PRIVATE_URL`: connect to PostgreSQL and create the `offchain` schema/tables if missing.
- Without database URL: use PGlite and store files under `.ponder/artifacts/` or `.ponder/ens/`.

```ts
const { db } = await createOffchainDb()
const { db: pgDb } = await createOffchainDb({ databaseUrl: 'postgresql://...' })
const { db: localDb } = await createOffchainDb({ dataDir: '.data/artifacts' })
```

## Common Pairings With Other 1001 Blocks

- Pair `simple-indexer` with `dapp-query` by exposing the local index as a custom source.
- Pair `ponder-artifacts` with `resolve-metadata` and `EvmArtifact` in the frontend.
- Pair `ponder-ens` with `EvmAccount`, `EvmAvatar`, and address-heavy UIs.
- Pair `simple-indexer` with `layers.evm` transaction flows by reindexing or invalidating after writes.
- Pair Ponder helper routes with `dapp-query` GraphQL/HTTP sources for fallback-aware frontend reads.

## Practical Implementation Patterns

### Browser-Owned Holder Index

```ts
import {
  createIndexer,
  createIdbStore,
} from '@1001-digital/simple-indexer'

const store = createIdbStore('holder-index', {
  schema: {
    balances: {
      indexes: [
        { name: 'by_owner', fields: ['owner'] },
        { name: 'by_token', fields: ['tokenId'] },
      ],
    },
  },
})

const indexer = createIndexer({
  client,
  store,
  version: 1,
  finalityDepth: 6,
  contracts: {
    Edition: {
      abi,
      address,
      startBlock,
      events: {
        async TransferSingle({ event, store }) {
          const tokenId = event.args.id as bigint
          const value = event.args.value as bigint

          async function addBalance(ownerRaw: unknown, delta: bigint) {
            const owner = String(ownerRaw).toLowerCase()
            if (owner === '0x0000000000000000000000000000000000000000') return

            const key = `${owner}:${tokenId}`
            const existing = await store.get('balances', key)
            const balance =
              ((existing?.balance as bigint | undefined) ?? 0n) + delta

            if (balance <= 0n) {
              await store.delete('balances', key)
              return
            }

            await store.set('balances', key, { owner, tokenId, balance })
          }

          await addBalance(event.args.from, -value)
          await addBalance(event.args.to, value)
        },
      },
    },
  },
})
```

### Expose Simple Indexer Through Dapp Query

```ts
import { customSource } from '@1001-digital/dapp-query-core'

const localOwners = customSource({
  id: 'local-owners',
  fetch: async (collection: string, tokenId: bigint) => {
    return indexer.store.get('owners', `${collection}:${tokenId}`)
  },
})
```

### Ponder Artifact Frontend Fetch

```ts
const token = await $fetch(`/artifacts/${collection}/${tokenId}`)
```

Use the returned image/animation fields directly for UI, or pass full metadata through the metadata guide's normalization flow if your frontend needs a stricter shape.

### ENS Profile Fetch

```ts
const profile = await $fetch(`/ens/${address}`)
```

Use this for UI profile cards and owner/minter tables instead of making the browser resolve ENS repeatedly.

## CSS/Config/Runtime/Env Details

- Browser indexes must use client-only setup. IndexedDB is not available during SSR.
- SQLite indexes should live in a writable data directory and should not be bundled into client code.
- RPC URLs belong in runtime/server config.
- For Ponder ENS, always configure an Ethereum mainnet RPC, even if the indexed app targets another chain.
- For Ponder helpers, decide whether the offchain cache should use Postgres or PGlite before deployment.
- For metadata cache gateways, configure IPFS/Arweave gateway URLs in route/service config.

## Gotchas And Failure Modes

- Event handlers must be replay-safe. Avoid timestamps, random IDs, live HTTP calls, or user-specific side effects in handlers.
- Forgetting to bump `version` after handler logic changes can leave old derived state.
- Changing schema triggers reindex. Make sure handler code can replay all cached events.
- Missing secondary indexes makes filtered `getAll` scan in memory.
- Wide historical ranges can hit RPC limits. Use realistic `startBlock` and chunk size.
- Browser indexes are user-local and can be cleared by the browser.
- ENS resolution requires mainnet access.
- Ponder offchain cache data should not be modeled as deterministic onchain data.
- Stale artifact/ENS cache is usually preferable to failing the page; design UI for stale-but-present data.

## Agent Checklist

- Choose `simple-indexer` for owned local event indexes.
- Choose `ponder-artifacts` for NFT metadata caches inside Ponder.
- Choose `ponder-ens` for ENS profile caches inside Ponder.
- Pick the store deliberately: memory for tests, IndexedDB for browser, SQLite for server.
- Set `startBlock` as narrowly as possible.
- Define secondary indexes before writing UI queries.
- Normalize addresses in keys.
- Keep handlers deterministic and replay-safe.
- Bump `version` when derived-state logic changes.
- Use `onStatus` for progress UI and `onChange` for reactive updates.
- Mount Ponder helper routes under stable paths such as `/artifacts` and `/ens`.
- Configure Postgres or PGlite intentionally for offchain caches.
