# Wallet Auth

Use this guide when an agent needs local in-app wallet behavior or Sign-In with Ethereum sessions.

## What To Use This For

- A wagmi connector backed by a locally derived mnemonic wallet.
- Onboarding or special-purpose flows where a browser-local wallet is acceptable.
- Test/dev wallets without browser extensions.
- AdonisJS server-side SIWE message creation, nonce handling, signature verification, and wallet sessions.
- Lazy wallet auth where users connect first and sign only when a trusted server action requires it.

## When Not To Use It

- Do not use `wagmi-in-app-wallet` without explaining local key custody to users.
- Do not treat `localStorage` as secure key storage for high-value custody.
- Do not use in-app wallet setup when the product requires hardware wallets, extension wallets, mobile wallets, or institutional custody.
- Do not call SIWE on normal wallet connect if the action does not need a server-trusted session.
- Do not use `adonis-siwe` as a user system by itself. It verifies wallet ownership and stores a wallet session; your app owns user linking and app authorization.

## Packages/Repos Involved

- Source repo: <https://github.com/1001-digital/wagmi-in-app-wallet>
- Source repo: <https://github.com/1001-digital/adonis-siwe>
- `@1001-digital/wagmi-in-app-wallet`: wagmi connector that derives a wallet from a BIP39 mnemonic and stores the private key locally.
- `@1001-digital/adonis-siwe`: AdonisJS 7 package for lazy SIWE sessions.
- Common pair: `@1001-digital/layers.evm` for wallet/auth UI components.

## Install Commands

In-app wallet:

```sh
pnpm add @1001-digital/wagmi-in-app-wallet @wagmi/core viem
```

AdonisJS SIWE:

```sh
npm install @1001-digital/adonis-siwe
node ace configure @1001-digital/adonis-siwe
```

`adonis-siwe` expects Node.js 24 or newer and AdonisJS 7.

## Minimal Setup

### In-App Wallet Connector

```ts
import {
  inAppWallet,
  prepareInAppWallet,
} from '@1001-digital/wagmi-in-app-wallet'
import { createConfig, http } from '@wagmi/core'
import { mainnet } from 'viem/chains'

const config = createConfig({
  chains: [mainnet],
  connectors: [
    inAppWallet(),
  ],
  transports: {
    [mainnet.id]: http(),
  },
})

const address = await prepareInAppWallet('your twelve word mnemonic ...')
await config.connectors[0].connect()
```

Custom storage key:

```ts
inAppWallet({
  storageKey: 'my-app:wallet-pk',
})
```

Current API caveat: `prepareInAppWallet(mnemonic)` writes to the default storage key, `evm:in-app-wallet-pk`. If you pass a custom `storageKey` to `inAppWallet`, seed that key yourself or keep the default connector key so `connect()` can find the prepared private key.

### Adonis SIWE

After install:

```sh
node ace configure @1001-digital/adonis-siwe
```

The configure hook creates `config/siwe.ts`, registers the provider, and adds basic environment variables.

Default routes:

```txt
POST /auth/siwe/message
POST /auth/siwe/verify
GET  /auth/siwe/session
POST /auth/siwe/logout
```

The package expects `@adonisjs/session` middleware to be enabled.

## Core APIs/Components/Contracts/Config

### `wagmi-in-app-wallet`

API:

| API | Use |
| --- | --- |
| `inAppWallet(parameters?)` | Create a wagmi connector. |
| `prepareInAppWallet(mnemonic)` | Derive private key, store it locally, return wallet address. |

Parameters:

```ts
type InAppWalletParameters = {
  storageKey?: string
}
```

Default storage key:

```txt
evm:in-app-wallet-pk
```

Custody model:

- The mnemonic is provided in the browser.
- A private key is derived locally.
- The private key is stored in `localStorage`.
- No extension or external signer is required.
- Anyone with access to the browser storage can access the key material.

Use product copy and UI that makes this explicit.

### `layers.evm` In-App Wallet UI

When using `@1001-digital/layers.evm`, enable the in-app wallet in app config:

```ts
export default defineAppConfig({
  evm: {
    inAppWallet: {
      enabled: true,
    },
  },
})
```

Then use normal layer connection components:

```vue
<EvmConnectDialog />
```

The layer includes `EvmInAppWalletSetup` and `EvmSeedPhraseInput` for setup flows.

### Adonis SIWE Config

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

Config decisions:

| Config | Use |
| --- | --- |
| `routes.enabled` | Disable if mounting custom controllers. |
| `routes.prefix` | Route namespace. |
| `message.domain` | SIWE domain. Should match the site origin/domain policy. |
| `message.uri` | App URI shown in SIWE message. |
| `message.statement` | Human-readable purpose. |
| `message.chainId` | Default chain ID. |
| `verification.allowedChainIds` | Restrict accepted chain IDs. |
| `verification.rpcUrls` | Enables contract-wallet verification paths for configured chains. |

With RPC URL configured and contract wallets enabled, verification supports EOA, ERC-1271 smart contract wallets, and EIP-6492 pre-deployed wallets through the underlying SIWE verifier path.

### Adonis Hooks

Use hooks to link wallets to users, issue app-specific tokens, or customize serialized session data.

```ts
export default defineConfig({
  hooks: {
    async onVerified(ctx, wallet) {
      // Link wallet to user, refresh profile data, issue an app token, etc.
      return { linked: true }
    },
    async serializeSession(ctx, wallet) {
      return {
        address: wallet.address,
        chainId: wallet.chainId,
        authenticatedAt: wallet.authenticatedAt,
        expiresAt: wallet.expiresAt,
      }
    },
  },
})
```

`onVerified` return data is included in `POST /auth/siwe/verify` as `data`.

### Reading Sessions In Controllers

```ts
import siwe from '@1001-digital/adonis-siwe/services/main'

export default class ProtectedController {
  async handle(ctx) {
    const wallet = await siwe.requireSession(ctx)

    return {
      address: wallet.address,
      chainId: wallet.chainId,
    }
  }
}
```

Use `requireSession` for routes that must have a verified wallet session. Use `siwe.getSession(ctx)` for optional personalization.

## Common Pairings With Other 1001 Blocks

- Pair `wagmi-in-app-wallet` with `layers.evm` when the UI should expose in-app wallet setup.
- Pair `adonis-siwe` with `layers.evm` wallet connection UI. Use the custom Adonis message/sign/verify flow below; the layer SIWE components expect a nonce endpoint, while `adonis-siwe` returns a complete SIWE message.
- Pair SIWE session addresses with `ponder-ens` for profile display.
- Pair authenticated writes with `EvmTransactionFlowDialog`.
- Pair backend-protected NFT actions with `erc721-extensions` contracts and wallet sessions.

## Practical Implementation Patterns

### Lazy SIWE Flow

Do not sign on connect. Sign only when the user starts an action that needs server-trusted wallet ownership.

```ts
import { signMessage } from '@wagmi/core'

const wagmiConfig = useConfig()

const { message } = await api.post('/auth/siwe/message', {
  address,
  chainId: 1,
})

const signature = await signMessage(wagmiConfig, { message })

await api.post('/auth/siwe/verify', {
  message,
  signature,
})
```

After verification, protected API routes can call `siwe.requireSession(ctx)`.

### Layers Wallet With Adonis Backend

```vue
<template>
  <EvmConnectDialog />

  <Button
    class="primary"
    :disabled="!account.address.value"
    @click="signIn"
  >
    Sign in to manage
  </Button>
</template>

<script setup lang="ts">
import { signMessage } from '@wagmi/core'

const account = useAccount()
const wagmiConfig = useConfig()

async function signIn() {
  if (!account.address.value || !account.chainId.value) return

  const response = await $fetch<{ message: string }>('/auth/siwe/message', {
    method: 'POST',
    body: {
      address: account.address.value,
      chainId: account.chainId.value,
    },
  })

  const signature = await signMessage(wagmiConfig, {
    message: response.message,
  })

  await $fetch('/auth/siwe/verify', {
    method: 'POST',
    body: {
      message: response.message,
      signature,
    },
  })

  await refreshNuxtData('session')
}
</script>
```

### In-App Wallet Onboarding

Use only for flows where local browser custody is acceptable:

1. Explain that the wallet key is stored locally.
2. Ask the user to create or enter a mnemonic.
3. Call `prepareInAppWallet(mnemonic)`.
4. Connect using the in-app wallet connector.
5. Provide a visible path to export/back up or reset, if the product supports that.

```ts
async function setupWallet(mnemonic: string) {
  const address = await prepareInAppWallet(mnemonic)
  await connect({ connector: inAppConnector })
  return address
}
```

### Session-Gated Server Action

```ts
export default class MintReservationsController {
  async store(ctx) {
    const wallet = await siwe.requireSession(ctx)

    return reserveMint({
      address: wallet.address,
      chainId: wallet.chainId,
    })
  }
}
```

Use SIWE for server trust. Use wallet transaction signatures for on-chain actions.

## CSS/Config/Runtime/Env Details

### Adonis Environment

Typical variables:

```txt
SIWE_DOMAIN=example.com
SIWE_URI=https://example.com
SIWE_RPC_URL=https://eth.llamarpc.com
```

Keep private RPC keys server-side. Do not pass private SIWE RPC URLs to public frontend runtime config.

### Frontend Runtime

For layers apps, configure wallet behavior in `app.config.ts`:

```ts
export default defineAppConfig({
  evm: {
    title: 'My App',
    defaultChain: 'mainnet',
    inAppWallet: {
      enabled: false,
    },
  },
})
```

Enable in-app wallets only per product decision.

### Session/CORS

When the frontend and Adonis backend are on different origins:

- Configure session cookies for the deployment domain.
- Send credentials on fetch requests if required.
- Ensure SIWE `domain` and `uri` match the user-visible app.
- Keep nonce/message endpoints protected against CSRF according to the app's session strategy.

## Gotchas And Failure Modes

- `localStorage` key material is not hardware-backed or encrypted by default.
- Losing browser storage can lose the in-app wallet unless the user has backed up the mnemonic.
- In-app wallet accounts are not the same UX/security model as extension wallets.
- Wallet connect and SIWE are separate. Connected does not mean authenticated.
- SIWE message domain/URI mismatch causes verification failures.
- Missing session middleware in Adonis breaks route/session behavior.
- Missing RPC URL can limit smart-contract-wallet verification.
- Allowed chain IDs must include the chain used in the signed message.
- Do not create users implicitly unless `onVerified` intentionally links or creates accounts.

## Agent Checklist

- Decide whether the app needs wallet connection only or server-trusted auth.
- Use `layers.evm` for normal wallet UI; use layer SIWE components only with nonce-based backends.
- Enable in-app wallet only when local browser custody is acceptable.
- Explain local key storage clearly in product UI.
- Use `prepareInAppWallet` before connecting the in-app wallet connector, and keep the default storage key unless you intentionally seed a custom key.
- Configure Adonis SIWE with domain, URI, statement, allowed chain IDs, and RPC URLs.
- Keep private RPC URLs server-side.
- Use lazy SIWE: ask for a signature only when a trusted backend action requires it.
- For `adonis-siwe`, sign the complete message returned by `/auth/siwe/message`; do not pass that message into nonce-based layer SIWE components.
- Use hooks to link wallets to users or return app-specific data.
- Read sessions in protected controllers with `siwe.requireSession(ctx)`.
- Test EOA, contract wallet if supported, wrong chain, expired session, logout, and domain mismatch.
