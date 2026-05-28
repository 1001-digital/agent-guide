# Contract Intelligence

Use this guide when an agent needs to understand deployed contracts: proxy implementations, diamond facets, composite ABIs, selectors, or verified NatSpec documentation.

## What To Use This For

- Explorer-style contract pages.
- Indexer backends that need to resolve implementation contracts.
- ABI fetchers and decoders.
- Developer tools that inspect EIP-1967 proxies, beacons, EIP-1167 clones, Safe proxies, EIP-1822, EIP-897, or ERC-2535 diamonds.
- UIs that display function/event/error descriptions from verified NatSpec.
- Workflows that combine proxy target discovery with ABI/NatSpec enrichment.

## When Not To Use It

- Do not use `proxies` as a source-code verifier. It detects proxy patterns and targets; enrichment is your responsibility.
- Do not expect recursive proxy resolution. Proxy detection is single-hop except defined beacon resolution.
- Do not use `natspec` for unverified contracts unless you already have raw userdoc/devdoc to parse.
- Do not assume every proxy can be detected. Some contracts use custom storage, custom fallback logic, or non-standard upgrade patterns.

## Packages/Repos Involved

- Source repo: <https://github.com/1001-digital/proxies>
- Source repo: <https://github.com/1001-digital/natspec>
- `@1001-digital/proxies`: Ethereum proxy pattern detection primitives.
- `@1001-digital/natspec`: fetch and normalize NatSpec documentation from Sourcify or parse raw userdoc/devdoc.

## Install Commands

```sh
pnpm add @1001-digital/proxies @1001-digital/natspec
```

If the app enriches ABIs from Sourcify, Etherscan, Blockscout, or an internal verifier, install/configure that client separately.

## Minimal Setup

Proxy detection:

```ts
import { createProxies } from '@1001-digital/proxies'

const proxies = createProxies()

const result = await proxies.fetch(
  'https://eth.llamarpc.com',
  '0x0000000000000000000000000000000000000000',
)

if (result) {
  console.log(result.pattern)
  console.log(result.targets)
}
```

NatSpec:

```ts
import { createNatSpec } from '@1001-digital/natspec'

const natspec = createNatSpec()
const docs = await natspec.fetch(
  1,
  '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2',
)

console.log(docs.functions.deposit?.notice)
```

## Core APIs/Components/Contracts/Config

### Proxy Detection Client

```ts
import { createProxies } from '@1001-digital/proxies'

const proxies = createProxies({
  fetch: globalThis.fetch,
  enrich: async (address) => {
    return null
  },
})
```

Client methods:

| API | Use |
| --- | --- |
| `detect(rpc, address)` | On-chain probe only. Returns raw proxy targets or `null`. |
| `fetch(rpc, address, options?)` | Detect, optionally enrich targets, and build composite ABI. |

`rpc` is a JSON-RPC URL. `address` is the contract to inspect.

### Supported Proxy Patterns

Detection order is intentional. First match wins.

| Pattern | What is checked |
| --- | --- |
| ERC-2535 diamond | ERC-165/loupe support and `facets()` fallback. |
| EIP-1967 | Implementation slot, optionally admin slot. |
| EIP-1967 beacon | Beacon slot, then `implementation()` on beacon. |
| EIP-1822 | PROXIABLE storage slot. |
| EIP-1167 | Minimal proxy bytecode. |
| Safe proxy | Storage slot 0. |
| EIP-897 | `implementation()` call as last resort. |

Tradeoffs:

- Detection is single-hop. If the implementation is also a proxy, run detection again on that target intentionally.
- One failed probe should not poison the whole detection pipeline.
- Beacon is a defined two-step pattern.
- Custom proxies may return `null`.

### Enrichment

`proxies` deliberately leaves ABI/source/docs retrieval to the consuming app.

Sourcify ABI enrichment pattern:

```ts
import { createProxies } from '@1001-digital/proxies'

async function sourcifyAbi(address: string) {
  const res = await fetch(
    `https://sourcify.dev/server/v2/contract/1/${address}?fields=abi`,
  )

  if (!res.ok) return null

  const { abi } = await res.json()
  return { abi }
}

const proxies = createProxies({ enrich: sourcifyAbi })
const result = await proxies.fetch(rpcUrl, proxyAddress)
```

When targets include selectors, enrichment can filter ABI entries to live selectors before composing.

Manual enrichment:

```ts
import {
  createProxies,
  filterAbiBySelectors,
  buildCompositeAbi,
} from '@1001-digital/proxies'

const proxies = createProxies()
const raw = await proxies.detect(rpcUrl, address)

if (raw) {
  const targets = await Promise.all(raw.targets.map(async (target) => {
    const source = await loadVerifierData(target.address)

    return {
      ...target,
      abi: source?.abi && target.selectors
        ? filterAbiBySelectors(source.abi, target.selectors)
        : source?.abi,
      metadata: source?.metadata,
    }
  }))

  const compositeAbi = buildCompositeAbi(
    targets.map((target) => target.abi).filter(Boolean),
  )
}
```

### Standalone Proxy Primitives

```ts
import {
  detectProxy,
  detectDiamond,
  detectEip1967,
  decodeFacets,
  computeSelector,
  canonicalSignature,
  filterAbiBySelectors,
  buildCompositeAbi,
  enrichTargets,
} from '@1001-digital/proxies'
```

Use primitives when:

- You need only one detector.
- You already have encoded `facets()` return data.
- You need selector math for an ABI.
- You need to filter an ABI by diamond selectors.
- You need to compose ABIs without running detection.

Selector example:

```ts
const selector = computeSelector('transfer(address,uint256)')
// '0xa9059cbb'
```

### NatSpec Client

```ts
import { createNatSpec } from '@1001-digital/natspec'

const natspec = createNatSpec({
  baseUrl: 'https://sourcify.dev/server',
  fetch: globalThis.fetch,
})
```

Client methods:

| API | Use |
| --- | --- |
| `fetch(chainId, address)` | Fetch userdoc/devdoc from Sourcify and normalize. |
| `parse(userdoc, devdoc)` | Pure parse of compiler docs. |
| `toMetadata(natspec)` | Convert normalized docs to contract-metadata-compatible shape. |

Direct exports:

```ts
import {
  parse,
  toMetadata,
} from '@1001-digital/natspec'
```

### NatSpec Output

Normalized shape:

```ts
type NatSpec = {
  contract?: {
    title?: string
    author?: string
    notice?: string
    details?: string
  }
  functions: Record<string, NatSpecEntry>
  events: Record<string, NatSpecEntry>
  errors: Record<string, NatSpecEntry>
}

type NatSpecEntry = {
  signature: string
  name: string
  notice?: string
  details?: string
  params?: Record<string, string>
  returns?: Record<string, string>
}
```

Function key rules:

- Non-overloaded functions can use the bare name, such as `transfer`.
- Overloaded functions use the full Solidity signature, such as `safeTransferFrom(address,address,uint256,bytes)`.

## Common Pairings With Other 1001 Blocks

- Pair with `layers.evm` for contract explorer pages and write/read UIs.
- Pair with `dapp-query` to fetch contract intelligence through a resilient backend API.
- Pair with `simple-indexer` or Ponder when indexing proxies/facets over time.
- Pair with `natspec` plus `proxies` to show implementation ABI docs for proxies and diamond facets.
- Pair with `resolve-metadata` when contract pages include contractURI/token metadata.

## Practical Implementation Patterns

### Contract Detail Endpoint

Server route pattern:

```ts
import { createProxies } from '@1001-digital/proxies'
import { createNatSpec } from '@1001-digital/natspec'

const proxies = createProxies({ enrich: loadAbi })
const natspec = createNatSpec()

export default defineEventHandler(async (event) => {
  const chainId = Number(getRouterParam(event, 'chainId'))
  const address = getRouterParam(event, 'address') as `0x${string}`
  const rpcUrl = rpcUrlFor(chainId)

  const proxy = await proxies.fetch(rpcUrl, address)
  const docs = await natspec.fetch(chainId, proxy?.targets[0]?.address ?? address)

  return {
    address,
    proxy,
    docs,
  }
})
```

### Diamond ABI

For an ERC-2535 diamond:

1. Detect the diamond.
2. Fetch ABI for each facet address.
3. Filter facet ABI by selectors returned by the diamond.
4. Build a composite ABI.
5. Use the composite ABI in the UI for read/write forms.

```ts
const result = await proxies.fetch(rpcUrl, diamondAddress, {
  enrich: loadAbi,
})

const compositeAbi = result?.compositeAbi
```

### NatSpec Function Labels

```ts
function docsForFunction(docs: NatSpec, signature: string, name: string) {
  return docs.functions[signature] ?? docs.functions[name]
}
```

Use `notice` for user-facing labels/help text. Use `details` for developer-facing explanations.

### Safe Fallback

If detection returns `null`:

- Treat the contract as non-proxy.
- Fetch ABI/docs for the original address.
- Show "implementation not detected" rather than an error.

## CSS/Config/Runtime/Env Details

- Run proxy and NatSpec lookups server-side when RPC keys or verifier keys are private.
- Cache detection results by `chainId:address:blockTag` or by deployment immutability assumptions.
- Cache NatSpec by `chainId:address` and verifier response freshness.
- In a frontend, call your own API endpoint instead of exposing private RPC keys.
- For multi-chain apps, map every chain ID to an RPC URL and a Sourcify/verifier strategy.

Example config shape:

```ts
const chains = {
  1: {
    rpcUrl: process.env.MAINNET_RPC_URL!,
    sourcifyBaseUrl: 'https://sourcify.dev/server',
  },
  8453: {
    rpcUrl: process.env.BASE_RPC_URL!,
    sourcifyBaseUrl: 'https://sourcify.dev/server',
  },
}
```

## Gotchas And Failure Modes

- Custom proxy patterns can return `null`.
- First match wins, so edge-case contracts satisfying multiple heuristics may need manual inspection.
- Detection is not recursive. Resolve nested proxies intentionally.
- Diamond facets without verified ABI cannot contribute useful function docs.
- ABI enrichment errors are isolated per target; partial results are possible.
- NatSpec exists only when verification includes userdoc/devdoc or raw docs are provided.
- Overloaded functions require signature-aware lookup.
- `detect`/`fetch` client methods swallow RPC-layer detection errors and return `null`; direct low-level decoding can throw.

## Agent Checklist

- Decide whether the app needs proxy detection, NatSpec, or both.
- Run proxy detection before ABI-driven UI for proxy/diamond-heavy apps.
- Use `fetch` when you want detect plus enrichment; use `detect` when you only need targets.
- Bring an ABI enricher. `proxies` does not fetch source code by itself.
- Filter diamond facet ABIs by selectors before composing.
- Use `natspec.fetch(chainId, address)` for verified contracts.
- Use `parse(userdoc, devdoc)` when compiler docs are already available.
- Handle `null` proxy detection as non-fatal.
- Cache results server-side when building public UIs.
- Keep private RPC/verifier keys out of client runtime config.
