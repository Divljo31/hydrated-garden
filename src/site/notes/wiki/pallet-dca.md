---
{"dg-publish":true,"permalink":"/wiki/pallet-dca/","title":"pallet-dca","tags":["dca","scheduled","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-dca","repo":"hydration-node","paths":["pallets/dca/src/lib.rs","pallets/dca/src/types.rs","pallets/dca/src/weights.rs"],"symbols":["Pallet","Config","Schedule","Order","ScheduleIdsPerBlock","ScheduleExecutionBlock","ScheduleExtraGas","ScheduleOwnership","Schedules","RemainingAmounts","RetriesOnError","schedule","terminate","RandomnessProvider","NoLongerSupported"],"traits_impl":[],"depends_on":["pallet-route-executor","pallet-broadcast"],"runtime_index":66,"tags":["dca","scheduled","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
---


# pallet-dca

**TL;DR:** Scheduled periodic trades ("dollar-cost averaging") with price-bounds safety. Users lock a bond; on-chain block hook triggers trades through [[wiki/pallet-route-executor\|pallet-route-executor]] at a cadence. Runtime index = 66.

## Role

Implements [[wiki/dca\|dca]]. Lets users automate recurring sells (e.g. "sell 10 HDX every hour for 100 hours, fail if price deviates >5% from oracle"). Block space is allocated fairly via per-block slot buckets.

> **`Order::Buy` schedules can no longer be created.** `schedule` rejects them with `Error::NoLongerSupported`. Buy schedules already in storage keep executing until they complete or are terminated, so the whole buy execution path in `on_initialize` is still live code.

## Config trait (excerpt)

```rust
// pallets/dca/src/lib.rs
pub trait Config: frame_system::Config + pallet_broadcast::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type RandomnessProvider: RandomProvider;
    type Currencies: MultiCurrencyExtended<Self::AccountId, CurrencyId = AssetId, Balance = Balance>
        + MultiReservableCurrency<Self::AccountId>;
    type RelayChainBlockHashProvider: RelayChainBlockHashProvider;
    type NativeAssetId: Get<AssetId>;
    type FeeReceiver: Get<Self::AccountId>;
    type NamedReserveId: Get<NamedReserveIdentifier>;
    type WeightToFee: WeightToFee<Balance = Balance>;
    type RouteExecutor: TradeExecution<Self::RuntimeOrigin, Self::AccountId, AssetId, Balance>;
    type RouteProvider: RouteProvider<AssetId>;
    type OraclePriceProvider: PriceOracle<AssetId, Price = EmaPrice>;
    type SpotPriceProvider: SpotPriceProvider<AssetId, Price = FixedU128>;
    type MaxPriceDifferenceBetweenBlocks: Get<Permill>;
    type MaxConfigurablePriceDifferenceBetweenBlocks: Get<Permill>;
    type MinimalPeriod: Get<BlockNumberFor<Self>>;
    type BumpChance: Get<Percent>;
    type MaxSchedulePerBlock: Get<u32>;
    type MaxNumberOfRetriesOnError: Get<u8>;
    #[pallet::constant] type MinBudgetInNativeCurrency: Get<Balance>;
    #[pallet::constant] type MinimumTradingLimit: Get<Balance>;
    #[pallet::constant] type SlippageLimitPercentage: Get<Permill>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Schedules` | StorageMap | `ScheduleId → Schedule` |
| `ScheduleOwnership` | StorageMap | `(AccountId, ScheduleId) → ()` |
| `RemainingAmounts` | StorageMap | `ScheduleId → Balance` |
| `RetriesOnError` | StorageMap | `ScheduleId → u8` |
| `ScheduleIdsPerBlock` | StorageMap | `BlockNumber → BoundedVec<ScheduleId, MaxSchedulePerBlock>` |
| `ScheduleExecutionBlock` | StorageMap | `ScheduleId → BlockNumber` |
| `ScheduleExtraGas` | StorageMap | `ScheduleId → u64` |
| `ScheduleIdSequencer` | StorageValue | `ScheduleId` |

There is no `Bonds` storage map — the schedule's budget is held via `Currencies::reserve_named(NamedReserveId, ...)` on the owner's account and released with `unreserve_named` on termination/completion.

## Events

`Scheduled`, `ExecutionPlanned`, `TradeExecuted`, `ExecutionFailed`, `Terminated`, `Completed`, `RandomnessGenerationFailed`.

## Errors

`ScheduleNotFound`, `InvalidState`, `NotScheduleOwner`, `BlockNumberIsNotInFuture`, `PriceUnstable`, `InvalidPeriod`, `SlippageLimitReached`, `BudgetTooLow`, `NoFreeBlockFound`, `MinTradeAmountNotReached`, `NoParentHashFound`, `CalculatingPriceError`, `TotalAmountIsSmallerThanMinBudget`, `MaxNumberOfRetriesReached`, `NoFreeExecutionSlotsAvailable`, `HasActiveSchedules`, `NoReservesLocked`, `NoLongerSupported` (buy orders can no longer be scheduled).

## Extrinsics

| Name | Description |
|------|-------------|
| `schedule` | Create DCA schedule with order, period, slippage, stability threshold, max retries |
| `terminate` | Cancel schedule, return remaining bond + budget |

## Hooks

`on_initialize` executes scheduled trades for the current block (consumes from `ScheduleIdsPerBlock[block]`). Each execution routes through `RouteExecutor::execute_sell/buy`.

## Integration

- **Traits implemented:** none external
- **Traits consumed:** `TradeExecution`, `RouteProvider`, `PriceOracle`, `SpotPriceProvider`, `RandomnessProvider`, `RelayChainBlockHashProvider`
- **Pallets depended on:** [[wiki/pallet-route-executor\|pallet-route-executor]], [[wiki/pallet-broadcast\|pallet-broadcast]], [[wiki/pallet-ema-oracle\|pallet-ema-oracle]]

## Key extrinsic: schedule

```rust
// pallets/dca/src/lib.rs
pub fn schedule(
    origin: OriginFor<T>,
    schedule: Schedule<T::AccountId, AssetId, BlockNumberFor<T>>,
    start_execution_block: Option<BlockNumberFor<T>>,
) -> DispatchResult {
    let who = ensure_signed(origin)?;
    ensure!(who == schedule.owner, Error::<T>::Forbidden);
    // Buy execution stays live for schedules stored before this restriction, so the
    // benchmarks that measure it still need to create one.
    #[cfg(not(feature = "runtime-benchmarks"))]
    ensure!(matches!(schedule.order, Order::Sell { .. }), Error::<T>::NoLongerSupported);
    ensure!(schedule.total_amount >= T::MinBudgetInNativeCurrency::get(), Error::<T>::BudgetTooLow);
    ensure!(schedule.period >= T::MinimalPeriod::get(), Error::<T>::InvalidPeriod);
    let id = Self::next_schedule_id()?;
    Self::reserve_bond(&who, id)?;
    Self::plan_execution(id, start_block)?;
    // ...
}
```

## Gotchas

- Bond size = `WeightToFee(execution_weight) * MaxRetries` — refunded when schedule completes or terminates voluntarily.
- Randomness bumps execution to later block with probability `BumpChance` to prevent MEV slot-squatting.
- `MaxSchedulePerBlock` caps executions per block; when a slot is full, planning jumps to the next available slot.
- Price-stability check: if spot deviates > `stability_threshold` from oracle, trade skips without consuming retry budget.
- Slippage check: actual execution against user's `min_out` / `max_in` is still enforced by route-executor.
- After `MaxNumberOfRetriesOnError` retries, schedule auto-terminates and remaining bond is slashed to the fee receiver.
- `schedule`'s weight annotation no longer adds `AmmTradeWeights::calculate_buy_trade_amounts_weight(...)` — it is plain `WeightInfo::schedule()` now that only sell orders can be created.
- **Retriable errors are a runtime-level list**, not pallet state: `RetryOnErrorForDca` in `runtime/hydradx/src/assets.rs`. It gained `pallet_route_executor::Error::TradingLimitReached` — erc20/aToken rounding can undershoot the dry-run output that DCA passes as the router's min limit, and that is the same class as DCA's own retriable `TradeLimitReached`. Existing members: omnipool `InsufficientBalance`, `pallet_dispatcher::EvmOutOfGas`, circuit-breaker `DepositLimitExceededForWhitelistedAccount`.
- `pallets/dca/src/tests/storage_injection_fidelity.rs` is the reference for how test fixtures inject schedule storage — check it before hand-rolling DCA state in a test.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/dca\|dca]]
