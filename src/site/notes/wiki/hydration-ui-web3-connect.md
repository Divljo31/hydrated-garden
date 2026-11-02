---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-web3-connect/","title":"hydration-ui web3-connect","tags":["wallet","web3","authentication","hydration","multisig","address-book","near","zcash"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui web3-connect","repo":"hydration-ui","paths":["packages/web3-connect/src","packages/web3-connect/src/index.ts","packages/web3-connect/src/config/wallet.ts","packages/web3-connect/src/config/providers.ts","packages/web3-connect/src/config/modal.ts","packages/web3-connect/src/config/evm.ts","packages/web3-connect/src/components/Web3ConnectSession.tsx","packages/web3-connect/src/components/content","packages/web3-connect/src/components/address-book","packages/web3-connect/src/signers","packages/web3-connect/src/wallets","packages/web3-connect/src/utils/walletSource.ts","packages/web3-connect/src/utils/permitGas.ts","packages/web3-connect/src/utils/multisig.ts"],"key_exports":["Web3ConnectModal","Web3ConnectSession","Web3ConnectButton","Web3ConnectModalPage","Web3ConnectAccountIdentity","AccountWalletAvatar","useAccount","useWallet","useWeb3Connect","useWeb3Enable","useWeb3EagerEnable","useWeb3ConnectModal","useAccountMultisigs","useEvmAddress","useMultisigStore","useAddressStore","useAddresses","AddressBookModal","getMultixSdk","multisigsByAccountIdsQuery"],"symbols":["WalletMode","WalletProviderType","WALLET_ACCOUNT_FILTER_OPTIONS","Web3ConnectModalPage","addressToPublicKey","groupWalletsBySource","selectWalletSources","getConnectAllGroups","isPolkadotSigner","isEthereumSigner","isSolanaSigner","isSuiSigner","isNearSigner","isZecSigner","EthereumSigner","SolanaSigner","SuiSigner","NearSigner","ZecSigner","BaseNearWallet","getPermitGasFromWeight","getPermitGasLimit","getPermitGasPrice","EVM_CALL_PERMIT_ADDRESS","EVM_DISPATCH_ADDRESS","deriveMultisigAddress","useMultixClient"],"key_deps":["@reown/appkit","@hot-labs/near-connect","@noir-wallet/sdk","@mysten/wallet-standard","@solana/web3.js","viem","bs58","@tanstack/react-query","@galacticcouncil/indexer"],"tags":["wallet","web3","authentication","hydration","multisig","address-book","near","zcash"],"last_updated":"2026-11-02"}}
---


# hydration-ui web3-connect

**TL;DR:** `@galacticcouncil/web3-connect` (`packages/web3-connect/`) — wallet connection, accounts, signers and the address book for Substrate, EVM, Solana, Sui, **NEAR** and **Zcash**. Since `770f868` NEAR and Zcash are real signing chains (NEAR via `@hot-labs/near-connect`, Zcash via the Noir wallet), the modal is the redesigned "wallet management" v2, and session lifetime moved into a separate `Web3ConnectSession` component.

## Wallets — `src/wallets/`

| Chain | Adapters |
|---|---|
| Substrate | PolkadotJS, Talisman, SubWallet, Nova, Enkrypt, Fearless, PolkaGate, Manta, AlephZero (`BaseSubstrateWallet`) |
| EVM / H160 | MetaMask, Rabby, Brave, Talisman/SubWallet/Nova EVM + H160 variants, Reown WalletConnect (`BaseEIP1193Wallet`, `ReownWalletConnect`) |
| Solana | Phantom, Solflare, Backpack, Brave, MetaMask, OKX, Talisman, Trust (`BaseSolanaWallet`) |
| Sui | Slush, Suiet, Phantom, Backpack, OKX (`BaseSuiWallet`) |
| NEAR | HOT, Meteor, Intear, Ledger, Ethereum wallets (`BaseNearWallet` over `@hot-labs/near-connect`) |
| Zcash | Noir (`NoirWallet`, `@noir-wallet/sdk`) |
| Other | `ExternalWallet` (watch-only address), `Multisig` |

Provider ids: `WalletProviderType` enum in `config/providers.ts` (e.g. `hot-wallet-near`, `meteor-wallet`, `noir-wallet`, `okx-wallet-sol`, `backpack-sui`, `walletconnect`, `multisig`, `external`).

## `WalletMode` — `config/wallet.ts`

```typescript
export enum WalletMode {
  Default = "default", EVM = "evm", Substrate = "substrate",
  SubstrateEVM = "substrate-evm", SubstrateH160 = "substrate-h160",
  Solana = "solana", Sui = "sui", Near = "near", Zcash = "zcash",
}
```

`WALLET_ACCOUNT_FILTER_OPTIONS` lists the filterable modes; per-mode metadata (name, icon, `chain: true`) lives in the same file. `utils/walletSource.ts` groups providers by source (`groupWalletsBySource`, `selectWalletSources`, `getConnectAllGroups`, `getWalletPrimaryMode`) for the modal; `utils/accountFilter.ts` filters accounts by mode. `addressToPublicKey(address)` (now `utils/publicKey.ts`) is the identity/dedupe key.

## Modal & session

- **`Web3ConnectModal`** — pure view over the store; mounted several times (globally in `__root.tsx` with `neckwork` + `papi` props, and inside cross-chain modals). Pages: `Web3ConnectModalPage.{Wallets, ExternalWallet, MultisigSetup, MultisigConfigSelect, MultisigSignerSelect}` (persisted `initialPage`; unknown values fall back to `Wallets`). Content: `WalletManagementContent` / `WalletManagementSource` / `WalletManagementAccounts` / `WalletManagementStates`, `ExternalWalletContent` (+ `RecentExternalWallets.store.ts`), multisig setup/select. Account rows show balances (`useAccountsWithBalance`, `useAccountBalancesMap`, `useBalanceCache`) and a wallet avatar (`AccountWalletAvatar`).
- **`Web3ConnectSession`** — renders nothing; runs `useWeb3EagerEnable()` + `useWalletSubscriptions()`. Must be mounted **exactly once** (the app does it in `Services`).

## Hooks (`src/hooks/index.ts` barrel)

`useAccount` (discriminated union on `isConnected`), `useWeb3Connect` (store), `useWeb3Enable`, `useWeb3EagerEnable`, `useWeb3ConnectInit`, `useWeb3ConnectModal`, `useWallet`, `useEvmAddress`, `useAccountMultisigs` / `useMultixClient`, `useMultisigStore`, `useMultisigConfigs`. Non-barrel: `useWalletProviders`, `useWalletSubscriptions`, `useActivateMultisig`, `useAccountsWithBalance`, `useAccountBalancesMap`, `useBalanceCache`, `useReorderAnimation`.

## Signers — `src/signers/`

Type guards in `utils/signer.ts`: `isPolkadotSigner`, `isEthereumSigner`, `isSolanaSigner`, `isSuiSigner`, **`isNearSigner`**, **`isZecSigner`** — `useSignAndSubmit` dispatches on them ([[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]]).

- Polkadot — papi `PolkadotSigner`.
- `EthereumSigner` — viem; `getPermit()` for EVM-call permits via `EVM_CALL_PERMIT_ADDRESS` (dispatch precompile `EVM_DISPATCH_ADDRESS = 0x…0401`).
- `SolanaSigner`, `SuiSigner` — wallet-standard.
- **`NearSigner`** — wraps `xc-sdk` `NearSigner`; batches via near-connect `signAndSendTransactions`; outcome reporting in `utils/near.ts`.
- **`ZecSigner`** — Noir builds and broadcasts the payment; fee = `ZEC_LOGICAL_ACTIONS × ZEC_MARGINAL_FEE` from [[wiki/xc-sdk\|xc-sdk]]; `utils/zec.ts`.

## Permit gas — `utils/permitGas.ts`

`getPermitGasFromWeight` (clamp `weight / EVM_GAS_TO_WEIGHT` to `[EVM_MIN_GAS_LIMIT, EVM_MAX_GAS_LIMIT]` + 30 %), `getPermitGasLimit` (max of weight-gas and gas, + 30 %), `getPermitGasPrice` (+5 %). `config/evm.ts`: `EVM_GAS_TO_WEIGHT = 25_000n`, `EVM_MIN_GAS_LIMIT = 100_000n`, `EVM_MAX_GAS_LIMIT = 15_000_000n`, `EVM_DECIMALS = 18`.

## Address book — `components/address-book/`

Persisted Zustand store (`AddressBook.store.ts`: `useAddressStore`, `useAddresses`, zod schema + versioned migrations) keyed by `publicKey`, entries `{ publicKey, name, address, provider?, mode, savedBy[], isCustom? }`. Merge/dedupe helpers moved to `AddressBook.utils.ts` (tested). UI: `AddressBook`, `AddressBookModal`, `AddressBookEntry`, `AddressBookSearch`, `AddressBookButton`.

## Multisig

`utils/multisig.ts` (`deriveMultisigAddress`, `normalizeMultisigEntry`) — deep-import only, not in the `utils` barrel. Multix client re-exported from [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] (`getMultixSdk`, `multisigsByAccountIdsQuery`); `useMultixClient()` instantiates it against `multix.graphql` ([[wiki/hydration-ui-utils\|hydration-ui-utils]]); account id format `hydradx-<publicKey>`.

## Identity

`utils/identity.ts` reads `Identity.IdentityOf` (display, legal, web, email, twitter + `deposit`); `AccountIdentity` (exported as `Web3ConnectAccountIdentity`) links via `getAccountExplorerLink` → Neckwork.

## Notes

- `utils/index.ts` barrel: `errors`, `ethereum`, `identity`, `permitGas`, `polkadot`, `publicKey`, `signer`, `wallet`. `multisig`, `solana`, `near`, `zec`, `walletSource`, `accountFilter` are deep-import.
- Tests: Vitest (`vitest.config.ts`) over `config/wallet.test.ts`, `utils/*.test.ts`, `AddressBook.utils.test.ts`.
- No `useConnect` / `useDisconnect` — connect via `useWeb3Enable()`, disconnect is on `useAccount()`.

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]], [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], [[wiki/hydration-ui-utils\|hydration-ui-utils]], [[wiki/hydration-ui-design-system\|hydration-ui-design-system]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/papi\|papi]]
