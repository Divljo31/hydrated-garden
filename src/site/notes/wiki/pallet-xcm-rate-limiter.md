---
{"dg-publish":true,"permalink":"/wiki/pallet-xcm-rate-limiter/","title":"pallet-xcm-rate-limiter","tags":["xcm","risk","rate-limit","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-xcm-rate-limiter","repo":"hydration-node","paths":["pallets/xcm-rate-limiter/src/lib.rs","pallets/xcm-rate-limiter/src/types.rs"],"symbols":["Pallet","Config","AccumulatedAmounts","DeferredMessages","TryDeferXcm","OnDeposit"],"traits_impl":["OnDeposit","TryDeferXcm"],"depends_on":[],"runtime_index":null,"tags":["xcm","risk","rate-limit","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-xcm-rate-limiter

**TL;DR:** Rate-limits inbound XCM deposits per asset per block window. Deposits exceeding the configured threshold are deferred to a later block rather than executed immediately. Runtime index: not individually listed (wired via adapter in the runtime).

## Role

Defensive layer against sudden bridge / XCM inflow spikes. Used to throttle deposits from [[wiki/asset-hub\|asset-hub]], Polkadot Asset Hub, and other parachains so incoming volume cannot saturate [[wiki/pallet-omnipool\|pallet-omnipool]] trading within a single block.

## Config trait (excerpt)

```rust
// pallets/xcm-rate-limiter/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    type AssetLocation: Parameter + Member + MaxEncodedLen;
    type Balance: Parameter + Member + AtLeast32BitUnsigned + Default + Copy + MaxEncodedLen;
    type CurrencyIdConvert: Convert<Self::AssetLocation, Option<Self::AssetId>>;
    type RateLimitFor: GetByKey<Self::AssetId, Option<Self::Balance>>;
    type DeferDuration: Get<BlockNumberFor<Self>>;
    type MaxDeferDuration: Get<BlockNumberFor<Self>>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `AccumulatedAmounts` | StorageMap | `AssetId → (Balance, BlockNumber)` (cumulative in current window) |
| `DeferredIndices` | StorageMap | `AssetId → BlockNumber` (latest deferred block) |

## Events

None (deferral is internal; XCM layer reports separately).

## Errors

None.

## Extrinsics

None (automatic via adapter hooks).

## Hooks

None (logic runs synchronously via `OnDeposit` trait called by XCM executor).

## Integration

- **Traits implemented:** `OnDeposit`, `TryDeferXcm`
- **Traits consumed:** `Convert<AssetLocation, AssetId>` (for XCM MultiLocation → AssetId resolution), `GetByKey<AssetId, Balance>` (per-asset threshold)
- **Pallets depended on:** [[wiki/pallet-asset-registry\|pallet-asset-registry]] (via CurrencyIdConvert)

## Key trait: TryDeferXcm

```rust
// pallets/xcm-rate-limiter/src/lib.rs
impl<T: Config> TryDeferXcm<T::AssetId, T::Balance, BlockNumberFor<T>> for Pallet<T> {
    fn try_defer(
        asset: T::AssetId,
        amount: T::Balance,
        current_block: BlockNumberFor<T>,
    ) -> Option<BlockNumberFor<T>> {
        let limit = T::RateLimitFor::get(&asset)?;
        let (accumulated, _) = Self::accumulated_for_block(asset, current_block);
        if accumulated.saturating_add(amount) > limit {
            // push to deferred queue
            Some(current_block.saturating_add(T::DeferDuration::get()))
        } else {
            None
        }
    }
}
```

## Gotchas

- Works in tandem with `cumulus_pallet_parachain_system::on_idle` deferral queue — deferred messages get re-executed later.
- `MaxDeferDuration` bounds how long a single message can sit in the deferred queue.
- Only applies to assets that have a configured rate limit — others pass through unchanged.
- Works alongside [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] for defense-in-depth (XCM limiter throttles inbound; circuit-breaker throttles on-chain impact).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/xcm\|xcm]]
