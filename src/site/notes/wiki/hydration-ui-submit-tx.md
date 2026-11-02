---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-submit-tx/","title":"hydration-ui transaction submission flow","tags":["transactions","signing","submission","hydration","multisig","permit"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"hydration-ui transaction submission flow","repo":"hydration-ui","paths":["apps/main/src/modules/submit-transaction/SubmitTransactionPage.tsx","apps/main/src/states/transactions.ts","apps/main/src/modules/transactions/TransactionManager.tsx","apps/main/src/modules/transactions/hooks/useSignAndSubmit.ts","apps/main/src/modules/transactions/hooks/useNonce.ts","apps/main/src/modules/transactions/utils/permitFee.ts","apps/main/src/modules/transactions/types/index.ts","apps/main/src/modules/transactions/utils/near.ts","apps/main/src/modules/transactions/utils/zec.ts","apps/main/src/modules/transactions/utils/xcSwap.ts","apps/main/src/modules/transactions/hooks/useIntentToasts.ts","apps/main/src/modules/transactions/utils/toasts"],"symbols":["SubmitTransactionPage","TransactionManager","useTransactionsStore","ReviewMultisig","useSignAndSubmit","useNonce","PendingTransaction","BRIDGE_PROVIDER_TAGS","signAndSubmitNearTx","signAndSubmitZecTx","getXcSwapSequence","useIntentToasts","useTxCallData","TransactionIntentMeta","TransactionXcSwapMeta","TransactionClosedError","getTxResultBlockHeight"],"key_deps":["@galacticcouncil/web3-connect","@galacticcouncil/sdk-next","@galacticcouncil/xc-core","polkadot-api","viem"],"tags":["transactions","signing","submission","hydration","multisig","permit"],"last_updated":"2026-11-02"}}
---


# hydration-ui transaction submission flow

**TL;DR:** Centralized tx signing & submission. In-app flows go through the `transactions/` module (`useTransactionsStore.createTransaction` → `ReviewTransaction` / `ReviewMultiTransaction` → `useSignAndSubmit`); the standalone `/submit-transaction` page handles externally-supplied extrinsics. Built on [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] for signers and [[wiki/papi\|papi]] for submission.

Signer paths cover Polkadot, EVM (incl. permits), Solana, Sui, **NEAR** and **Zcash** (verified at `770f868`). ICE intents and cross-chain swaps (`XcSwap`) carry their own tx meta and toasts.

## Flow overview

1. **Construct** the tx — [[wiki/sdk-next\|sdk-next]] (trades, intents), `rpc.papi.tx.*`, a viem EVM call, or an xc-sdk NEAR/Zcash/Solana/Sui call
2. **Queue** it — `useTransactionsStore().createTransaction(input, options)`
3. **Review** — `TransactionManager` renders `ReviewTransaction` / `ReviewMultiTransaction` (fee, tip, mortality, JSON view)
4. **Sign & submit** — `useSignAndSubmit` picks the path by signer × tx kind
5. **Track** — per-tx subscriptions call `onSubmitted` / `onSuccess` / `onError`; toasts via sonner; `useNeckworkSync` refreshes indexer-backed views

`/submit-transaction` is only for call data supplied from outside the app.

## Extrinsic review: `SubmitTransactionPage.tsx`

Displays:
- Decoded extrinsic parameters (e.g., "Swap 100 AUSD for HDX")
- Estimated fee + tip
- Gas estimate (if EVM-related via [[wiki/pallet-frontier\|pallet-frontier]])
- Account selector
- Sign button

Powered by:
- [[wiki/papi\|papi]]-based type decoding
- [[wiki/hydration\|hydration]] descriptors for pallet metadata
- react-hook-form for form state

## Signer selection: [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]]

The Web3Connect modal integrates:
- Polkadot.js browser extension (PJS)
- WalletConnect (mobile)
- EVM wallets (MetaMask, etc. via viem)
- Multisig support (Polkasafe, etc.)

The flow selects an account and retrieves the signer function (sign_extrinsic or sign_payload).

## Multisig transaction review: `ReviewMultisig`

For multisig accounts, the review flow displays:
- List of required signatories
- Current approval count
- Status of each approver (signed, pending, expired)
- Multisig threshold visualization

Located in `apps/main/src/modules/transactions/review/ReviewMultisig/`. Integrates with [[wiki/hydration-ui-api\|hydration-ui-api]] `multisig.ts` hooks to track approval state.

## Submission & tracking

```
┌──────────────────────────────────────────────────┐
│ 1. Sign extrinsic (via signer from web3-connect) │
├──────────────────────────────────────────────────┤
│ 2. Submit to papi RPC (await broadcast)          │
├──────────────────────────────────────────────────┤
│ 3. Monitor finalization (in-block, finalized)    │
├──────────────────────────────────────────────────┤
│ 4. Emit events (fee paid, rewards, etc.)         │
├──────────────────────────────────────────────────┤
│ 5. Update Zustand store + toast user             │
└──────────────────────────────────────────────────┘
```

After submission, the `TransactionManager` (lazy-loaded component in `__root.tsx`) polls tx status and updates the `transactions` Zustand store.

## `useSignAndSubmit` — multi-chain signer dispatch

`apps/main/src/modules/transactions/hooks/useSignAndSubmit.ts` is the central mutation hook used by the review modal. It type-discriminates on signer kind (`isPolkadotSigner`, `isEthereumSigner`, `isSolanaSigner`, `isSuiSigner`, `isNearSigner`, `isZecSigner` from `@galacticcouncil/web3-connect`) and on the tx payload kind (`isPapiTransaction`, `isEvmCall`, `isSolanaCall`, `isSuiCall`, `isNearCall`, `isZecCall`), then routes to the appropriate `signAndSubmit*` utility under `modules/transactions/utils/`:

| Signer × tx kind | Path |
|------------------|------|
| EVM-call + Ethereum signer (permit eligible) | `signer.getPermit(tx)` → `transformPermitToPapiTx` → `submitUnsignedPolkadotTx` |
| Papi tx + Ethereum signer (permit eligible) | encoded data → `signer.getPermit(hex)` → permit tx → unsigned submit |
| Papi tx + Polkadot signer | `signAndSubmitPolkadotTx` |
| Papi tx + Ethereum signer | `signAndSubmitEvmDispatchTx` (EVM dispatch precompile) |
| EVM call + Ethereum signer | `signAndSubmitEvmTx` |
| EVM call + Polkadot signer | wrapped Polkadot dispatch |
| Solana / Sui tx | `signAndSubmitSolanaTx` / `signAndSubmitSuiTx` |
| NEAR call + NEAR signer | `signAndSubmitNearTx` (`utils/near.ts`) |
| Zcash call + Zec signer | `signAndSubmitZecTx` (`utils/zec.ts`, payment built by Noir) |
| Unsigned papi tx | `submitUnsignedPolkadotTx` |

Subscriptions are tracked in `utils/subscriptions.ts` (`addTxSubscription` / `deleteTxSubscription`) so they can be cleaned up on unmount.

Wormhole/NTT transfers submit as ordinary papi/EVM transactions via the NTT client in [[wiki/xc-core\|xc-core]]. `TxOptions.signerFeeAsset?: Asset` (from `@galacticcouncil/xc-core`) lets cross-chain legs declare the asset the *source-chain* signer pays gas in.

### Intents, cross-chain swaps, toasts

- `TransactionCommon.intent?: TransactionIntentMeta` (`{ id?, assetIn, assetOut, amountIn, minAmountOut }`) marks ICE intent submissions; `hooks/useIntentToasts.ts` + `utils/toasts/intents.ts` / `intentOutcomes.ts` follow the intent to fill / expiry via Neckwork.
- `TransactionType.XcSwap` + `TransactionXcSwapMeta` (src/dst asset, amounts, fees, `dstAddress`, `sequence`, `depositAddress`, `correlationId`); `utils/xcSwap.ts` → `getXcSwapSequence(result)`.
- `utils/toasts/extrinsics.ts`, `successToastUpdate.ts` — toast pipeline; `hooks/useTxCallData.ts` — encoded call data for review.
- `getTxResultBlockHeight(result)` (Hydration block for substrate/EVM results, `null` for Solana/Sui) feeds `useNeckworkSync` arming ([[wiki/hydration-ui-indexer\|hydration-ui-indexer]]).

## Permit fee estimation: `utils/permitFee.ts`

Estimating the cost of an EVM-signer transaction that dispatches a substrate call through the EVM dispatch precompile. Previously approximated; now computed end-to-end:

1. `getPermitWeight(rpc, address, anyTx)` — dry-run weight of the wrapped call
2. `getPermitGas` / `getPermitGasLimit(gas, weight)` / `getPermitGasPrice(base)` (from `@galacticcouncil/web3-connect/src/utils/permitGas`) against `rpc.evm.getGasPrice()`
3. `dispatchFeeWei = gasLimit × gasPrice` — wallets validate balance against the *limit*, even though the chain only charges `gas_used × gas_price`
4. `+ getNestedEvmCallsFeeWei(anyTx)` when the call contains nested EVM calls (`containsEvmCall`)
5. Converted to the display fee asset via `spotPriceQuery` / `paymentInfoQuery`

Returns `{ feeEstimateNative, feeEstimate, feeAssetId }`. Feeds `useEstimateFee` and the `useMaxBalanceWithFee` family (see [[wiki/hydration-ui-modules\|hydration-ui-modules]]).

## Nonce handling: `hooks/useNonce.ts`

Two nonce spaces: the **permit nonce** (`useAccountPermitNonce`, EVM signer paying a non-native fee asset) and the **substrate nonce**. The substrate nonce is now optimistically advanced from the pending-tx queue so rapid successive submissions don't collide:

```typescript
// apps/main/src/modules/transactions/hooks/useNonce.ts
const pendingNonces = pendingTransactions
  .filter(
    (tx) =>
      !tx.isPermit &&
      tx.address === account?.address &&
      tx.meta.srcChainKey === HYDRATION_CHAIN_KEY &&
      tx.nonce >= chainNonce,
  )
  .map((tx) => tx.nonce)

// Optimistically increment the substrate nonce for the next transaction
const substrateNonce = pendingNonces.length
  ? Math.max(...pendingNonces) + 1
  : chainNonce
```

`useAccountPermitNonce` is now `enabled: isUsingPermit` (was fetched unconditionally in `__root.tsx`).

## Review components

- `ReviewMultiTransaction.tsx` — Stepper modal for `MultiTransaction` (array of `SingleTransaction`). Resolves promise/function-typed `tx` entries, threads `TSuccessResult[]` through subsequent steps, blocks route changes mid-flight via `useRouteBlock`.
- `ReviewTransactionJsonView/` — JSON inspector for the decoded extrinsic (call hash, encoded args)
- `ReviewTransactionSummary.tsx` — Compact summary card

## Zustand store: `states/transactions.ts`

The store is a **queue + promise bridge**, not a history log. `createTransaction(input, options)` pushes a `Transaction` (or `MultiTransaction`) and returns a promise that the review modal resolves.

```typescript
// apps/main/src/states/transactions.ts
interface TransactionsStore {
  transactions: Transaction[]
  pendingTransactions: PendingTransaction[]
  createTransaction: (
    transaction: TransactionInput,
    options?: TransactionOptions,
  ) => Promise<TSuccessResult | void>
  cancelTransaction: (id: string) => void
  addPendingTransaction: (transaction: PendingTransaction) => void
  removePendingTransaction: (id: string) => void
}

export type PendingTransaction = {
  id: string
  meta: TransactionMeta
  nonce: number
  address: string
  isPermit: boolean
}
```

Store features:

| Feature | Detail |
|---|---|
| **Cross-tab sync** | `new BroadcastChannel("hydration:pending-tx")`; `addPendingTransaction` / `removePendingTransaction` broadcast `{type:"add"\|"remove"}` and an `onmessage` handler mirrors (dedup by id) into other tabs' stores — so nonce math in `useNonce` accounts for txs signed in another tab |
| **`resolveOn`** | `TransactionOptions.resolveOn?: "submitted" \| "success"` (default `"submitted"`); `onSubmitted` resolves the promise early so UIs can close the modal before finalization. Return type widened to `Promise<TSuccessResult \| void>` |
| **`activity`** | `TransactionCommon.activity?: ActivityType` (from `@galacticcouncil/utils`) merged into `meta`; classifies a tx for toasts/history |
| **`executedAmount`** | `TExecutedAmount = { amount: string; assetId: string }` on `TransactionCommon` — the realized amount for success toasts |
| **`signerFeeAsset`** | `Asset` from `@galacticcouncil/xc-core`, threaded to `TxOptions` |
| **`BRIDGE_PROVIDER_TAGS`** | `[Basejump, NttExecutor, Wormhole, Snowbridge]` — order is load-bearing (`getPrimaryBridgeTag` takes the first match) |
| **Step hooks** | `SingleTransaction.beforeNext?: () => Promise<void>` (awaited before the stepper advances), `initialError?` (open review in error state) |
| **Close** | closing the modal rejects with `TransactionClosedError` (was a bare string) |

`XcmTag = tags.Tag` is re-exported from [[wiki/xc-cfg\|xc-cfg]].

Components subscribe to the store to drive review modals and toast processing.

## TransactionManager: modal host + toast pump

Lazy-loaded in `__root.tsx` inside `<Services />`. It is a **renderer**, not a poller — it maps `useTransactionsStore().transactions` to review modals and pumps toasts:

```typescript
// apps/main/src/modules/transactions/TransactionManager.tsx
export const TransactionManager = () => {
  const { transactions } = useTransactionsStore()
  const { toasts } = useToasts()

  const transactionToasts = toasts.filter(isTransactionToast)

  useProcessTransactionToasts(transactionToasts)

  return transactions.map((transaction) =>
    isMultiTransaction(transaction) ? (
      <ReviewMultiTransaction key={transaction.id} transaction={transaction} />
    ) : (
      <TransactionProvider key={transaction.id} transaction={transaction}>
        <ReviewTransaction />
      </TransactionProvider>
    ),
  )
}
```

Status/finalization tracking happens per-tx in `utils/subscriptions.ts` + the `signAndSubmit*` utilities, which call back into `onSubmitted` / `onSuccess` / `onError`.

## Integration with modules

In-app flows do **not** navigate to `/submit-transaction`. They construct a tx (via [[wiki/sdk-next\|sdk-next]], papi, or a viem EVM call) and hand it to the store:

```typescript
// pattern used across modules/*/**.submit.ts and *.tx.ts
const { createTransaction } = useTransactionsStore()

await createTransaction(
  { tx: rpc.papi.tx.GigaHdx.giga_stake({ amount }), title, activity },
  { resolveOn: "success", onSuccess: () => queryClient.invalidateQueries(...) },
)
```

`/submit-transaction` remains the entry point for extrinsics supplied from outside the app (deep links, external tooling).

## Error handling

Errors at each stage are caught and displayed:
- **Construction:** "Invalid trade route" → show error banner
- **Signing:** "User rejected signature" → dismiss modal, return to form
- **Submission:** "Insufficient balance" → tx failure toast
- **Finalization:** "Extrinsic failed" (from decoded events) → error toast with details

## For developers

To submit a tx from a module:
1. Construct the tx — [[wiki/sdk-next\|sdk-next]] for trades, `rpc.papi.tx.*` for direct extrinsics, or an `ExtendedEvmCall` for EVM (wrapped by `Dispatcher.dispatch_evm_call`)
2. Call `useTransactionsStore().createTransaction(input, options)` — set `activity`, `invalidateQueries`, and `resolveOn` as needed
3. `TransactionManager` renders the review modal; `useSignAndSubmit` picks the signer path
4. Await the returned promise, or use `onSuccess` / toast processors for side effects

For multi-step flows, pass a `MultiTransaction` (`tx` entries may be functions or promises resolved lazily) — `ReviewMultiTransaction` drives the stepper and blocks route changes via `useRouteBlock`.

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui-main-app\|hydration-ui-main-app]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/papi\|papi]], [[wiki/sdk-next\|sdk-next]], [[wiki/xc-core\|xc-core]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/hydration\|hydration]]
