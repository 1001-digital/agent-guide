# Layers

Use this guide when an agent needs to build a Nuxt/Vue dapp with 1001 UI, EVM wallet flows, transaction UX, and the shared design system.

## What To Use This For

- New Nuxt EVM dapps.
- Wallet connection and account/profile UI.
- Sign-In with Ethereum UI flows.
- Transaction request, chain check, receipt waiting, success/error UX.
- Base product UI: shell, sidebar, forms, buttons, dialogs, dropdowns, toasts, tags, cards, tooltips.
- CSS variables, design tokens, component-level styling, and app-specific overrides.
- Direct Vue component usage outside Nuxt when a lower-level package wants the component library but not the full layer.

## When Not To Use It

- Do not use the EVM layer for a non-EVM app that only needs base UI. Use `@1001-digital/layers.base`.
- Do not import `@1001-digital/components.evm` directly in a Nuxt app unless you are deliberately bypassing the layer. The layer handles auto-registration, client-only wrappers, wagmi setup, and aliases.
- Do not use `layers.evm` as a backend auth system. It provides client UI and message helpers; the app still owns nonce storage, session persistence, and server verification.
- Do not hard-code colors, spacing, border radii, or button styles when a token exists. Override CSS variables first.

## Packages/Repos Involved

- Source repo: <https://github.com/1001-digital/layers>
- `@1001-digital/layers.evm`: Nuxt layer for EVM apps. Extends the base layer.
- `@1001-digital/layers.base`: Nuxt layer for base app UI, global plugins, icons, and styles.
- `@1001-digital/components.evm`: Vue EVM components, composables, and utilities.
- `@1001-digital/components`: Vue base component library.
- `@1001-digital/styles`: CSS reset, base styles, design tokens, component variables, and utilities.

Dependency direction:

```txt
@1001-digital/layers.evm
  -> @1001-digital/layers.base
  -> @1001-digital/components
  -> @1001-digital/components.evm
  -> @1001-digital/styles
```

## Install Commands

For a Nuxt EVM dapp:

```sh
pnpm add @1001-digital/layers.evm
```

If the package manager or workspace setup does not expose transitive layer packages cleanly, add the public packages directly:

```sh
pnpm add @1001-digital/layers.evm @1001-digital/components @1001-digital/components.evm @1001-digital/styles
```

For a Nuxt app that only needs base UI:

```sh
pnpm add @1001-digital/layers.base
```

For a non-Nuxt Vue package or app:

```sh
pnpm add @1001-digital/components @1001-digital/styles
pnpm add @1001-digital/components.evm @wagmi/vue @wagmi/core viem
```

## Minimal Setup

### Nuxt EVM App

Use `@1001-digital/layers.evm` in `nuxt.config.ts`. Do not also extend the base layer; EVM already does it.

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

Use `app.config.ts` for app behavior that is safe to bundle:

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

The `defaultChain` value must match a key in `chains`. The layer sorts that chain first because wagmi treats the first configured chain as the default.

### Environment Variables

Declare runtime config keys in `nuxt.config.ts`, then override them with Nuxt public env vars:

```sh
NUXT_PUBLIC_EVM_WALLET_CONNECT_PROJECT_ID=...
NUXT_PUBLIC_EVM_CHAINS_MAINNET_RPCS="https://eth.llamarpc.com https://rpc.example"
NUXT_PUBLIC_EVM_CHAINS_SEPOLIA_RPCS="https://sepolia.example"
NUXT_PUBLIC_EVM_ENS_INDEXERS="https://ens-indexer.example"
```

RPC and ENS indexer values are whitespace-separated lists. For each configured chain, the layer builds a wagmi fallback transport from custom RPCs, injected connector transport, and default HTTP transport.

### SSR

SSR is supported by the layer. Wallet and EVM components that depend on browser APIs are registered as client-only by the Nuxt layer.

If a specific deployment needs a fully client-rendered dapp, disable SSR with:

```sh
NUXT_SSR=false
```

Do not manually wrap every EVM component in `<ClientOnly>` unless the consuming page has its own SSR issue. The layer already marks wallet-dependent components client-only.

## Core APIs/Components/Contracts/Config

### What The EVM Layer Adds

| Area | What agents can rely on |
| --- | --- |
| Components | Auto-registers EVM components from `@1001-digital/components.evm`. |
| Wallets | Configures injected wallets, Safe, Base Account, MetaMask, WalletConnect, and optional in-app wallet. |
| Data | Installs wagmi Vue integration and TanStack Query through the layer plugin. |
| Config | Merges `app.config.ts` behavior with `runtimeConfig.public.evm` endpoint values. |
| Utilities | Exposes EVM composables and utilities for Nuxt auto-imports. |
| Safe | Provides Safe connector support through wagmi. |
| Styles | Loads shared reset, base CSS, component variables, and utility classes. |

### Wallet Connection Components

Use these when the app needs wallet connection, account status, QR connector flows, or in-app wallet setup.

| Component | Purpose | Notes |
| --- | --- | --- |
| `EvmConnectDialog` | Dialog-based wallet connection. | Best default for app headers and CTA buttons. |
| `EvmConnect` | Inline wallet selection UI. | Use inside custom dialogs or onboarding panels. |
| `EvmConnectionStatus` | Connection state display. | Reads wagmi connection state. |
| `EvmConnectorQR` | Generic QR connector UI. | Usually used by connector-specific components. |
| `EvmMetaMaskQR` | MetaMask QR flow. | Emits `back`. |
| `EvmWalletConnectQR` | WalletConnect QR flow. | Requires WalletConnect URI. |
| `EvmWalletConnectWallets` | WalletConnect wallet list. | Emits `back`. |
| `EvmInAppWalletSetup` | In-app wallet setup UI. | Requires in-app wallet to be enabled/configured. |
| `EvmSeedPhraseInput` | Seed phrase input. | `v-model`, emits validity and submit events. |

Default connection pattern:

```vue
<template>
  <EvmConnectDialog
    @connected="onConnected"
    @disconnected="onDisconnected"
  >
    <template #default>
      Connect wallet
    </template>

    <template #connected="{ address }">
      <EvmAccount :address="address" resolve-ens />
    </template>
  </EvmConnectDialog>
</template>

<script setup lang="ts">
function onConnected(payload: { address: string | undefined }) {
  console.log('connected', payload.address)
}

function onDisconnected() {
  console.log('disconnected')
}
</script>
```

### Account, Profile, And Input Components

| Component | Purpose | Use when |
| --- | --- | --- |
| `EvmAccount` | Short address display with optional ENS. | Rendering a connected account, recipient, owner, or minter. |
| `EvmAvatar` | ENS avatar or generated fallback. | Showing profile UI. |
| `EvmProfile` | Wallet profile, network switcher, disconnect. | Header/sidebar account menus. |
| `EvmSidebarProfile` | Sidebar profile variant. | Apps with persistent sidebars. |
| `EvmSwitchNetwork` | Network switching UI. | The action depends on a target chain. |
| `EvmAddressInput` | Address or ENS text input. | Recipient/owner/admin forms. |
| `EvmEthInput` | ETH amount input with wei model. | Payable actions or pricing forms. |

Address and ETH input pattern:

```vue
<template>
  <Form>
    <FormLabel label="Recipient">
      <EvmAddressInput v-model="recipient" placeholder="vitalik.eth or 0x..." />
    </FormLabel>

    <FormLabel label="Amount">
      <EvmEthInput
        v-model="ethAmount"
        v-model:wei="weiAmount"
        placeholder="0.05"
      />
    </FormLabel>

    <EvmAccount
      v-if="recipientAddress"
      :address="recipientAddress"
      resolve-ens
    />
  </Form>
</template>

<script setup lang="ts">
import { isAddress, type Address } from 'viem'

const recipient = ref('')
const ethAmount = ref('')
const weiAmount = ref<bigint | null>(null)

const recipientAddress = computed<Address | undefined>(() =>
  isAddress(recipient.value) ? recipient.value : undefined,
)
</script>
```

### Transaction Flow Components

Use transaction flow components whenever a user action calls `writeContract`, sends ETH, or triggers a wallet signature that must be tracked to receipt.

| Component | Purpose | Notes |
| --- | --- | --- |
| `EvmTransactionFlow` | Inline transaction state machine. | Useful inside custom panels. |
| `EvmTransactionFlowDialog` | Dialog wrapper around the flow. | Best default for user-triggered writes. |
| `EvmMultiTransactionFlow` | Multi-transaction flow. | Use when one user action sends a sequence. |
| `EvmMultiTransactionFlowDialog` | Dialog wrapper for multi-transaction flow. | Best for batch operations. |

Basic write pattern:

```vue
<template>
  <EvmTransactionFlowDialog
    chain="sepolia"
    :request="mint"
    :text="{
      title: { confirm: 'Mint token' },
      action: { confirm: 'Mint', error: 'Try again' },
    }"
    @complete="onComplete"
    @cancel="onCancel"
  >
    <template #start="{ start }">
      <Button class="primary" @click="start">
        Mint
      </Button>
    </template>
  </EvmTransactionFlowDialog>
</template>

<script setup lang="ts">
import { writeContract } from '@wagmi/core'
import type { Hash } from 'viem'

const wagmiConfig = useConfig()

async function mint(): Promise<Hash> {
  return writeContract(wagmiConfig, {
    address: '0x0000000000000000000000000000000000000000',
    abi: [],
    functionName: 'mint',
    args: [],
  })
}

function onComplete(receipt: unknown) {
  console.log('complete', receipt)
}

function onCancel() {
  console.log('cancelled')
}
</script>
```

Transaction flow responsibilities:

- Optional confirmation step.
- Target-chain validation and switching.
- Wallet request state.
- Receipt waiting state.
- Success/error state.
- Optional toast handoff while waiting.
- `complete`, `cancel`, and step update events.

### SIWE And Auth UI

Use SIWE components when the frontend must prove wallet ownership to a backend that exposes nonce and verification endpoints compatible with the layer flow.

| Component | Purpose | Backend responsibility |
| --- | --- | --- |
| `EvmSiwe` | Inline SIWE flow. | Nonce endpoint, verification endpoint, session persistence. |
| `EvmSiweDialog` | Dialog SIWE flow. | Same as above. |
| `EvmConnectAuth` | Wallet connect plus SIWE. | Same as above. |
| `EvmConnectAuthDialog` | Dialog wrapper for connect plus SIWE. | Same as above. |

Client pattern:

```vue
<template>
  <EvmConnectAuthDialog
    :get-nonce="getNonce"
    :verify="verify"
    statement="Sign in to continue."
    @authenticated="onAuthenticated"
    @error="onError"
  />
</template>

<script setup lang="ts">
const getNonce = () => $fetch<string>('/api/siwe/nonce')

const verify = (message: string, signature: string) =>
  $fetch<boolean>('/api/siwe/verify', {
    method: 'POST',
    body: { message, signature },
  })

function onAuthenticated(payload: unknown) {
  console.log(payload)
}

function onError(error: unknown) {
  console.error(error)
}
</script>
```

These layer components create the SIWE message in the browser after `getNonce()` returns a nonce string. A backend that returns a complete SIWE message, such as the default `@1001-digital/adonis-siwe` route flow, needs a custom frontend signing flow instead of these nonce-based components.

### Artifact Component

Use `EvmArtifact` to render NFT/media artifacts.

```vue
<template>
  <EvmArtifact
    :metadata="metadata"
    aspect-ratio="1 / 1"
    use-background-color
    @error="onArtifactError"
  />
</template>
```

Pair with:

- `@1001-digital/dweb-fetch` for fetching `ipfs://`, `ar://`, and `eip155:` metadata.
- `@1001-digital/resolve-metadata` for normalizing token and contract metadata before rendering.

### Base Components

Use base components for normal app UI.

| Category | Components |
| --- | --- |
| Layout | `AppShell`, `Sidebar`, `BottomNav`, `Card`, `CardLink`, `Prose`, `Actions` |
| Buttons and feedback | `Button`, `Alert`, `Toasts`, `Loading`, `Progress`, `Tag`, `Tags`, `CopyText` |
| Overlays | `Dialog`, `ConfirmDialog`, `Popover`, `Tooltip`, `Dropdown` and dropdown item/group components |
| Forms | `Form`, `FormItem`, `FormGroup`, `FormLabel`, `FormInputGroup`, `FormTextarea`, `FormCheckbox`, `FormRadioGroup`, `FormSwitch`, `FormSlider`, `FormSelect`, `FormDateField`, `FormDatePicker`, `PinInput`, `TagsInput`, `Combobox`, `Autocomplete`, `ColorPicker` |
| Media/utilities | `Avatar`, `Icon`, `Opepicon`, `Embed`, `Calendar`, `Globals` |

Base UI pattern:

```vue
<template>
  <AppShell>
    <template #sidebar>
      <Sidebar>
        <NuxtLink to="/">Dashboard</NuxtLink>
      </Sidebar>
    </template>

    <Card>
      <Actions>
        <Button class="primary">
          <Icon name="check" />
          <span>Confirm</span>
        </Button>
        <Button class="tertiary">Cancel</Button>
      </Actions>
    </Card>
  </AppShell>
</template>
```

### Composables And Utilities

Wagmi composables are available from the layer setup:

```ts
const { address, chainId, status, connector } = useAccount()
const wagmiConfig = useConfig()
```

Use `@wagmi/core` for reads and writes:

```ts
import { readContract, writeContract } from '@wagmi/core'
import { parseEther } from 'viem'

const config = useConfig()

const balance = await readContract(config, {
  address: '0x0000000000000000000000000000000000000000',
  abi: [],
  functionName: 'balanceOf',
  args: [address.value],
})

const hash = await writeContract(config, {
  address: '0x0000000000000000000000000000000000000000',
  abi: [],
  functionName: 'mint',
  value: parseEther('0.01'),
})
```

Layer composables and helpers agents should look for:

| API | Use |
| --- | --- |
| `useChainConfig(chainKey)` | Read configured chain ID/block explorer by app chain key. |
| `useMainChainId()` | Resolve the configured default chain ID. |
| `useBlockExplorer(chainKey)` | Build explorer links. |
| `useEnsureChainIdCheck(chainKey)` | Check/switch to the expected chain before writes. |
| `useEns(addressOrName)` | Resolve ENS profile data. |
| `useGasPrice()` | Display current gas price. |
| `usePriceFeed()` | Fetch cached Chainlink ETH/USD data on mainnet and convert wei to USD. |
| `useSiwe()` | Client-side SIWE session state/actions. |
| `useToast()` | Global toast state and actions. |
| `useConfirm()` | Global confirm dialog. |
| `shortAddress(address, chars?)` | Shorten addresses for UI. |
| `formatETH(value, decimals?)` | Format ETH-denominated string/number values for display. |
| `resolveChain(id)` | Resolve viem chain objects by numeric chain ID. |
| `useDwebClient()` | Create a configured dweb client for fetch/URL resolution. |
| `useResolvedUrl(uri)` | Resolve IPFS/Arweave/data/blob/HTTP URLs for rendering. |

Toast pattern:

```ts
const { add, update, dismiss } = useToast()

const id = add({
  title: 'Indexing',
  description: 'Fetching events...',
  variant: 'info',
  loading: true,
})

update(id, {
  title: 'Done',
  description: 'Events are synced.',
  variant: 'success',
  loading: false,
  duration: 3000,
})

dismiss(id)
```

Confirm pattern:

```ts
const { confirm } = useConfirm()

const ok = await confirm({
  title: 'Delete item?',
  description: 'This action cannot be undone.',
})

if (ok) {
  // continue
}
```

The base layer mounts global toasts and confirm dialogs automatically. Do not manually add `<Toasts />` or `<ConfirmDialog />` to `app.vue` in a Nuxt app using the layer.

## Common Pairings With Other 1001 Blocks

- Pair `EvmArtifact` with `dweb-fetch` and `resolve-metadata` for NFT display.
- Pair transaction flows with `dapp-query` invalidation after writes.
- Pair wallet connection UI with `adonis-siwe` when using AdonisJS, but use Adonis' full-message signing flow rather than the layer's nonce-based SIWE components.
- Pair `EvmAddressInput` and `EvmAccount` with `ponder-ens` when an indexer provides cached ENS profiles.
- Pair `useResolvedUrl`, `useDwebClient`, or configured gateways with `ipfs.server` when the app owns its gateway.
- Pair `proxies` and `natspec` with `EvmArtifactModel` or contract-detail pages that need implementation ABI/docs.

## Practical Implementation Patterns

### App Header

```vue
<template>
  <header class="app-header">
    <NuxtLink to="/" class="brand">
      My dapp
    </NuxtLink>

    <nav>
      <NuxtLink to="/explore">Explore</NuxtLink>
      <NuxtLink to="/mint">Mint</NuxtLink>
    </nav>

    <EvmConnectDialog>
      <template #connected="{ address }">
        <EvmProfile>
          <EvmAccount :address="address" resolve-ens />
        </EvmProfile>
      </template>
    </EvmConnectDialog>
  </header>
</template>
```

### Write Action With Chain Guard

```ts
import { writeContract } from '@wagmi/core'

const config = useConfig()
const ensureChain = useEnsureChainIdCheck('sepolia')

async function requestMint() {
  const ok = await ensureChain()
  if (!ok) throw new Error('Wrong chain')

  return writeContract(config, {
    address: '0x0000000000000000000000000000000000000000',
    abi: [],
    functionName: 'mint',
  })
}
```

Pass `requestMint` into `EvmTransactionFlowDialog` rather than hand-rolling loading states.

### Metadata Rendering

```ts
import { createDwebFetch } from '@1001-digital/dweb-fetch'
import { resolveTokenMetadata } from '@1001-digital/resolve-metadata'

const dweb = createDwebFetch({
  ipfs: { mode: 'gateway' },
})

const raw = await dweb.fetch(tokenUri).then((res) => res.json())
const metadata = resolveTokenMetadata(raw)
```

```vue
<EvmArtifact :metadata="metadata" />
```

### Global App CSS

Put global token overrides in a non-scoped stylesheet loaded by Nuxt, or in an unscoped `app.vue` style block:

```vue
<style>
:root {
  --font-family: Inter, system-ui, sans-serif;
  --background: #ffffff;
  --color: #141414;
  --primary: #1f5eff;
  --border-radius: 8px;
  --button-border-radius: 8px;
  --card-border-radius: 8px;
}
</style>
```

Avoid scoped token definitions for global UI variables. Scoped styles add attributes and will not reliably affect layer components mounted outside the component subtree, such as global toasts.

## CSS/Config/Runtime/Env Details

### Cascade Layers

`@1001-digital/styles` uses this cascade layer order:

```css
@layer variables, reset, base, components, utilities;
```

| Layer | Purpose |
| --- | --- |
| `variables` | CSS custom properties and token definitions. |
| `reset` | Modern reset/normalize behavior. |
| `base` | Element defaults, forms, and scrolling behavior. |
| `components` | Component styles. |
| `utilities` | Utility classes intended to override component defaults when needed. |

Styles outside `@layer` win over layer styles. App-level CSS can override component styles directly, but prefer variables when available.

### Core Tokens

Use these before writing hard-coded styles:

```css
:root {
  --black: #000;
  --white: #fff;
  --background: #fff;
  --background-semi: rgba(255, 255, 255, 0.8);
  --color: #111;
  --color-semi: rgba(17, 17, 17, 0.7);
  --primary: #1f5eff;
  --muted: #777;
  --error: #cc2b2b;
  --success: #238636;

  --font-family: system-ui, sans-serif;
  --font-base: 16px;
  --font-sm: 0.875rem;
  --font-lg: 1.125rem;
  --font-weight: 400;
  --font-weight-bold: 700;
  --line-height: 1.5;

  --spacer-xs: 0.125rem;
  --spacer-sm: 0.5rem;
  --spacer: 1rem;
  --spacer-md: 1.5rem;
  --spacer-lg: 2rem;

  --border-width: 1px;
  --border-radius-sm: 4px;
  --border-radius: 8px;
  --border-radius-lg: 12px;
  --border-color: #ddd;
  --border: var(--border-width) solid var(--border-color);

  --shadow: 0 1rem 3rem rgba(0, 0, 0, 0.14);
  --speed-fast: 100ms;
  --speed: 150ms;
  --speed-slow: 250ms;
}
```

### UI And Component Tokens

Common UI variables:

```css
:root {
  --ui-font-family: var(--font-family);
  --ui-font-size: var(--font-sm);
  --ui-font-weight: 500;
  --ui-text-transform: none;
  --ui-letter-spacing: 0;
  --ui-line-height: 1.2;

  --button-border-radius: var(--border-radius);
  --button-background: var(--background);
  --button-background-highlight: #f4f4f4;
  --button-color: var(--color);
  --button-color-highlight: var(--color);
  --button-primary-background: var(--primary);
  --button-primary-color: var(--white);

  --card-border-radius: var(--border-radius);
  --card-background: var(--background);
  --card-background-highlight: #f8f8f8;

  --dialog-width: 32rem;
  --backdrop-background: rgba(0, 0, 0, 0.5);
}
```

Use `.light` or `.dark` on a subtree to force a scheme when the underlying color tokens support it.

### Utility Classes

Use the built-in utilities when they fit:

| Class | Purpose |
| --- | --- |
| `.ui` | Applies UI typography and color. |
| `.muted` | Muted text color. |
| `.font-sm` | Small font token. |
| `.visible-sm` | Hidden by default, visible from the small breakpoint. |
| `.visible-md` | Hidden by default, visible from the medium breakpoint. |

### Button Reset Gotcha

The base CSS styles native buttons. A custom `button` class can inherit layout, block size, shadow, and border radius from the framework.

When building a fully custom button, either use the `Button` component or reset the inherited button behavior:

```css
.my-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  block-size: auto;
  min-inline-size: auto;
  box-shadow: none;
  border: 1px solid currentColor;
  border-radius: 0;
  background: transparent;
}
```

### Dialog Gotchas

If an app reset includes `* { margin: 0; }`, restore native dialog centering:

```css
dialog {
  margin: auto;
}
```

If Safari stretches a native dialog because grid rows expand unexpectedly, override the dialog layout with flexbox:

```css
.dialog {
  display: flex;
  flex-direction: column;
  block-size: fit-content;
  max-block-size: 80dvh;
}
```

For narrow viewports:

```css
@media (max-width: 480px) {
  :root {
    --dialog-width: 18rem;
  }
}
```

### Vite/Pnpm Resolution

The EVM layer aliases `@wagmi/vue` and `eventemitter3` to a single package instance. This avoids duplicated dependency instances in pnpm installs from breaking Vue provide/inject behavior or connector interop.

If wallet state appears disconnected across components, inspect duplicate versions first:

```sh
pnpm why @wagmi/vue
pnpm why eventemitter3
```

## Gotchas And Failure Modes

- Missing runtime config keys: Nuxt env overrides only work for keys declared in `runtimeConfig`. Declare every chain key that should receive env-provided RPCs.
- Chain key mismatch: `defaultChain` must match a key in `app.config.ts` `evm.chains`.
- Empty RPCs: a chain without RPCs can fall back to defaults, but production dapps should configure reliable endpoints.
- WalletConnect without project ID: WalletConnect connectors need `walletConnectProjectId`.
- Server-only imports in client components: keep server calls behind API routes or Nitro endpoints.
- In-app wallet UX without consent: only enable in-app wallet flows when the product explicitly explains local key custody.
- Safe App metadata: do not assume the installed `@1001-digital/layers.evm` package serves `/manifest.json`. The published package may omit the server route even when the source repo has one; add an app-local route or confirm the installed version ships it before relying on Safe App browser metadata.
- Scoped CSS variables: scoped styles may not affect global overlays, toasts, or dialogs.
- Manual `Toasts` rendering: unnecessary in Nuxt layer apps; the global plugin mounts them.
- Direct component imports outside Nuxt: you must configure Vue, wagmi, viem, styles, and client-only behavior yourself.

## Agent Checklist

- Choose `@1001-digital/layers.evm` for any Nuxt EVM dapp.
- Choose `@1001-digital/layers.base` only when the app does not need EVM.
- Configure chains in `app.config.ts` and RPC/indexer endpoints in `runtimeConfig.public.evm`.
- Add `NUXT_PUBLIC_EVM_*` env vars only for declared runtime config keys.
- Use `EvmConnectDialog` for connection, `EvmAccount` for address display, and `EvmTransactionFlowDialog` for writes.
- Pair SIWE UI with nonce/verify backend endpoints; use a custom signing flow for backends that return a complete SIWE message.
- Use base components before creating app-specific primitives.
- Override CSS through tokens in global CSS first; write component CSS only when tokens are insufficient.
- Check button/dialog CSS inheritance when custom UI looks stretched or miscentered.
- Pair with metadata, data, indexing, wallet/auth, or contract-intelligence guides when the app crosses into those tasks.
