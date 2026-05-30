# Layers

Use `@1001-digital/layers.evm` as the default foundation for Nuxt/Vue EVM dapps. It brings the base UI layer, EVM components, wallet setup, transaction UX, and shared CSS system into one Nuxt layer.

## Use This For

- Nuxt/Vue apps that connect wallets or interact with EVM contracts.
- Wallet connection, account/profile UI, network switching, SIWE UI, and transaction flows.
- Base app UI: shell, sidebar, forms, buttons, dialogs, dropdowns, toasts, tags, cards, and tooltips.
- Shared 1001 styling: CSS variables, component tokens, cascade layers, and utility classes.

## Do Not Use This For

- Non-EVM Nuxt apps that only need base UI. Use `@1001-digital/layers.base`.
- Backend auth by itself. The layer gives UI and signing helpers; the app/backend still owns nonce storage, session persistence, and verification.
- Direct package-level component usage in Nuxt unless intentionally bypassing the layer.

## Packages

- Source repo: <https://github.com/1001-digital/layers>
- `@1001-digital/layers.evm`: Nuxt layer for EVM apps. Extends the base layer.
- `@1001-digital/layers.base`: Nuxt layer for base app UI.
- `@1001-digital/components.evm`: EVM Vue components and composables used by the layer.
- `@1001-digital/components`: Base Vue component library used by the layer.
- `@1001-digital/styles`: CSS reset, variables, component tokens, and utilities.

## Install

For a Nuxt EVM app:

```sh
pnpm add @1001-digital/layers.evm
```

That is enough to use the layer. Do not install `@1001-digital/components`, `@1001-digital/components.evm`, `@1001-digital/styles`, wagmi, or viem just to use `@1001-digital/layers.evm`.

Only add lower-level packages when app source imports them directly. For example, custom contract helpers that import `@wagmi/core` or `viem` should declare those packages in the app:

```sh
pnpm add @wagmi/core viem
```

For a Nuxt app that only needs base UI:

```sh
pnpm add @1001-digital/layers.base
```

## Minimal Nuxt Setup

Use the EVM layer in `nuxt.config.ts`. Do not also extend the base layer; EVM already does that.

```ts
export default defineNuxtConfig({
  extends: ['@1001-digital/layers.evm'],
  runtimeConfig: {
    public: {
      evm: {
        walletConnectProjectId: '',
        chains: {
          mainnet: {
            rpcs: '',
          },
          sepolia: {
            rpcs: '',
          },
        },
        ens: {
          indexers: '',
        },
      },
    },
  },
})
```

Use `app.config.ts` for non-secret app behavior:

```ts
export default defineAppConfig({
  evm: {
    title: 'My dapp',
    appLogoUrl: '/icon.png',
    defaultChain: 'mainnet',
    chains: {
      mainnet: {
        id: 1,
        blockExplorer: 'https://etherscan.io',
      },
      sepolia: {
        id: 11155111,
        blockExplorer: 'https://sepolia.etherscan.io',
      },
    },
    ens: {
      mode: 'indexer',
    },
    ipfsGateway: 'https://ipfs.io/ipfs/',
    arweaveGateway: 'https://arweave.net/',
    inAppWallet: {
      enabled: false,
    },
  },
})
```

`defaultChain` must match a key in `chains`. Runtime RPC and ENS values can be overridden with Nuxt public env vars:

```sh
NUXT_PUBLIC_EVM_WALLET_CONNECT_PROJECT_ID=...
NUXT_PUBLIC_EVM_CHAINS_MAINNET_RPCS="https://eth.llamarpc.com https://rpc.example"
NUXT_PUBLIC_EVM_ENS_INDEXERS="https://ens-indexer.example"
```

## Main Components

Wallet and account:

- `EvmConnectDialog`: best default wallet connect trigger/dialog.
- `EvmConnect`: inline wallet selector for custom onboarding surfaces.
- `EvmConnectionStatus`: compact connection state.
- `EvmAccount`: address display with optional ENS resolution.
- `EvmAvatar`: ENS avatar or fallback.
- `EvmProfile`: connected account menu with profile/network/disconnect UI.
- `EvmSwitchNetwork`: network switch UI.
- `EvmInAppWalletSetup`: in-app wallet setup UI when enabled.

Transaction and contract UX:

- `EvmTransactionFlow` / `EvmTransactionFlowDialog`: one transaction request with status and receipt handling.
- `EvmMultiTransactionFlow` / `EvmMultiTransactionFlowDialog`: sequential transaction flows.
- `EvmAddressInput`: address or ENS input.
- `EvmEthInput`: ETH amount input with wei model.
- `EvmArtifact`: NFT/media artifact rendering from normalized metadata.

Auth:

- `EvmSiwe` / `EvmSiweDialog`: nonce-based SIWE UI.
- `EvmConnectAuth` / `EvmConnectAuthDialog`: connect wallet and SIWE in one flow.

Base UI:

- Use the layer's `Button`, `Card`, `Dialog`, `Form`, `FormItem`, `FormLabel`, `FormTextarea`, `FormSelect`, `Dropdown`, `Sidebar`, `Tag`, `Tooltip`, `Actions`, `useToast`, `useConfirm`, and layout primitives before creating app-specific replacements.

## Common Patterns

Connect button:

```vue
<template>
  <EvmConnectDialog>
    <template #default>
      Connect wallet
    </template>

    <template #connected="{ address }">
      <EvmProfile>
        <EvmAccount :address="address" resolve-ens />
      </EvmProfile>
    </template>
  </EvmConnectDialog>
</template>
```

Transaction dialog with a custom write:

```vue
<template>
  <EvmTransactionFlowDialog
    chain="sepolia"
    :request="mint"
    :text="{ title: { confirm: 'Mint token' }, action: { confirm: 'Mint' } }"
  >
    <template #start="{ start }">
      <Button class="primary" @click="start">Mint</Button>
    </template>
  </EvmTransactionFlowDialog>
</template>

<script setup lang="ts">
import { writeContract } from '@wagmi/core'

const wagmiConfig = useConfig()

async function mint() {
  return writeContract(wagmiConfig, {
    address: '0x0000000000000000000000000000000000000000',
    abi: [],
    functionName: 'mint',
  })
}
</script>
```

This pattern imports raw wagmi from app code, so add `@wagmi/core` directly when using it.

SIWE UI:

```vue
<template>
  <EvmConnectAuthDialog
    :get-nonce="() => $fetch('/api/siwe/nonce')"
    :verify="verify"
    statement="Sign in to continue."
  />
</template>

<script setup lang="ts">
const verify = (message: string, signature: string) =>
  $fetch('/api/siwe/verify', {
    method: 'POST',
    body: { message, signature },
  })
</script>
```

## Styling Basics

The layer loads `@1001-digital/styles`, which defines reset, base styles, component styles, utility classes, and CSS variables.

Core rule: customize with variables first, then use focused selector overrides only when no useful token exists.

Good app-level theme starting point:

```css
:root {
  color-scheme: light;
  --background: #fffdf8;
  --color: #151515;
  --primary: #1f5eff;
  --muted: #6f6a60;

  --button-primary-background: #151515;
  --button-primary-background-highlight: #31302d;
  --button-primary-border-color: #151515;
  --button-primary-border-color-highlight: #31302d;
  --button-primary-color: #fffdf8;
  --button-primary-color-highlight: #fffdf8;
}
```

Add global app CSS through Nuxt so it loads after the layer:

```ts
export default defineNuxtConfig({
  css: ['~/assets/css/app.css'],
})
```

When customizing heavily, ask for or infer a clear style direction first: product/tool, gallery, editorial, playful, protocol-native, dense admin UI, etc. Then adjust variables as a coherent system: background, text, muted text, primary color, buttons, inputs, cards, dialogs, and focus states.

After any theme override, visually check disconnected wallet, connected profile, dialogs, inputs, primary buttons, hover/focus states, and mobile layout.

## Pair With

- [`data.md`](data.md) when reads need fallback, cache, or live updates.
- [`metadata.md`](metadata.md) when rendering NFT or decentralized media.
- [`indexing.md`](indexing.md) when the app needs event-derived local data.
- [`wallet-auth.md`](wallet-auth.md) when SIWE backend sessions or in-app wallets matter.
- [`contract-intelligence.md`](contract-intelligence.md) when the UI must understand proxy implementations or verified docs.

## Agent Checklist

- Start Nuxt EVM apps with `pnpm add @1001-digital/layers.evm`.
- Configure chains in `app.config.ts`; put endpoint-like values in runtime config/env.
- Use layer components before making custom wallet, account, transaction, dialog, or form primitives.
- Add wagmi/viem only when app code imports them directly.
- Customize via CSS variables first.
- Ask when product direction, style direction, wallet UX, chains, contracts, or auth flow are underspecified.
