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

## Styling And Customization

The layer loads `@1001-digital/styles`, which defines reset, base styles, component styles, utility classes, CSS cascade layers, and CSS variables.

The core rule is: customize with variables first, then use focused selector overrides only when no useful token exists. Almost anything can be restyled, but agents need to understand which layer they are changing.

### Style Brief

Before heavy customization, make the visual target explicit. Agents should not infer a brand system from package names or from the default layer theme.

Ask the prompter when these details would change the result:

- Style direction: product/tool, gallery, editorial, playful, protocol-native, dense admin UI, or another named direction.
- Density: compact workspace, spacious showcase, mobile-first, desktop-first, or responsive parity.
- Scheme: light, dark, both, or locked one-scheme app.
- Palette: brand colors, neutral base, semantic colors, and whether gradients, textures, or images are allowed.
- Typography: system fonts, brand font, monospace/protocol feel, editorial headings, or tight utility UI.
- Shape language: radius scale, borders, shadows, dividers, cards, panels, and native vs custom controls.
- Wallet UX tone: minimal header connect, prominent onboarding, Safe/operator workflow, in-app wallet education, or SIWE session app.
- States to polish: disconnected, connecting, connected, wrong chain, signing, pending transaction, success, failure, empty data, and loading data.

If the brief is incomplete but the user wants progress, state the assumptions and keep them centralized in tokens so they can be changed quickly.

### Customization Ladder

Use the narrowest customization layer that matches the intent:

1. Global semantic tokens: product-wide theme through `:root` variables such as `--background`, `--color`, `--primary`, `--muted`, `--border-radius`, spacing, font, and border tokens.
2. Component tokens: all buttons, forms, cards, dialogs, popovers, dropdowns, toasts, and wallet/profile triggers through their dedicated variables.
3. Component props/classes: props like `class-name` on EVM components and regular `class` on base components for one product surface.
4. App CSS selectors: app-owned selectors for layout, composition, and one-off styling. Use global CSS when the target is a dialog, toast, wallet overlay, or any component mounted outside the local subtree.
5. App-local component override: create an app component with the same name only when markup or behavior must change. Nuxt resolves app components over layer components.
6. Package-level changes: change 1001 packages only when the customization is reusable upstream behavior.

Prefer tokens for broad visual language, classes for per-instance product styling, and component overrides for behavior or markup changes. Avoid deep selectors until you inspect the rendered component and confirm no token, prop, slot, or class hook covers the need.

### Token Map

Layer components share token families. Overriding only one family can make the UI inconsistent.

| Need | Start with these tokens |
| --- | --- |
| Page/app palette | `color-scheme`, `--background`, `--background-semi`, `--color`, `--color-semi`, `--muted`, `--primary`, `--error`, `--success` |
| UI typography | `--ui-font-family`, `--ui-font-size`, `--ui-font-weight`, `--ui-text-transform`, `--ui-letter-spacing`, `--ui-line-height`, `--ui-color`, `--ui-placeholder-color` |
| Layout rhythm | `--spacer-*`, `--size-*`, `--content-width*`, `--form-width`, `--dialog-width` |
| Borders/shadows/radius | `--border-width`, `--border-color`, `--border-color-highlight`, `--border-radius*`, `--border`, `--border-shadow`, `--border-shadow-highlight`, `--shadow` |
| Buttons and wallet triggers | `--button-background`, `--button-background-highlight`, `--button-color`, `--button-color-highlight`, `--button-icon-color`, `--button-icon-color-highlight`, primary/tertiary button tokens |
| Inputs/selects/textareas | `--input-background`, `--input-background-highlight`, `--input-text-transform`, shared `--ui-*`, shared border tokens |
| Cards/artifacts/list items | `--card-background`, `--card-background-highlight`, `--card-border`, `--card-border-radius`, `--card-border-color-highlight` |
| Dialogs and wallet overlays | `--dialog-width`, `--dialog-border-radius`, `--dialog-header-background`, `--dialog-close-color`, `--backdrop-background` |
| Popovers/dropdowns | `--popover-*`, `--dropdown-*` |
| Toasts | `--toast-width`, `--toast-inset`, `--toast-border-radius`, `--toast-*-color`, `--toast-*-background`, `--toast-*-border-color`, `--toast-*-header-background` |

Native form elements are styled by the base layer too. Buttons, inputs, textareas, and selects all consume `--ui-*`, `--button-*`, form height, border shadow, and sometimes `--input-*`. Themes should update button and input tokens together.

### Global App CSS

Put global token overrides in a non-scoped stylesheet loaded by Nuxt:

```ts
export default defineNuxtConfig({
  css: ['~/assets/css/app.css'],
})
```

Then define the theme in that file:

```css
:root {
  color-scheme: light;
  --font-family: Inter, system-ui, sans-serif;
  --background: #fffdf8;
  --color: #151515;
  --primary: #1f5eff;
  --muted: #6f6a60;
  --border-radius: 8px;

  --button-border-radius: var(--border-radius);
  --card-border-radius: var(--border-radius);
  --dialog-border-radius: var(--border-radius);
}
```

Avoid scoped token definitions for global UI variables. Scoped styles add attributes and will not reliably affect layer components mounted outside the component subtree, such as global toasts and dialogs.

Use one app-level CSS file as the theme source of truth. Keep token definitions near the top, then product layout classes, then one-off component classes. When a value is reused more than once, make it a token before copying it into selectors.

### Cascade Layers

`@1001-digital/styles` uses this order:

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

Styles outside `@layer` win over layer styles. App-level CSS can override component styles directly, but variables are safer when a token exists.

Do not hard-code colors, spacing, border radii, or button styles when a token exists. Override CSS variables first, then use focused selector overrides only for component structure or one-off states that do not have a token.

### Theme And Wallet Contrast

When an app changes the layer palette, pin the browser color scheme and override the component tokens that wallet dialogs, profile buttons, inputs, and native form controls actually consume. A common failure is setting `--color` to dark text while the browser or component tokens still resolve dark backgrounds, producing unreadable wallet buttons.

For a light custom app theme:

```css
:root {
  color-scheme: light;
  --background: #f6f3ec;
  --color: #151515;
  --primary: #237a57;
  --ui-color: var(--color);
  --ui-placeholder-color: #8a8377;

  --button-background: #ffffff;
  --button-background-highlight: #f0ebe1;
  --button-color: var(--color);
  --button-color-highlight: var(--color);
  --button-icon-color: #6b665d;
  --button-icon-color-highlight: var(--color);

  --button-primary-background: #151515;
  --button-primary-background-highlight: #31302d;
  --button-primary-border-color: #151515;
  --button-primary-border-color-highlight: #31302d;
  --button-primary-color: #fffdf8;
  --button-primary-color-highlight: #fffdf8;

  --input-background: #ffffff;
  --input-background-highlight: #ffffff;
}
```

If the header uses a custom connect button class, apply equivalent styling to the connected profile trigger too. `EvmConnectDialog`'s `class-name` styles the disconnected trigger; the `#connected` slot owns the connected button.

```vue
<EvmConnectDialog class-name="wallet-button">
  <template #default>
    Connect
  </template>

  <template #connected="{ address }">
    <EvmProfile class-name="wallet-button">
      <EvmAccount :address="address" resolve-ens />
    </EvmProfile>
  </template>
</EvmConnectDialog>
```

```css
.wallet-button {
  border: 1px solid #151515;
  background: #151515;
  color: #fffdf8 !important;
}

.wallet-button :is(span, strong, small, .icon) {
  color: inherit;
}
```

Before shipping, test the disconnected connect trigger, wallet selection dialog, in-app wallet setup path, connected profile trigger, profile dialog, network switcher, and disconnect action. Browser-computed `color` and `background-color` are the fastest way to catch contrast mismatches.

### Core Tokens

Use these before writing hard-coded styles:

```css
:root {
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

Common UI and component variables:

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
  --button-primary-color: #fff;

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

### Button And Dialog Gotchas

The base CSS styles native buttons. A custom `button` class can inherit layout, block size, shadow, and border radius from the framework.

When building a fully custom native button, either use the `Button` component or reset the inherited button behavior:

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

### Styling Checklist

- Confirm style direction, density, scheme, palette, typography, shape language, wallet UX tone, and important states.
- Put global token overrides in Nuxt CSS, not scoped component CSS.
- Override tokens before selectors.
- Set `color-scheme` when overriding the app palette.
- Override button and input tokens together.
- Style both the disconnected `EvmConnectDialog` trigger and the connected `EvmProfile` trigger.
- Check wallet connect, in-app wallet setup, connected profile, profile/network dialogs, buttons, inputs, hover, focus, and mobile layout after theme changes.

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
- Ask when product direction, style direction, wallet UX, chains, contracts, or auth flow are underspecified.
- Customize via global CSS variables first; use selectors only after checking tokens, props, slots, and class hooks.
- Set `color-scheme` and override button/input/wallet tokens together when changing the palette.
- Test both disconnected connect and connected profile states after styling wallet UI.
