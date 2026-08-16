---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-web3-connect/","title":"hydration-ui web3-connect","tags":["wallet","web3","authentication","hydration","multisig","address-book"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui web3-connect","repo":"hydration-ui","paths":["packages/web3-connect/src","packages/web3-connect/src/components/address-book","packages/web3-connect/src/config/wallet.ts","packages/web3-connect/src/signers","packages/web3-connect/src/utils/permitGas.ts"],"key_exports":["Web3ConnectModal","Web3ConnectButton","Web3ConnectAccount","Web3ConnectAccountIdentity","useAccount","useWallet","useWeb3Connect","useWeb3Enable","useWeb3ConnectModal","useAccountMultisigs","useEvmAddress","useMultisigStore","useAddressStore","useAddresses","AddressBookModal","AddressBookEntry","getMultixSdk","multisigsByAccountIdsQuery"],"symbols":["WalletMode","WALLET_ACCOUNT_FILTER_OPTIONS","WalletAccountFilterOption","getWalletModeByAddress","getWalletModeName","getWalletModesByProviderType","addressToPublicKey","getUniqueAccountKey","getPermitGasFromWeight","getPermitGasLimit","getPermitGasPrice","EthereumSigner","SolanaSigner","dedupeAddresses","normalizeStoredAddress","deriveMultisigAddress","normalizeMultisigEntry"],"key_deps":["@mysten/wallet-standard","@reown/appkit","@solana/web3.js","viem","@tanstack/react-query","@galacticcouncil/indexer"],"tags":["wallet","web3","authentication","hydration","multisig","address-book"],"last_updated":"2026-08-15"}}
---


# hydration-ui web3-connect

**TL;DR:** Wallet connection abstraction (`packages/web3-connect/`) covering Substrate extensions, EVM (Reown AppKit / MetaMask / Nova), Solana and Sui — plus, since Aug 2026, a persisted **address book / contacts** layer and NEAR + Zcash address modes (validation-only, no signer). Exports `Web3ConnectModal`, `Web3ConnectButton`, `useAccount`, `useWeb3Connect`, `useWeb3Enable`, `useAccountMultisigs`, `useAddressStore`.

## Purpose

Abstracts wallet/signer selection across multiple chains and signature methods. The main app ([[wiki/hydration-ui-main-app\|hydration-ui-main-app]]) uses this to:
1. Prompt user to select wallet
2. Retrieve connected account + balance
3. Get a signing function for tx submission ([[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]])

## Wallet adapters

| Wallet | Chain | Method |
|--------|-------|--------|
| Polkadot.js / Talisman / SubWallet / Nova | Substrate (Hydration) | `polkadot-api/pjs-signer`, `BaseSubstrateWallet` |
| Reown AppKit (WalletConnect v2) | Multi-chain | `@reown/appkit` |
| MetaMask / Nova (EIP-1193) | EVM (Frontier) | `window.ethereum`, `BaseEIP1193Wallet` |
| Phantom & co. | Solana | wallet-standard |
| Sui wallets | Sui | `@mysten/wallet-standard` |

`getWallet(type?)` (`wallets/index.ts`) now accepts `undefined` and returns `undefined` rather than throwing on an unknown provider.

## `WalletMode` — moved and extended

`WalletMode` moved out of `hooks/useWeb3Connect.ts` into **`config/wallet.ts`** (still re-exported from the hook for back-compat) and gained two chains:

```typescript
// packages/web3-connect/src/config/wallet.ts
export enum WalletMode {
  Default = "default", EVM = "evm", Substrate = "substrate",
  SubstrateEVM = "substrate-evm", SubstrateH160 = "substrate-h160",
  Solana = "solana", Sui = "sui",
  Near = "near", Zcash = "zcash",   // NEW — validation/receive only
  Unknown = "unknown",
}

export const WALLET_ACCOUNT_FILTER_OPTIONS = [
  WalletMode.Substrate, WalletMode.SubstrateH160, WalletMode.EVM,
  WalletMode.Solana, WalletMode.Sui, WalletMode.Near, WalletMode.Zcash,
] as const satisfies Array<WalletMode>
```

`PROVIDERS_BY_WALLET_MODE[Near]` and `[Zcash]` map to `[]` — there is no signer for these; they exist so an address can be stored, filtered and displayed (deposit/onramp destinations). `getWalletModeByAddress()` now also detects Sui, NEAR and Zcash; `getWalletModeName()` maps a mode to a display label ("Polkadot", "EVM", "Solana", "Sui", "Substrate H160", "NEAR", "Zcash"); `getWalletModesByProviderType()` inverts the provider map. `getWalletModeIcon()` gained NEAR/Zcash entries and now folds `SubstrateH160` into the Substrate icon.

`addressToPublicKey(address)` (new, `utils/wallet.ts`) is the canonical identity key: lowercased hex for EVM, decoded public key for SS58, the raw address for Solana / Sui / NEAR / Zcash, `""` otherwise.

## Address book / contacts (`components/address-book/`)

Largest change in this package. A Zustand + `persist` store of named addresses, shared across wallets.

| File | Role |
|---|---|
| `AddressBook.store.ts` | `useAddressStore` / `useAddresses`, zod schemas + **versioned migrations** (v2 → v3 → current) |
| `AddressBook.merge.ts` (NEW) | `normalizeStoredAddress`, `dedupeAddresses`, `buildAddresses`, `selectAddresses`, `isVisibleToWallet`, `getAllAddresses` |
| `AddressBookEntry.tsx` (NEW) | Single contact row (rename via `EditableText`, copy, delete) |
| `AddressBookModal.tsx` | Contact list modal (rewritten) |
| `AddressBookEmptyState.tsx` | Empty state (expanded) |

Current entry shape — note `publicKey` is the dedupe key, `mode` replaces provider-derived typing, and `savedBy` scopes an entry to the wallets that saved it:

```typescript
// packages/web3-connect/src/components/address-book/AddressBook.store.ts
const addressSchema = z.object({
  publicKey: z.string(),
  name: z.string(),
  address: z.string(),
  provider: z.enum(WalletProviderType).optional(),
  mode: modeSchema,                 // Substrate | SubstrateH160 | EVM | Solana | Sui | Near | Zcash
  savedBy: z.array(z.string()).default([]),
  isCustom: z.boolean().optional(),
})
```

`mergeDuplicateEntries()` prefers the user-supplied (`isCustom`) name over the wallet-supplied one, unions `savedBy`, and lowercases EVM addresses/keys. `useWeb3Enable` now hands the store an `AddressInput` (no pre-computed `publicKey`) and filters out `provider: undefined`. The store imports zod as `zod/v4` explicitly.

Account UI in the same wave: `AccountEditButton` + `AccountNameEdit` were **deleted**, replaced by inline editing through the new `EditableText` component in [[wiki/hydration-ui-design-system\|hydration-ui-design-system]]; `AccountActionButton.styled.ts` is new; `AccountFilter` was rewritten on top of `WALLET_ACCOUNT_FILTER_OPTIONS` (its old `AccountFilterOption` / `allAccountFilterOptions` exports are gone — use `WalletAccountFilterOption` from `config/wallet.ts`).

## API hooks

Barrel: `src/hooks/index.ts`.

| Hook | Returns / role |
|---|---|
| `useAccount()` | Discriminated union — `{ isConnected: true, account, accounts, disconnect }` or `{ isConnected: false, account: null, accounts: [], disconnect }` |
| `useWeb3Connect()` | Zustand store (accounts, provider status, `disconnect`) |
| `useWeb3Enable()` | Mutation: enable a provider, hydrate accounts, push them into the address book |
| `useWeb3EagerEnable()`, `useWeb3ConnectInit()` | Session restore on boot |
| `useWeb3ConnectModal()` | Modal open state |
| `useWallet()` | Resolve the active `Wallet` adapter |
| **`useEvmAddress()`** (NEW) | `safeConvertSS58toH160(account.address)` for the connected account |
| `useAccountMultisigs()` / `useMultixClient()` | Multix-backed multisigs for the connected account |
| `useMultisigStore()`, `useMultisigConfigs()` | Selected multisig + stored configs |
| `useSolanaNativeBalance()`, `useSuiNativeBalance()` | Native balances for non-Substrate chains |

There is **no** `useConnect()` / `useDisconnect()` hook — connect goes through `useWeb3Enable()` and disconnect is a field on `useAccount()` / `useWeb3Connect()`.

### `Web3ConnectModal`
Component that renders the wallet selector UI (exported alongside `Web3ConnectButton`).

## Signer interface

Once connected, a signer is available. Signer kind is discriminated by `isPolkadotSigner` / `isEthereumSigner` / `isSolanaSigner` / `isSuiSigner` (exported type-guards), which `useSignAndSubmit` ([[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]]) uses to pick a submission path.

Key signers (`src/signers/`: `EthereumSigner.ts`, `SolanaSigner.ts`, `SuiSigner.ts`):
- **Polkadot signer** — papi `PolkadotSigner` interface; signs extrinsics directly
- **Ethereum signer** (`signers/EthereumSigner.ts`) — viem-backed; supports `getPermit(tx, options)` for EVM-call permits dispatched through Hydration's `EVM_CALL_PERMIT` precompile (`EVM_CALL_PERMIT_ABI`/`_ADDRESS`/`_TYPES` constants). Used in the permit transformation path to submit EVM intents as unsigned papi extrinsics.
- **Solana signer** — `dataToVersionedTx()` is now **async** and takes the `Connection` as first arg (blockhash is fetched, not assumed); `signAndSendBatch` awaits `Promise.all`. A `null` Jito bundle simulation now raises `SolanaTxError.SIMULATION_FAILED` instead of throwing on property access.
- **Sui** — wallet-standard adapter

### Permit gas math extracted (`utils/permitGas.ts`, NEW)

The magic-number gas maths moved out of `EthereumSigner` into three pure helpers so the money-market estimation hooks can reuse them:

```typescript
// packages/web3-connect/src/utils/permitGas.ts
const GAS_LIMIT_SURPLUS_PERCENT = 30n
const GAS_PRICE_SURPLUS_PERCENT = 5n

export const getPermitGasFromWeight = (weight: bigint): bigint => { … }  // clamp(weight / EVM_GAS_TO_WEIGHT, MIN, MAX) + 30%
export const getPermitGasLimit = (gas: bigint, weight: bigint): bigint => { … } // max(gasByWeight, gas) + 30%
export const getPermitGasPrice = (gasPriceBase: bigint): bigint => { … }        // base + 5%
```

`config/evm.ts` constants: `EVM_GAS_TO_WEIGHT = 25_000n`, `EVM_MIN_GAS_LIMIT = 100_000n`, `EVM_MAX_GAS_LIMIT = 15_000_000n`, and new `EVM_DECIMALS = 18`.

## Multisig support (`utils/multisig.ts`)

Helpers for deriving and normalizing multisig accounts using `@polkadot-api/substrate-bindings`:

```typescript
// packages/web3-connect/src/utils/multisig.ts
export function deriveMultisigAddress(
  signers: string[],
  threshold: number,
): string {
  const pubkey = getMultisigAccountId({
    threshold,
    signatories: signers.map((s) => AccountId().enc(s)),
  })
  return safeConvertPublicKeyToSS58(toHex(pubkey))
}

export function normalizeMultisigEntry(entry: MultisigEntry): MultisigPendingTx {
  // multisigAddress, callHash, when, deposit, depositor, approvals
}
```

`utils/multisig.ts` is **not** in the `utils/index.ts` barrel — consumers deep-import it (`@galacticcouncil/web3-connect/src/utils/multisig` from `apps/main/src/providers/MultisigProvider.tsx`, `@/utils/multisig` from `components/multisig/MultisigSetupNew.tsx`).

The multix GraphQL client is now owned by **this** package, not the app: `src/index.ts` re-exports `getMultixSdk`, `multisigsByAccountIdsQuery`, `MultixSdk` and `MultisigsByAccountIdsQuery` from `@galacticcouncil/indexer/multix` ([[wiki/hydration-ui-indexer\|hydration-ui-indexer]]), and `hooks/useAccountMultisigs.ts` instantiates it against `multix.graphql` from [[wiki/hydration-ui-utils\|hydration-ui-utils]]:

```typescript
// packages/web3-connect/src/hooks/useAccountMultisigs.ts
export const useAccountMultisigs = () => {
  const { account } = useAccount()
  const client = useMultixClient()
  const accountId = account?.publicKey ? `hydradx-${account.publicKey}` : ""
  return useQuery(multisigsByAccountIdsQuery(client, accountId ? [accountId] : []))
}
```

## Identity (`utils/identity.ts` + `components/account/AccountIdentity.tsx`)

Queries `Identity.IdentityOf` storage via the [[wiki/hydration\|hydration]] descriptors (papi). Decodes `IDENTITY_INFO_FIELDS = [display, legal, web, email, twitter]` from `Binary` hex into strings; `IdentityInfo` now additionally carries **`deposit: bigint`** (surfaced as "identity reserves" in the UI, commit `bc6e207`). `AccountIdentity` is a `Text`-based component that renders the on-chain display name or falls back to `shortenAccountAddress`; its external link now resolves through `getAccountExplorerLink()` ([[wiki/hydration-ui-utils\|hydration-ui-utils]]) — i.e. **Neckwork**, not Subscan.

## Integration with Hydration

For Hydration (Polkadot):
1. User selects a Substrate extension
2. Extension prompts to select account
3. `useAccount()` returns account address + signer
4. When submitting tx, the papi `PolkadotSigner` signs the extrinsic

For EVM (borrowing module via [[wiki/hydration-ui-money-market\|hydration-ui-money-market]]):
1. User selects MetaMask / Reown AppKit
2. Wallet prompts to select account
3. `useEvmAddress()` / `useAccount()` returns the EVM address
4. `EthereumSigner` signs — permit path for precompile dispatch, plain tx otherwise

## Implementation notes

- **Wallet standard:** `@mysten/wallet-standard` `^0.19.9` for Sui/wallet detection
- **WalletConnect v2:** `@reown/appkit` `^1.8.19` for multi-chain support
- **EVM:** viem `^2.30.0` (hoists to 2.48.x) for the Frontier precompile path — no ethers here (ethers 5.7 lives only in [[wiki/hydration-ui-money-market\|hydration-ui-money-market]])
- **React Query:** `^5.100.14` (bumped from `^5.59.15`); signer/account/multisig queries cached there
- **Barrel discipline:** `utils/` re-exports `errors, ethereum, identity, permitGas, polkadot, signer, wallet` — `multisig` and `solana` are deep-import only. `wallets/NovaWallet` now deep-imports `@/utils/ethereum` + `@/utils/polkadot` to avoid a barrel cycle.

## For developers

To use web3-connect in a module:

```typescript
import { useAccount } from "@galacticcouncil/web3-connect"

export const MyComponent = () => {
  const { account, isConnected } = useAccount()

  if (!isConnected) return <ConnectWalletButton />

  return <div>Connected as {account.address}</div>
}
```

`useAccount()` is a discriminated union: narrow on `isConnected` before touching `account` (`account` is `null` otherwise). There is no `chain` field — use `getWalletModeByAddress(account.address)` or `account.provider`.

In submit-transaction, the signer is retrieved and used to sign extrinsics before [[wiki/papi\|papi]] submission.

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]], [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], [[wiki/hydration-ui-utils\|hydration-ui-utils]], [[wiki/hydration-ui-design-system\|hydration-ui-design-system]], [[wiki/papi\|papi]]
