# Contracts

Use the contract blocks for OpenZeppelin-based NFT contracts and simple Solidity address-type checks.

## Use This For

- ERC-721 and ERC-1155 NFT contracts built on OpenZeppelin 5.
- Limited supply, token assignment, wallet limits, soulbound behavior, metadata helpers, royalties, withdrawals, and pricing helpers.
- Lightweight address checks when their limitations are acceptable.

## Do Not Use This For

- OpenZeppelin 4.x projects.
- High-value randomness. `RandomlyAssigned` style assignment is not a secure randomness service.
- Hard anti-contract or anti-bot gates. `check-address` can be bypassed by constructor-time calls.
- Blindly combining multiple `_update`-based extensions without writing the correct override.

## Packages

- Source repo: <https://github.com/1001-digital/erc721-extensions>
- Source repo: <https://github.com/1001-digital/check-address>
- `@1001-digital/erc721-extensions`: Solidity NFT extensions for OpenZeppelin.
- `@1001-digital/check-address`: Solidity library for checking whether an address currently has code.
- Peer dependency: `@openzeppelin/contracts`.

## Install

```sh
pnpm add @1001-digital/erc721-extensions @openzeppelin/contracts
```

For address checks:

```sh
pnpm add @1001-digital/check-address
```

Compile with Solidity compatible with the imported contracts, typically `^0.8.20`.

## Minimal ERC-721 Example

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
    _safeMint(msg.sender, nextToken());
  }
}
```

## Extension Families

Supply and assignment:

- `WithLimitedSupply`: maximum supply and `nextToken()`.
- `LinearlyAssigned`: token IDs assigned linearly from a start index.
- `RandomlyAssigned`: semi-random ERC-721 token ID assignment.
- `RandomlyAssigned1155`: weighted semi-random ERC-1155 type assignment.
- `WithAdditionalMints`: expand supply after initial mint.

Sale and pricing:

- `WithSaleStart`: public sale start time.
- `HasPriceFeed`: Chainlink USD/ETH helper.
- `WithTokenPrices`: default and token-specific prices.

Wallet restrictions and transfer behavior:

- `OnePerWallet`: at most one token per address.
- `LimitedTokensPerWallet`: at most N tokens per address.
- `Soulbound`: ERC-721 mint/burn allowed, transfers blocked.
- `Soulbound1155`: ERC-1155 mint/burn allowed, transfers blocked.

Metadata:

- `WithContractMetaData`: collection-level `contractURI`.
- `WithIPFSMetaData`: token metadata in IPFS folders.
- `WithFreezableMetadata`: irreversible metadata freeze guard.
- `WithENSReverseLookup`: ENS reverse display names with short-address fallback.

Royalties and withdrawals:

- `WithFees`: ERC-2981 royalties.
- `WithMarketOffers`: offer-based market helper.
- `OnlyOnGainsCreatorFeesMarket`: creator fees only on price appreciation.
- `WithWithdrawals`: owner ETH withdrawal.
- `WithERC20Withdrawals`: recover ERC-20 tokens.

Inspect the package source before composing many extensions. OpenZeppelin 5 transfer/mint/burn behavior flows through `_update`, so multiple parents may require explicit override lists.

## CheckAddress

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

Use this only as soft policy. A contract can call during its constructor before code is stored at its address, so `isExternal` is not a hard security boundary.

## Pair With

- [`layers.md`](layers.md) for minting and transaction UI.
- [`data.md`](data.md) for resilient contract reads after deployment.
- [`indexing.md`](indexing.md) for owner, transfer, or activity indexes.
- [`metadata.md`](metadata.md) for token and contract metadata handling.
- [`contract-intelligence.md`](contract-intelligence.md) for deployed contract inspection tools.

## Agent Checklist

- Confirm OpenZeppelin 5 and Solidity `^0.8.20`.
- Pick the smallest extension set that matches the contract.
- Inspect override requirements before combining extensions.
- Avoid using assignment helpers as secure randomness.
- Treat `check-address` as bypassable soft policy.
- Add focused Solidity tests for supply, mint limits, transfers, metadata, royalties, and withdrawals used by the contract.
