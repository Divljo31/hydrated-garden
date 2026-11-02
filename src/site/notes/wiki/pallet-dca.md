---
{"dg-publish":true,"permalink":"/wiki/pallet-dca/","title":"pallet-dca","tags":["dca","scheduled","ice","migration","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-dca","repo":"hydration-node","paths":["pallets/dca/src/lib.rs","pallets/dca/src/types.rs","pallets/dca/src/weights.rs","runtime/hydradx/src/assets.rs"],"symbols":["Pallet","Config","Schedule","Order","ScheduleIdsPerBlock","ScheduleExecutionBlock","ScheduleExtraGas","ScheduleOwnership","Schedules","RemainingAmounts","RetriesOnError","MigrationEnabled","schedule","terminate","unlock_reserves","set_migration_enabled","force_cancel_schedules","migrate_or_cancel","convert_schedule","cancel_for_migration","CancelReason","IntentMigrator","AccountFeeCurrency","RetryOnErrorForDca","NoLongerSupported","MigrationInProgress"],"traits_impl":[],"depends_on":["pallet-route-executor","pallet-broadcast","pallet-intent","pallet-transaction-multi-payment"],"runtime_index":66,"tags":["dca","scheduled","ice","migration","runtime","rust","substrate"],"last_updated":"2026-11-02"}}
---


# pallet-dca

**TL;DR:** Legacy scheduled periodic trades ("dollar-cost averaging"), executed in `on_initialize` through [[wiki/pallet-route-executor\|pallet-route-executor]]. **Being replaced by DCA intents in [[wiki/pallet-intent\|pallet-intent]] / [[wiki/pallet-ice\|pallet-ice]]**: when governance sets `MigrationEnabled`, each schedule that comes due is converted into an ICE DCA intent (or cancelled and refunded) instead of executing, and new schedules are refused. Only `Order::Sell` can be created. Runtime index = 66.

## Role

Implements the classic [[wiki/dca\|dca]] flow: owner reserves a budget, the pallet plans executions at `period` cadence into per-block buckets, and each due schedule trades via the router with oracle-based stability and slippage checks. New DCA work should target ICE DCA intents ([[wiki/pallet-intent\|pallet-intent]]); this pallet is the migration source.

## Config trait (runtime-relevant items)

```rust
// pallets/dca/src/lib.rs
type AssetId: Parameter + Member + Copy + MaybeSerializeDeserialize + MaxEncodedLen + Into<ice_support::AssetId>;
type IntentMigrator: IntentMigrator<Self::AccountId>;          // runtime: Intent
type TerminateOrigin: EnsureOrigin<Self::RuntimeOrigin>;       // runtime: Root | TC majority | GeneralAdmin
type AccountFeeCurrency: AccountFeeCurrency<Self::AccountId, AssetId = Self::AssetId>;
type RetryOnError: Contains<DispatchError>;                    // runtime: RetryOnErrorForDca
type MinimalPeriod: Get<u32>;                                  // runtime: 15 blocks (= 30 s at 2 s blocks)
type MaxSchedulePerBlock: Get<u32>;                            // runtime: 6
// + RouteExecutor, RouteProvider, OraclePriceProvider, NativePriceOracle, ExtraGasSupport,
//   GasWeightMapping, MaxPriceDifferenceBetweenBlocks, MaxConfigurablePriceDifferenceBetweenBlocks,
//   MaxNumberOfRetriesOnError, BumpChance, MinimumTradingLimit, NativeAssetId,
//   MinBudgetInNativeCurrency, FeeReceiver, NamedReserveId, WeightToFee, AmmTradeWeights, WeightInfo
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `ScheduleIdSequencer` | StorageValue | `ScheduleId` |
| `Schedules` | StorageMap | `ScheduleId → Schedule` |
| `ScheduleOwnership` | StorageDoubleMap | `(AccountId, ScheduleId) → ()` |
| `RemainingAmounts` | StorageMap | `ScheduleId → Balance` |
| `RetriesOnError` | StorageMap | `ScheduleId → u8` |
| `ScheduleExecutionBlock` | StorageMap | `ScheduleId → BlockNumber` |
| `ScheduleIdsPerBlock` | StorageMap | `BlockNumber → BoundedVec<ScheduleId, MaxSchedulePerBlock>` |
| `ScheduleExtraGas` | StorageMap | `ScheduleId → u64` |
| `MigrationEnabled` | StorageValue | `bool` — when true, due schedules migrate instead of execute |

Budget is held with `reserve_named(NamedReserveId, …)` on the owner, not in a pallet account.

## Extrinsics

| # | Name | Origin | Description |
|---|------|--------|-------------|
| 0 | `schedule` | Signed (owner) | Create a **sell** schedule. Rejected with `NoLongerSupported` for `Order::Buy`, `MigrationInProgress` while migration is on |
| 1 | `terminate` | Owner, or `TerminateOrigin` | Cancel; unreserve remaining budget |
| 2 | `unlock_reserves(who, asset_id)` | Signed | Release stale named reserves when the account has no active schedules |
| 3 | `set_migration_enabled(bool)` | `TerminateOrigin` | Turn the DCA → ICE migration on/off |
| 4 | `force_cancel_schedules(ids)` | `TerminateOrigin` | Cancel + refund listed schedules (`CancelReason::ForceCancelled`) |

## Events / errors

Events: `ExecutionStarted`, `Scheduled`, `ExecutionPlanned`, `TradeExecuted`, `TradeFailed`, `Terminated`, `Completed`, `RandomnessGenerationFailed`, `ReserveUnlocked`, `Migrated { id, who, intent_id }`, `MigrationCancelled { id, who, asset, refunded, reason }`, `MigrationEnabledSet { enabled }`.

Errors: `ScheduleNotFound`, `MinTradeAmountNotReached`, `Forbidden`, `BlockNumberIsNotInFuture`, `PriceUnstable`, `Bumped`, `CalculatingPriceError`, `TotalAmountIsSmallerThanMinBudget`, `BudgetTooLow`, `NoFreeBlockFound`, `ManuallyTerminated`, `MaxRetryReached`, `TradeLimitReached`, `SlippageLimitReached`, `NoParentHashFound`, `InvalidState`, `PeriodTooShort`, `StabilityThresholdTooHigh`, `HasActiveSchedules`, `NoReservesLocked`, `NoLongerSupported`, `MigrationInProgress`.

## Migration to ICE intents

In `on_initialize`, if `MigrationEnabled`, every schedule due in the block goes through `migrate_or_cancel` instead of trading:

1. `Order::Buy` → cancel (`CancelReason::BuyOrder` — intents are sell-only).
2. Fixed-budget schedule whose `RemainingAmounts < amount_in` → cancel (`BudgetBelowTrade`).
3. Otherwise build `DcaParams { asset_in, asset_out, amount_in, amount_out: min_amount_out, slippage: schedule.slippage or MaxPriceDifferenceBetweenBlocks, budget (None for rolling schedules), period }` and `convert_schedule`: unreserve the DCA reserve (non-zero remainder → `UnreserveRemainder`), delete the schedule, call `IntentMigrator::add_migrated_intent` (= `pallet_intent::do_add_migrated_intent`, which takes its own reserve). Success → `Migrated`; intent rejection → `IntentCreationFailed(err)` cancel.

Every cancel path refunds via `try_unreserve_all` and emits `MigrationCancelled`. `CancelReason = BuyOrder | BudgetBelowTrade | UnreserveRemainder | IntentCreationFailed(DispatchError) | ForceCancelled`. Migration is off by default.

## Fees

Execution fees are charged in `asset_in` only if it is a payment currency (`AccountFeeCurrency::is_payment_currency`, i.e. in [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]] `AcceptedCurrencies` or HDX); otherwise the fee is taken from the owner's fee currency by a separate transfer, and the min-budget check is skipped for that asset (`budget_transaction_fee` returns zero).

## Gotchas

- Randomness bumps execution to a later block with probability `BumpChance` to prevent slot-squatting; full buckets push planning to the next free block.
- Price-stability check: spot vs oracle deviation beyond the threshold skips the trade without consuming retries.
- **Retriable errors are a runtime list**, `RetryOnErrorForDca` (`runtime/hydradx/src/assets.rs`): omnipool `AssetNotFound` / `NotAllowed` / `InsufficientBalance` (aToken rounding), router `TradingLimitReached`, dispatcher `EvmOutOfGas` / `UniswapV3InsufficientLiquidity`, circuit-breaker `DepositLimitExceededForWhitelistedAccount`.
- Periods are in blocks; with 2 s blocks `MinimalPeriod = 15` is 30 s.
- Existing `Order::Buy` schedules still execute (when migration is off) — the buy path in `on_initialize` is live code.
- `pallets/dca/src/tests/storage_injection_fidelity.rs` is the reference for injecting schedule storage in tests.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/dca\|dca]], [[wiki/pallet-intent\|pallet-intent]], [[wiki/pallet-ice\|pallet-ice]]
