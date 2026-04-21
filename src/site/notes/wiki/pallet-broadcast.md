---
{"dg-publish":true,"permalink":"/wiki/pallet-broadcast/","title":"pallet-broadcast","tags":["broadcast","events","indexer","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-broadcast","repo":"hydration-node","paths":["pallets/broadcast/src/lib.rs","pallets/broadcast/src/types.rs"],"symbols":["Pallet","Config","ExecutionContext","IncrementalIdProvider","Swapped3","Asset","Fee","Filler","TradeOperation"],"traits_impl":["IncrementalIdProvider","ExecutionTypeStack"],"depends_on":[],"runtime_index":204,"tags":["broadcast","events","indexer","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-broadcast

**TL;DR:** Pure event-emitter pallet. Provides an `ExecutionContext` stack (max 16 deep) so trading pallets can tag emitted `Swapped3` events with the outer operation context (DCA, Router, OTC, etc.). No extrinsics, no meaningful storage beyond the stack. Runtime index = 204.

## Role

Standardize trade-event emission across all AMM/DCA/OTC/HSM pallets so off-chain indexers can reconstruct full trade graphs. Instead of each pallet emitting its own flavor of "Swapped" event, they all push context onto this pallet and call `deposit_trade_event` → uniform `Swapped3` event with filler, fees, context breadcrumbs, and a monotonic incremental id.

## Config trait (excerpt)

```rust
// pallets/broadcast/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `ExecutionContextStack` | StorageValue | `BoundedVec<ExecutionType, ConstU32<MAX_STACK_SIZE=16>>` |
| `IncrementalId` | StorageValue | `u32` (monotonic counter) |

## Events

`Swapped3` — unified trade event with fields: `swapper, filler, filler_type, operation (Buy/Sell/ExactIn/ExactOut), inputs: Vec<Asset>, outputs: Vec<Asset>, fees: Vec<Fee>, operation_stack: Vec<ExecutionType>`.

## Errors

None.

## Extrinsics

None — this pallet exposes no callables.

## Hooks

None.

## Integration

- **Traits implemented:** `IncrementalIdProvider<u32>` (monotonic id), `ExecutionTypeStack` (push/pop/get context)
- **Traits consumed:** none
- **Consumers:** [[wiki/pallet-omnipool\|pallet-omnipool]], [[wiki/pallet-stableswap\|pallet-stableswap]], [[wiki/pallet-xyk\|pallet-xyk]], [[wiki/pallet-lbp\|pallet-lbp]], [[wiki/pallet-otc\|pallet-otc]], [[wiki/pallet-route-executor\|pallet-route-executor]], [[wiki/pallet-hsm\|pallet-hsm]], [[wiki/pallet-dca\|pallet-dca]] — each calls `push_execution_type` / `pop_execution_type` around inner trades and `deposit_trade_event` to emit

## Key types

```rust
// pallets/broadcast/src/types.rs
pub enum ExecutionType {
    Omnipool(u32), Stableswap(u32), XYK(u32), LBP(u32), OTC(u32),
    Router(u32), DCA(u32), HSM(u32), Batch(u32), Solver(u32), ICE(u32),
}

pub enum TradeOperation { ExactIn, ExactOut }

pub enum Filler {
    Omnipool, Stableswap(poolId), XYK(shareAssetId), LBP,
    OTC(orderId), HSM(collateralId),
}

pub struct Asset<AssetId, Balance> { pub asset_id: AssetId, pub amount: Balance }
pub enum Fee { Asset { asset_id, amount, recipient }, Protocol {..}, ... }
```

## Key entry point

```rust
// pallets/broadcast/src/lib.rs
pub fn deposit_trade_event(
    swapper: AccountId, filler: AccountId, filler_type: Filler,
    operation: TradeOperation, inputs: Vec<Asset<AssetId, Balance>>,
    outputs: Vec<Asset<AssetId, Balance>>, fees: Vec<Fee<...>>,
) {
    let stack = ExecutionContextStack::<T>::get().into_inner();
    Self::deposit_event(Event::Swapped3 {
        swapper, filler, filler_type, operation,
        inputs, outputs, fees, operation_stack: stack,
    });
}
```

## Gotchas

- `MAX_STACK_SIZE = 16` — nested routing/DCA/ICE operations beyond 16 frames will panic in debug or silently drop the push in release (push returns `Result`, callers must handle).
- `ExecutionContextStack` is cleared at end of block implicitly by being per-extrinsic (each extrinsic starts with empty stack assumed); pallets must pair every `push` with a `pop`.
- `IncrementalId` is persistent across blocks (monotonic forever).
- The older `Swapped` / `Swapped2` events are still emitted by some pallets for backward compatibility — indexers should prefer `Swapped3`.
- No fee is charged; no benchmarking needed (no extrinsics).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
