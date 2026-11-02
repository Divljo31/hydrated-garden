---
{"dg-publish":true,"permalink":"/wiki/pallet-lazy-executor/","title":"pallet-lazy-executor","tags":["ice","intents","evm","callbacks","ocw","queue","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-lazy-executor","repo":"hydration-node","paths":["pallets/lazy-executor/src/lib.rs","traits/src/lazy_executor.rs","pallets/intent/contracts/IIntentResolutionReceiver.sol","runtime/hydradx/src/assets.rs"],"symbols":["Pallet","Config","ForwardAction","Source","Mutate","MAX_FORWARD_DATA","CallQueue","Sequencer","DispatchNextId","MaxTxPerBlock","MaxCallWeight","StoredForward","dispatch_top","add_to_queue","IntentForwardGasLimit"],"traits_impl":["Mutate","ValidateUnsigned"],"depends_on":["pallet-intent","pallet-evm-accounts","pallet-currencies","hydration-precompiles"],"runtime_index":97,"tags":["ice","intents","evm","callbacks","ocw","queue","runtime","rust","substrate"],"last_updated":"2026-11-02"}}
---


# pallet-lazy-executor

**TL;DR:** FIFO queue of post-resolution **forwards** for [[wiki/ice\|ice]] intents. When an intent with `OnResolved::Forward { contract, data }` is resolved, [[wiki/pallet-intent\|pallet-intent]] queues a `ForwardAction`; an OCW later submits unsigned `dispatch_top`, which pushes the resolved output to the named EVM contract and calls its `IIntentResolutionReceiver` hook as a best-effort EVM call. Push + call run in one storage transaction and roll back on revert / wrong ack, leaving the owner's funds untouched. Runtime index = 97.

## Types (`traits/src/lazy_executor.rs`)

```rust
pub const MAX_FORWARD_DATA: u32 = 512;
pub enum Source { ICE(Identificator) }
pub struct ForwardAction {
    pub contract: EvmAddress, pub intent_id: u128,
    pub asset_in: AssetId, pub amount_in: Balance,
    pub asset_out: AssetId, pub amount_out: Balance,
    pub data: BoundedVec<u8, ConstU32<MAX_FORWARD_DATA>>,
}
pub trait Mutate<AccountId> {
    type Error;
    fn queue(src: Source, origin: AccountId, forward: ForwardAction) -> Result<(), Self::Error>;
}
```

Queued forwards are typed data, **not arbitrary `RuntimeCall`s**.

## Config (runtime)

`RuntimeCall` (only for the unsigned `dispatch_top` + fee machinery), `Currency = Currencies`, `Evm = evm::Executor<Runtime>`, `EvmAccounts = EVMAccounts`, `Erc20Mapping = HydraErc20Mapping`, `GasWeightMapping = FixedHydraGasWeightMapping`, `GasLimit = IntentForwardGasLimit` (2,000,000), `EvmErrorDecoder`, `UnsignedLongevity = 2`, `UnsignedPriority = 100`.

## Storage

| Name | Kind | Value |
|---|---|---|
| `CallQueue` | Map | `CallId (u128) → StoredForward<AccountId>` |
| `Sequencer` | Value | next id to assign |
| `DispatchNextId` | Value | queue cursor (next id to execute) |
| `MaxTxPerBlock` | Value | OCW submissions per block |
| `MaxCallWeight` | Value | per-forward weight cap |

## Extrinsics

| # | Name | Origin | Description |
|---|---|---|---|
| 1 | `dispatch_top(_id)` | none (OCW) | Execute the forward at `DispatchNextId`; weight = base + `gas_to_weight(GasLimit)` |

`add_to_queue(src, owner, action)` (via `Mutate::queue`) is internal: it **charges the execution fee up front** from the owner and fails if they can't pay (`FailedToPayFees`).

Events: `Queued`, `Executed`. Errors: `IdOverflow`, `Overflow`, `FailedToPayFees`, `FailedToDepositFees`, `EmptyQueue`, `Overweight`, `ForwardFailed`.

## Hooks

`offchain_worker`: from `DispatchNextId`, submits up to `MaxTxPerBlock` `dispatch_top` unsigned txs while `CallQueue` has entries; submission is best-effort (a full pool leaves the job queued).

## Gotchas

- Strict FIFO: `dispatch_top` always consumes the job at the cursor, regardless of the `_id` argument.
- Forward failure (revert / wrong ack) is non-fatal for the user: the push is rolled back, resolved funds stay with the owner, `Executed` reports the decoded EVM error.
- Receiver contracts must implement `pallets/intent/contracts/IIntentResolutionReceiver.sol`; `InvalidForwardContract` is raised at intent submission for bad targets.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/ice\|ice]], [[wiki/pallet-intent\|pallet-intent]]
