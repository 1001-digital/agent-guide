# Contract Intelligence

Use these blocks when an app or tool needs to understand deployed contracts: proxy targets, diamond facets, composite ABIs, or verified NatSpec docs.

## Use This For

- Explorer-style contract pages.
- ABI fetchers, decoders, and developer tools.
- Indexers that need implementation contracts rather than only proxy addresses.
- UIs that display verified function/event/error descriptions.
- Workflows that combine proxy target discovery with ABI or NatSpec enrichment.

## Do Not Use This For

- Source-code verification. Bring a verifier client such as Sourcify, Etherscan, Blockscout, or an internal service.
- Guaranteed detection of every custom proxy pattern.
- Recursive proxy resolution unless the app explicitly repeats detection on the next target.
- Unverified contracts unless the app already has raw userdoc/devdoc.

## Packages

- Source repo: <https://github.com/1001-digital/proxies>
- Source repo: <https://github.com/1001-digital/natspec>
- `@1001-digital/proxies`: Ethereum proxy pattern detection.
- `@1001-digital/natspec`: fetch and normalize NatSpec documentation from Sourcify or parse raw compiler docs.

## Install

```sh
pnpm add @1001-digital/proxies @1001-digital/natspec
```

Add any verifier or ABI source used by the app separately.

## Proxy Detection

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

Main methods:

- `detect(rpc, address)`: on-chain detection only.
- `fetch(rpc, address, options?)`: detect, optionally enrich targets, and build a richer result.

Supported patterns include ERC-2535 diamonds, EIP-1967 proxies, EIP-1967 beacons, EIP-1822, EIP-1167 clones, Safe proxies, and EIP-897.

`proxies` detects targets; it does not fetch source or ABIs by itself. Pass an `enrich` function when the app has a verifier source:

```ts
const proxies = createProxies({
  enrich: async (address) => {
    const res = await fetch(
      `https://sourcify.dev/server/v2/contract/1/${address}?fields=abi`,
    )
    if (!res.ok) return null
    const { abi } = await res.json()
    return { abi }
  },
})
```

## NatSpec

```ts
import { createNatSpec } from '@1001-digital/natspec'

const natspec = createNatSpec()

const docs = await natspec.fetch(
  1,
  '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2',
)

console.log(docs.functions.deposit?.notice)
```

Main methods:

- `fetch(chainId, address)`: fetch userdoc/devdoc from Sourcify and normalize.
- `parse(userdoc, devdoc)`: parse raw compiler docs.
- `toMetadata(natspec)`: convert normalized docs to portable metadata.

Use NatSpec for labels, helper text, contract details panels, decoded transaction previews, and developer docs.

## Pair With

- [`layers.md`](layers.md) for contract detail UIs.
- [`data.md`](data.md) for cached contract intelligence reads.
- [`indexing.md`](indexing.md) when an indexer needs implementation-aware ABI handling.
- [`contracts.md`](contracts.md) for your own deployed NFT contracts.

## Agent Checklist

- Use `proxies` to find implementation/facet/beacon targets.
- Use a separate verifier/enrichment source for ABIs and source metadata.
- Use `natspec` only when docs are verified or already available.
- Treat custom proxy detection failures as possible, not exceptional.
- Repeat detection intentionally if the implementation is also a proxy.
