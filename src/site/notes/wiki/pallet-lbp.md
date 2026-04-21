---
{"dg-publish":true,"permalink":"/wiki/pallet-lbp/","title":"pallet-lbp","tags":["amm","lbp","liquidity-bootstrapping","launchpad","runtime","rust"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-lbp","repo":"hydration-node","paths":["pallets/lbp/src/lib.rs","pallets/lbp/src/types.rs","pallets/lbp/src/trade_execution.rs","pallets/lbp/src/provider.rs","pallets/lbp/src/weights.rs","pallets/lbp/src/invariants.rs"],"symbols":["Pallet","Config","Pool","LBPWeight","WeightCurveType","create_pool","update_pool_data","add_liquidity","remove_liquidity","sell","buy","start_sale","stop_sale","LBPWeightCalculation"],"traits_impl":["LBPWeightCalculation","TradeExecution"],"depends_on":["pallet-broadcast"],"runtime_index":73,"tags":["amm","lbp","liquidity-bootstrapping","launchpad","runtime","rust"],"last_updated":"2026-04-13"}}
---


# pallet-lbp

**TL;DR:** Liquidity Bootstrapping Pool with time-weighted price curve. Weights shift linearly from initial to final over the sale window. Pairs trade through a dynamic Balancer-style invariant with repayment-target fee phases. Runtime index = 73.

## Role

Implements [[wiki/lbp\|lbp]] — used historically for HDX fair-launch and for external token launches (e.g. the HYDRADX→HDX swap, project launchpads). Ownership-based: pool owner controls lifecycle.

## Config trait (excerpt)

```rust
// pallets/lbp/src/lib.rs
pub trait Config: frame_system::Config + pallet_broadcast::Config {
    type MultiCurrency: MultiCurrencyExtended<Self::AccountId, CurrencyId = AssetId>
        + MultiLockableCurrency<Self::AccountId>;
    type LockedBalance: LockedBalance<AssetId, Self::AccountId, Balance>;
    type CreatePoolOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type LBPWeightFunction: LBPWeightCalculation<BlockNumberFor<Self>>;
    type AssetPairAccountId: AssetPairAccountIdFor<AssetId, PoolId<Self>>;
    type WeightInfo: WeightInfo;
    #[pallet::constant] type MinTradingLimit: Get<Balance>;
    #[pallet::constant] type MinPoolLiquidity: Get<Balance>;
    #[pallet::constant] type MaxInRatio: Get<u128>;
    #[pallet::constant] type MaxOutRatio: Get<u128>;
    type BlockNumberProvider: BlockNumberProvider<BlockNumber = BlockNumberFor<Self>>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Pools` | StorageMap | `PoolId (AccountId) → Pool` |

## Events

`PoolCreated`, `PoolUpdated`, `LiquidityAdded`, `LiquidityRemoved`, `SellExecuted`, `BuyExecuted`, `FeeCollected`, `PoolDestroyed`.

## Errors (selected)

`CannotCreatePoolWithSameAssets`, `NotOwner`, `SaleStarted`, `SaleNotEnded`, `SaleIsNotRunning`, `MaxSaleDurationExceeded`, `InsufficientAssetBalance`, `PoolNotFound`, `PoolAlreadyExists`, `InvalidBlockRange`, `WeightCalculationError`, `InvalidWeight`, `MaxInRatioExceeded`, `MaxOutRatioExceeded`, `FeeCollectorWithAssetAlreadyUsed`.

## Extrinsics

| Name | Description |
|------|-------------|
| `create_pool` | Create pool with owner, asset pair, initial/final weights, fee, repay_target |
| `add_liquidity` | Owner adds liquidity (before sale starts) |
| `remove_liquidity` | Owner removes liquidity (before start or after end) |
| `sell` | Trade with time-weighted price |
| `buy` | Buy asset_out with `max_sell_amount` |
| `update_pool_data` | Owner updates fee_collector, repay_target, weight curve |
| `start_sale` | Initiate sale (sets start/end blocks) |
| `stop_sale` | Terminate sale early (owner) |

## Hooks

`on_initialize` runs integrity test only; no production mutations.

## Integration

- **Traits implemented:** `LBPWeightCalculation`, `TradeExecution`, pool-manager
- **Traits consumed:** `MultiCurrencyExtended`, `MultiLockableCurrency`, `LockedBalance`, `BlockNumberProvider`, `AssetPairAccountIdFor`
- **Pallets depended on:** [[wiki/pallet-broadcast\|pallet-broadcast]]

## Key extrinsic: `create_pool`

```rust
// pallets/lbp/src/lib.rs
pub fn create_pool(
    origin: OriginFor<T>,
    asset_a: AssetId,
    asset_b: AssetId,
    initial_weight: LBPWeight,
    final_weight: LBPWeight,
    weight_curve: WeightCurveType,
    fee: (u32, u32),
    fee_collector: T::AccountId,
    repay_target: Balance,
) -> DispatchResult {
    let owner = ensure_signed(origin)?;
    ensure!(asset_a != asset_b, Error::<T>::CannotCreatePoolWithSameAssets);
    ensure!(
        initial_weight <= MAX_WEIGHT && final_weight <= MAX_WEIGHT,
        Error::<T>::InvalidWeight
    );
    // ... build Pool and insert into Pools
}
```

## Gotchas

- `MAX_SALE_DURATION ≈ 14 days` (201,600 blocks at 6s/block).
- Weights in range 0..=100_000_000 (100%); linear interpolation between start & end blocks.
- Fee structure has a *repayment target*: until `fee_collector` receives the target amount of asset_a, fee is elevated (≈ 20%); drops to configured fee afterwards.
- Pool owner cannot modify weights once the sale has started; only `fee_collector` / `repay_target` via `update_pool_data`.
- `MaxInRatio` / `MaxOutRatio` apply; mitigates flash-loan-style price manipulation.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/lbp\|lbp]]
