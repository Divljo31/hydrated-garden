---
{"dg-publish":true,"permalink":"/wiki/xc-sdk/","title":"xc-sdk","tags":["sdk","cross-chain","wallet","transfers","ntt"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"xc-sdk","repo":"sdk","paths":["packages/xc-sdk/src/Wallet.ts","packages/xc-sdk/src/types.ts","packages/xc-sdk/src/FeeSwap.ts","packages/xc-sdk/src/transfer/DataOriginProcessor.ts","packages/xc-sdk/src/transfer/DataDestinationProcessor.ts","packages/xc-sdk/src/platforms/types.ts","packages/xc-sdk/src/platforms/substrate/SubstrateEvm.ts","packages/xc-sdk/src/platforms/evm/EvmPlatform.ts","packages/xc-sdk/src/clients/WormholeTransfer.ts","packages/xc-sdk/src/platforms/near","packages/xc-sdk/src/platforms/zec","packages/xc/docs/ntt-reorg-recovery.md"],"key_exports":["Wallet","TransferBuilder","FeeSwap","Transfer","TransferSourceData","TransferDestinationData","Platform","Call","EvmCall","SubstrateCall","DryRunResult","PlatformAdapter","SubstratePlatform","EvmPlatform","SolanaPlatform","SuiPlatform","SubstrateEvm","EvmClaim","SubstrateClaim","SolanaClaim","SuiClaim","NearPlatform","NearClaim","NearSigner","ZecPlatform","ZecSigner","ChainBalancesResult","WormholeScan","WormholeTransfer"],"tags":["sdk","cross-chain","wallet","transfers","ntt"],"source_count":1,"last_updated":"2026-11-02"}}
---


# xc-sdk

**TL;DR:** `@galacticcouncil/xc-sdk` (**v2.7.0**, papi v2) is the wallet interface and execution layer for cross-chain transfers, with platform adapters for Substrate, EVM, Solana, Sui, **NEAR** and **Zcash**. A transfer is an ordered *sequence* of calls; routes can be one-way (`reversible`); `Wallet.getAllBalances()` reads an address across every chain it is valid on.

## Wallet API

```ts
const transfer = await wallet.transfer(asset, srcAddr, srcChain, dstAddr, dstChain);
await transfer.estimateFee(amount);
await transfer.validate();
const calls = await transfer.buildCalls(amount);  // ordered; transfer call is last
const next  = await transfer.buildCall(amount);   // head of buildCalls()

// multichain balances
wallet.getChainsForAddress(address, chains);      // chains whose address format accepts it
await wallet.getAllBalances(address, chains?);   // Promise.allSettled per chain → ChainBalancesResult[] { chainKey, balances, error? }
```

`Wallet.getTransferData()` resolves balances, minimums, EDs, source + destination fees and fee-swap opportunities in parallel, then re-estimates the source fee with a 5% margin (`padByPct(5n)`) if a fee swap applies.

## `Transfer` shape (`src/types.ts`)

```ts
// packages/xc-sdk/src/types.ts
export interface Transfer {
  source: TransferSourceData;
  destination: TransferDestinationData;
  /** True if a return-trip route is registered. Gate UI swap-direction on it. */
  reversible: boolean;
  buildCall(amount): Promise<Call>;    // head of buildCalls
  buildCalls(amount): Promise<Call[]>; // ordered; prerequisites first, transfer last
  estimateFee(amount): Promise<AssetAmount>;
  estimateDestinationFee(amount): Promise<AssetAmount>;
  validate(fee?, amount?): Promise<TransferValidationReport[]>; // amount overrides ctx.amount
}
```

`buildCalls()` returns **one call per signature the sender has to give**, which is not one per config: a batching platform (Substrate) collapses them, a non-batching one (EVM, Solana) may also *prepend* account prerequisites (`WETH.deposit`, `erc20.approve`) derived by `getEvmPrerequisites()` in [[wiki/xc-core\|xc-core]]. Each prerequisite drops off once executed, so re-deriving after signing yields a shorter list.

## Unidirectional routes

`DataReverseProcessor` is **deleted**, replaced by `DataDestinationProcessor` — a thin `DataProcessor` constructed with `(chain, asset)` directly instead of inferring them from a reverse `TransferConfig`. That lets a route resolve destination balance / min / ED with no return-trip route registered; `TransferConfigs.reversible` (from `ChainRoutes.getAssetDestinationRoutesOrEmpty()`) is surfaced on the DTO. Spec: `packages/xc/docs/unidirectional-routes.md`.

## Signer-origin selection

`DataOriginProcessor.getTransfer()` picks the execution path from the route builders:

```ts
// packages/xc-sdk/src/transfer/DataOriginProcessor.ts
const { contract, deposit, extrinsic, functionCall, program, move } = route;
if (extrinsic && !(contract && EvmAddr.isValid(ctx.sender))) { /* extrinsic path */ }
const callable = contract || program || move || functionCall || deposit;
```

An **h160 sender takes `contract`** and signs the EVM calls itself; anyone else takes `extrinsic`, which on an EVM parachain wraps the very same calls in `EVM.call` + `Utility.batch_all` (`SubstrateEvm`) — one signature, fee quoted by the runtime in the sender's fee currency. This is the dual-signer-origin mechanism the NTT / Snowbridge templates in [[wiki/xc-cfg\|xc-cfg]] rely on.

`SubstrateEvm` (`src/platforms/substrate/SubstrateEvm.ts`) also backs manual claims; it derives the EVM source with `chain.getDerivatedAddress(from)`, pads `maxFeePerGas` by 10% over the current gas price, and encodes U256 values as 4 little-endian u64 limbs for papi. Requires the account to be **bound** on chain (`EVMAccounts.bind_evm_address`), because `EnsureAddressTruncated` otherwise resolves to the unrelated `ETH\0` phantom account — see `apps/src/rescue/` for the recovery helper.

## Platform Adapters

`Platform<T extends BaseConfig>` (`src/platforms/types.ts`) is now `buildCalls(account, amount, feeBalance, configs) => Promise<Call[]>` plus `estimateFee(...)` — plural configs in, plural calls out.

- **SubstratePlatform** — Substrate extrinsics; call data hex via `Binary.toHex`, subscriptions via `fn.watchValue(...args, { at: 'best' })`.
- **EvmPlatform** — viem contract calls; prepends prerequisites, appends follow-up calls (e.g. `Executor.requestExecution`).
- **SolanaPlatform** / **SuiPlatform** — programs / moves. (`SolanaLilJit` / Jito-bundle claiming was removed.)
- **NearPlatform** (`platforms/near/`) — `Platform<FunctionCallConfig>`; `NearSigner`, `NearClaim` (claim an NTT transfer on NEAR with `complete`), utils. Uses `NearClient` from [[wiki/xc-core\|xc-core]].
- **ZecPlatform** (`platforms/zec/`) — `Platform<DepositConfig>`; a Zcash transparent transfer to the Omni deposit address; `ZecSigner`.

`Call` carries `{ from, data, type: CallType, dryRun() }`; `EvmCall` extends it with `{ to, value?, abi?, allowance? }` (the `allowance` field is what marks an approve step in a UI).

## Balance reads moved out

All per-platform balance implementations (`platforms/{evm,solana,sui}/balance/*`) are **deleted** — balances are read from the chain object in [[wiki/xc-core\|xc-core]] (`chain.getBalance()` / `subscribeBalance()`), driven by each chain's declared `balance:` / `balanceOverrides:`. `Wallet.getBalances()` / `subscribeBalance()` now delegate there, batched and narrowed.

## Fee Handling

Source transaction fee, destination execution fee, and optional fee swaps (converting between fee assets via `HydrationDex` from [[wiki/xc-cfg\|xc-cfg]]). `FeeSwap` gained a spec (`src/FeeSwap.spec.ts`); `getDestinationFee(amount?, address?)` takes the recipient because some fees depend on accounts the delivery opens there (an SVM redeem opening the recipient's ATA) and must match what the transfer was built with — the Wormhole Executor honours a quote only for what it priced.

`calculateMax(srcBalance, srcFee, srcMin, srcEd, reserves)` now also reserves a **prepaid** destination fee (`destination.fee.prepaid`, e.g. Executor delivery) and the input of a destination-fee swap (+5%) — the old over-stated `max` on `weth_wh → eth` executor transfers is fixed.

## Transfer Validation

`TransferValidator` runs the `validations` array from [[wiki/xc-cfg\|xc-cfg]]: balance sufficiency, fee adequacy, minimums, route availability, plus the new NTT rate-limit checks.

## NTT messages lost to a Hydration reorg

`packages/xc/docs/ntt-reorg-recovery.md`: Frontier's synthetic EVM tx hash embeds the substrate block hash, so a re-included extrinsic after a reorg surfaces under a **different** EVM hash; guardians keyed on the first hash can drop the message although the burn is final. The doc covers finding the surviving tx and getting the VAA signed.

## Signing

Platform-specific signers: `SubstrateSigner` (polkadot-api), `EvmSigner` (viem WalletClient), `SolanaSigner`, `SuiSigner`, `NearSigner`, `ZecSigner`. The SDK builds transactions but does not sign — the consumer provides the signer.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
