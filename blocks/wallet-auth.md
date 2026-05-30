# Wallet Auth

Use these blocks when a dapp needs a local in-app wallet or Sign-In with Ethereum sessions.

There are two separate jobs:

- `wagmi-in-app-wallet`: creates a wagmi connector backed by locally stored key material.
- `adonis-siwe`: creates and verifies SIWE messages for AdonisJS-backed sessions.

## Use This For

- Onboarding or test flows where a browser-local wallet is acceptable.
- Apps that intentionally offer an in-app wallet alongside external wallets.
- AdonisJS apps that need wallet-authenticated sessions.
- Lazy auth where users connect first and sign only for server-trusted actions.

## Do Not Use This For

- Hidden custody. Users must understand that in-app wallet key material is local.
- High-value custody where local browser storage is unacceptable.
- Replacing an app user/permission model. SIWE proves wallet ownership; the app still owns authorization.
- Signing on every wallet connection when the app only needs read-only wallet state.

## Packages

- Source repo: <https://github.com/1001-digital/wagmi-in-app-wallet>
- Source repo: <https://github.com/1001-digital/adonis-siwe>
- `@1001-digital/wagmi-in-app-wallet`: wagmi connector for a locally derived wallet.
- `@1001-digital/adonis-siwe`: AdonisJS package for SIWE message/session handling.
- Common pair: `@1001-digital/layers.evm` for frontend wallet/auth UI.

## Install

In-app wallet direct usage:

```sh
pnpm add @1001-digital/wagmi-in-app-wallet @wagmi/core viem
```

AdonisJS SIWE:

```sh
npm install @1001-digital/adonis-siwe
node ace configure @1001-digital/adonis-siwe
```

Check the package README for current Node and Adonis version requirements.

## In-App Wallet Basics

```ts
import {
  inAppWallet,
  prepareInAppWallet,
} from '@1001-digital/wagmi-in-app-wallet'
import { createConfig, http } from '@wagmi/core'
import { mainnet } from 'viem/chains'

const config = createConfig({
  chains: [mainnet],
  connectors: [inAppWallet()],
  transports: {
    [mainnet.id]: http(),
  },
})

const address = await prepareInAppWallet('your twelve word mnemonic ...')
await config.connectors[0].connect()
```

Custody model:

- The mnemonic is provided in the browser.
- A private key is derived locally.
- Key material is stored in browser storage.
- Anyone with access to that storage may access the key material.

Product copy and UI should make this clear.

With `@1001-digital/layers.evm`, enable the in-app wallet in app config and use the normal connection UI:

```ts
export default defineAppConfig({
  evm: {
    inAppWallet: {
      enabled: true,
    },
  },
})
```

```vue
<EvmConnectDialog />
```

## Adonis SIWE Basics

After configure, the package provides the SIWE config, provider registration, and default auth routes.

Typical routes:

```txt
POST /auth/siwe/message
POST /auth/siwe/verify
GET  /auth/siwe/session
POST /auth/siwe/logout
```

Configure message/domain/chain behavior in `config/siwe.ts`:

```ts
import env from '#start/env'
import { defineConfig } from '@1001-digital/adonis-siwe'

export default defineConfig({
  routes: {
    enabled: true,
    prefix: '/auth/siwe',
  },
  message: {
    domain: env.get('SIWE_DOMAIN'),
    uri: env.get('SIWE_URI'),
    statement: 'Sign in to My App',
    chainId: 1,
  },
  verification: {
    allowedChainIds: [1],
    rpcUrls: {
      1: env.get('SIWE_RPC_URL'),
    },
  },
})
```

Frontend flow:

1. Ask backend for a SIWE message or nonce, depending on the frontend component flow.
2. Ask the connected wallet to sign.
3. Send message and signature to the backend.
4. Read backend session state for server-trusted actions.

## Pair With

- [`layers.md`](layers.md) for `EvmConnectDialog`, SIWE UI, and in-app wallet setup UI.
- [`contracts.md`](contracts.md) when sessions gate minting or admin actions.
- [`data.md`](data.md) when authenticated reads need cache/fallback.

## Agent Checklist

- Do not enable in-app wallets unless the product explicitly accepts local-key custody.
- Explain local-key storage in user-facing UI.
- Use SIWE only when the app needs a server-trusted wallet session.
- Keep app authorization separate from SIWE verification.
- Confirm the frontend SIWE flow matches the backend route style.
