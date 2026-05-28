# Contracts

Use this guide when an agent needs 1001 Solidity building blocks for NFT contracts and address-type checks.

## What To Use This For

- OpenZeppelin-based ERC-721 and ERC-1155 NFT contracts.
- Limited supply, linear token assignment, semi-random token assignment, one-per-wallet, per-wallet limits, soulbound tokens, royalties, markets, pricing, withdrawals, metadata, and ENS display helpers.
- Simple address type checks inside Solidity when the known limitation is acceptable.

## When Not To Use It

- Do not use these contracts with OpenZeppelin 4.x. `erc721-extensions` targets OpenZeppelin 5.x and Solidity `^0.8.20`.
- Do not use on-chain randomness from `RandomlyAssigned` or `RandomlyAssigned1155` for high-value cryptographic randomness. Use a secure randomness service when fairness/security requires it.
- Do not rely on `check-address` to block all contracts. Contract-constructor calls can bypass address-type checks.
- Do not combine multiple `_update`-based extensions without explicitly writing the correct override.

## Packages/Repos Involved

- Source repo: <https://github.com/1001-digital/erc721-extensions>
- Source repo: <https://github.com/1001-digital/check-address>
- `@1001-digital/erc721-extensions`: composable Solidity extensions for OpenZeppelin ERC-721 and ERC-1155.
- `@1001-digital/check-address`: Solidity library for checking whether an address currently has code.
- Peer dependency: `@openzeppelin/contracts`.

## Install Commands

```sh
pnpm add @1001-digital/erc721-extensions @openzeppelin/contracts
```

For address type checks:

```sh
pnpm add @1001-digital/check-address
```

Hardhat projects should compile with Solidity compatible with the imported contracts, typically `^0.8.20`.

## Minimal Setup

### ERC-721 Extension

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@1001-digital/erc721-extensions/contracts/WithLimitedSupply.sol";

contract ExampleToken is ERC721, WithLimitedSupply {
  constructor()
    ERC721("Example", "EX")
    WithLimitedSupply(1000)
  {}

  function mint() external ensureAvailability {
    uint256 tokenId = nextToken();
    _safeMint(msg.sender, tokenId);
  }
}
```

### Address Type Check

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@1001-digital/check-address/contracts/CheckAddress.sol";

contract Gate {
  function enter() external view {
    require(CheckAddress.isExternal(msg.sender), "Contracts not allowed");
  }
}
```

Security rule: only use this if bypass during construction is acceptable.

## Core APIs/Components/Contracts/Config

### OpenZeppelin 5 Expectations

`erc721-extensions` targets OpenZeppelin 5.x. Agents should expect:

- Solidity `^0.8.20`.
- Custom errors instead of `require` strings in extension internals.
- No `Counters` dependency.
- Transfer/mint/burn hooks implemented through OpenZeppelin 5 `_update`.
- Explicit override lists when multiple parents implement `_update` or `supportsInterface`.

### Supply And Assignment

| Contract | Use |
| --- | --- |
| `WithLimitedSupply` | Track a maximum supply and expose `nextToken()`. |
| `LinearlyAssigned` | Assign token IDs linearly from a configured start index. |
| `RandomlyAssigned` | Semi-randomly assign ERC-721 token IDs from a fixed supply. |
| `RandomlyAssigned1155` | Weighted semi-random ERC-1155 token type assignment. |
| `WithAdditionalMints` | Expand supply after initial mint and update metadata CID. |

Limited supply:

```solidity
contract RareToken is ERC721, WithLimitedSupply {
  constructor()
    ERC721("RareToken", "RARE")
    WithLimitedSupply(1000)
  {}

  function mint() external ensureAvailability {
    _safeMint(msg.sender, nextToken());
  }
}
```

Multiple mints:

```solidity
function mintMany(uint256 amount) external ensureAvailabilityFor(amount) {
  for (uint256 i = 0; i < amount; i++) {
    _safeMint(msg.sender, nextToken());
  }
}
```

Semi-random ERC-721:

```solidity
contract RandomToken is ERC721, RandomlyAssigned {
  constructor()
    ERC721("RandomToken", "RND")
    RandomlyAssigned(1000, 1)
  {}

  function mint() external ensureAvailability {
    _safeMint(msg.sender, nextToken());
  }
}
```

### Sale Timing And Pricing

| Contract | Use |
| --- | --- |
| `WithSaleStart` | Owner-controlled public sale start time. |
| `HasPriceFeed` | Chainlink USD to ETH pricing helper. |
| `WithTokenPrices` | Default and token-specific prices. |

Sale start:

```solidity
contract TimedToken is ERC721, WithSaleStart {
  constructor()
    ERC721("TimedToken", "TIME")
    Ownable(msg.sender)
    WithSaleStart(1735686000)
  {}

  function mint() external afterSaleStart {
    _safeMint(msg.sender, 1);
  }
}
```

Token prices:

```solidity
contract PricedToken is ERC721, WithTokenPrices {
  uint256 private _tokenId;

  constructor()
    ERC721("PricedToken", "PRICE")
    Ownable(msg.sender)
    WithTokenPrices(0.08 ether)
  {}

  function mint() external payable meetsPrice(_tokenId + 1) {
    _tokenId++;
    _safeMint(msg.sender, _tokenId);
  }
}
```

### Holder Restrictions And Soulbound Tokens

| Contract | Use |
| --- | --- |
| `OnePerWallet` | Every address can hold at most one token. |
| `LimitedTokensPerWallet` | Every address can hold at most N tokens. |
| `Soulbound` | ERC-721 mint/burn allowed, transfers blocked. |
| `Soulbound1155` | ERC-1155 mint/burn allowed, transfers blocked. |

OpenZeppelin 5 `_update` override pattern:

```solidity
contract OneForAllToken is ERC721, OnePerWallet {
  constructor()
    ERC721("OneForAllToken", "OFA")
  {}

  function _update(address to, uint256 tokenId, address auth)
    internal
    override(ERC721, OnePerWallet)
    returns (address)
  {
    return OnePerWallet._update(to, tokenId, auth);
  }
}
```

Soulbound ERC-721:

```solidity
contract SoulboundToken is ERC721, ERC721Burnable, Soulbound {
  constructor()
    ERC721("SoulboundToken", "SBT")
  {}

  function _update(address to, uint256 tokenId, address auth)
    internal
    override(ERC721, Soulbound)
    returns (address)
  {
    return Soulbound._update(to, tokenId, auth);
  }
}
```

### Metadata

| Contract | Use |
| --- | --- |
| `WithContractMetaData` | Collection-level `contractURI` metadata. |
| `WithIPFSMetaData` | Token metadata hosted in IPFS folders. |
| `WithFreezableMetadata` | Irreversibly freeze metadata updates. |
| `WithENSReverseLookup` | On-chain ENS reverse display names with short-address fallback. |

Contract metadata:

```solidity
contract Token is ERC721, WithContractMetaData {
  constructor()
    ERC721("Token", "TKN")
    Ownable(msg.sender)
    WithContractMetaData("ipfs://Qm.../metadata.json")
  {}
}
```

IPFS metadata:

```solidity
contract CleanToken is ERC721, WithIPFSMetaData {
  constructor()
    ERC721("CleanToken", "CLEAN")
    WithIPFSMetaData("Qm0123456789...")
  {}
}
```

Metadata-freeze pattern:

```solidity
contract FreezableToken is ERC721, WithFreezableMetadata {
  string private _baseTokenURI;

  constructor()
    ERC721("FreezableToken", "FRZ")
    Ownable(msg.sender)
  {}

  function setBaseURI(string memory nextBaseURI) external onlyOwner unfrozen {
    _baseTokenURI = nextBaseURI;
  }
}
```

### Royalties, Markets, And Withdrawals

| Contract | Use |
| --- | --- |
| `WithFees` | ERC-2981 royalties. |
| `WithMarketOffers` | Built-in offer-based marketplace. |
| `OnlyOnGainsCreatorFeesMarket` | Creator fees only on price appreciation. |
| `WithWithdrawals` | Owner ETH withdrawal. |
| `WithERC20Withdrawals` | Recover ERC-20 tokens sent to the contract. |

Royalty support-interface pattern:

```solidity
contract RoyaltyToken is ERC721, WithFees {
  constructor(address payable beneficiary)
    ERC721("RoyaltyToken", "ROY")
    Ownable(msg.sender)
    WithFees(beneficiary, 500)
  {}

  function supportsInterface(bytes4 interfaceId)
    public
    view
    override(WithFees, ERC721)
    returns (bool)
  {
    return WithFees.supportsInterface(interfaceId);
  }
}
```

Withdrawal:

```solidity
contract TreasuryToken is ERC721, WithWithdrawals {
  uint256 public constant PRICE = 0.05 ether;
  uint256 private _tokenId;

  constructor()
    ERC721("TreasuryToken", "TRY")
    Ownable(msg.sender)
  {}

  function mint() external payable {
    require(msg.value == PRICE, "Wrong payment");
    _tokenId++;
    _safeMint(msg.sender, _tokenId);
  }

  receive() external payable {}
}
```

### CheckAddress

Use `CheckAddress` to distinguish addresses with code from addresses without code at the time of the call.

Expected pattern:

```solidity
import "@1001-digital/check-address/contracts/CheckAddress.sol";

if (CheckAddress.isContract(account)) {
  // contract path
}

if (CheckAddress.isExternal(account)) {
  // EOA-like path
}
```

Important caveat:

- A contract can call while its constructor is running, before code is stored at its address.
- Therefore `isExternal` can be bypassed by constructor-time calls.
- This makes it unsuitable as a hard anti-contract or anti-bot security boundary.

## Common Pairings With Other 1001 Blocks

- Pair `WithIPFSMetaData` and `WithContractMetaData` with `dweb-fetch` and `resolve-metadata` in frontends.
- Pair ERC-721 events with `simple-indexer` for local owner/balance/activity indexes.
- Pair ERC-721 metadata and owner data with `ponder-artifacts` in Ponder apps.
- Pair contract UIs with `layers.evm` transaction flows.
- Pair contract explorer pages with `proxies` and `natspec`.
- Pair SIWE-gated mint/admin endpoints with `adonis-siwe`.

## Practical Implementation Patterns

### Mint With Limited Supply

```solidity
contract MintableToken is ERC721, WithLimitedSupply, WithWithdrawals {
  uint256 public constant PRICE = 0.05 ether;

  constructor()
    ERC721("MintableToken", "MINT")
    Ownable(msg.sender)
    WithLimitedSupply(1000)
  {}

  function mint() external payable ensureAvailability {
    require(msg.value == PRICE, "Wrong payment");

    uint256 tokenId = nextToken();
    _safeMint(msg.sender, tokenId);
  }
}
```

When combining extensions, check whether multiple parents implement the same hook. If they do, write an explicit override that calls the intended extension implementation.

### Combining `_update` Extensions

When two extensions both use `_update`, write the combined override only after checking both parent implementations in the installed package. The safe default for agents is to keep the first version of a contract simple, then add one hook-based extension at a time with tests.

Agents must verify exact inherited behavior before finalizing complex multi-extension overrides. Hook composition is the highest-risk part of this block.

### Metadata Lifecycle

Recommended lifecycle:

1. Deploy with placeholder or pre-reveal metadata when rarity sniping matters.
2. Publish final IPFS folder after sale/reveal.
3. Update base CID if the extension supports it.
4. Freeze metadata with `WithFreezableMetadata` when final.
5. Frontend reads `tokenURI` or `contractURI`, then uses the metadata guide to normalize and render.

### Address Check As Soft Policy

Acceptable use cases:

- UI hints or analytics.
- Soft allowlist/denylist policy where bypass is tolerable.
- Branching behavior that does not protect funds or core security.

Avoid use cases:

- Preventing all contract participation.
- Protecting a mint from bots.
- Enforcing compliance-critical identity boundaries.
- Guarding funds against smart contract callers.

## CSS/Config/Runtime/Env Details

- Solidity compiler version should be compatible with `^0.8.20`.
- OpenZeppelin dependency should be `^5.2.0` or compatible with the installed extension package.
- Hardhat/Foundry remappings must resolve `@1001-digital/erc721-extensions/contracts/...`.
- If using Chainlink price feeds through `HasPriceFeed`, configure feed addresses per deployment chain.
- If using ENS reverse lookup, confirm ENS registry availability on the target chain; fallback display should still work.
- Frontends should not assume metadata URLs are HTTP URLs. Expect `ipfs://` and use the metadata block.

## Gotchas And Failure Modes

- Calling `nextToken()` more than once inside a mint function can skip/alter intended IDs depending on implementation. Store it in a local variable when in doubt.
- Multi-mint functions must use `ensureAvailabilityFor(amount)`, not just `ensureAvailability`.
- Random assignment is not secure randomness.
- OpenZeppelin 5 `_update` overrides must include all relevant parents.
- `supportsInterface` must include all parents that add interfaces, especially royalties.
- `WithMarketOffers` offers can remain active until cancelled or token movement; approval revocation alone may not clear internal offers.
- Publishing full metadata before sale can expose rarity distributions.
- `CheckAddress.isExternal` can be bypassed by constructor calls.
- Library linking may be needed for `check-address` depending on the toolchain.

## Agent Checklist

- Confirm the contract project uses OpenZeppelin 5.x and Solidity `^0.8.20`.
- Pick only the extensions needed for the product behavior.
- Write explicit `_update` and `supportsInterface` overrides where inheritance requires them.
- Use `ensureAvailabilityFor(amount)` for batch mints.
- Use secure randomness if high-value randomness matters.
- Treat metadata reveal/freeze as a product lifecycle decision.
- Configure chain-specific feeds, registries, and metadata gateways.
- Use `check-address` only as a soft policy unless constructor bypass is harmless.
- Add tests for minting, transfers, burns, supply limits, wallet limits, royalty interface support, metadata changes, withdrawals, and failure cases.
- Pair frontend metadata handling with `dweb-fetch` and `resolve-metadata`.
