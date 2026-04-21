---
{"dg-publish":true,"permalink":"/wiki/pallet-stableswap/","title":"pallet-stableswap","tags":["amm","stableswap","curve","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-stableswap","repo":"hydration-node","paths":["pallets/stableswap/src/lib.rs","pallets/stableswap/src/types.rs","pallets/stableswap/src/trade_execution.rs","pallets/stableswap/src/traits.rs","pallets/stableswap/src/weights.rs","pallets/stableswap/src/migrations/"],"symbols":["Pallet","Config","PoolInfo","PoolPegInfo","Tradability","PoolSnapshot","sell","buy","create_pool","create_pool_with_pegs","add_assets_liquidity","remove_assets_liquidity","update_amplification","StableswapHooks","PegRawOracle"],"traits_impl":["TradeExecution"],"depends_on":["pallet-broadcast","pallet-asset-registry","pallet-ema-oracle"],"runtime_index":70,"tags":["amm","stableswap","curve","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-stableswap

**TL;DR:** Curve-style AMM for stablecoins and correlated assets using the StableSwap invariant. Up to 5 assets per pool, configurable amplification, drifting pegs via oracle/MM. Runtime index = 70.

## Role

Implements [[wiki/stableswap\|stableswap]] pools. Backs [[wiki/hollar\|hollar]] (HSM uses stableswap pools as peg-arbitrage venues) and [[wiki/gdot\|gdot]]/[[wiki/geth\|geth]]/[[wiki/gsol\|gsol]] strategy tokens.

## Config trait (excerpt)

```rust
// pallets/stableswap/src/lib.rs
pub trait Config: frame_system::Config + pallet_broadcast::Config {
    type BlockNumberProvider: BlockNumberProvider<BlockNumber = BlockNumberFor<Self>>;
    type AssetId: Member + Parameter + Ord + Default + Copy + HasCompact + ...;
    type Currency: MultiCurrency<Self::AccountId, CurrencyId = Self::AssetId, Balance = Balance>;
    type ShareAccountId: AccountIdFor<Self::AssetId, AccountId = Self::AccountId>;
    type AssetInspection: Inspect<AssetId = Self::AssetId>;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type UpdateTradabilityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type DustAccountHandler: DustRemovalAccountWhitelist<Self::AccountId>;
    type Hooks: StableswapHooks<Self::AssetId>;
    #[pallet::constant] type MinPoolLiquidity: Get<Balance>;
    #[pallet::constant] type MinTradingLimit: Get<Balance>;
    #[pallet::constant] type AmplificationRange: Get<RangeInclusive<NonZeroU16>>;
    type TargetPegOracle: PegRawOracle<Self::AssetId, Balance, BlockNumberFor<Self>>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Pools` | StorageMap | `AssetId (pool_id) → PoolInfo` |
| `PoolPegs` | StorageMap | `AssetId (pool_id) → PoolPegInfo` |
| `AssetTradability` | StorageDoubleMap | `(AssetId, AssetId) → Tradability` |
| `PoolSnapshots` | StorageMap | `AssetId (pool_id) → PoolSnapshot` |
| `BlockFee` | StorageMap | `AssetId (pool_id) → Permill` |

## Events

`PoolCreated`, `FeeUpdated`, `LiquidityAdded`, `LiquidityRemoved`, `SellExecuted`, `BuyExecuted`, `TradableStateUpdated`, `AmplificationChanging`, `PoolDestroyed`, `PoolPegSourceUpdated`, `PoolMaxPegUpdateUpdated`.

## Errors (selected)

`IncorrectAssets`, `MaxAssetsExceeded`, `PoolNotFound`, `PoolExists`, `AssetNotInPool`, `ShareAssetNotRegistered`, `ShareAssetInPoolAssets`, `AssetNotRegistered`, `InvalidAssetAmount`, `InsufficientBalance`, `InsufficientShares`, `InsufficientLiquidity`, `InsufficientLiquidityRemaining`, `InsufficientTradingAmount`, `BuyLimitNotReached`, `SellLimitExceeded`, `InvalidInitialLiquidity`, `InvalidAmplification`, `PastBlock`.

## Extrinsics

| Name | Description |
|------|-------------|
| `create_pool` | Create pool with assets + amplification + fee |
| `create_pool_with_pegs` | Create pool with peg configuration |
| `add_assets_liquidity` | First LP adds all assets; subsequent LPs can add single asset |
| `remove_assets_liquidity` | Burn shares, receive single or all assets |
| `sell` | Sell asset_in → asset_out with `min_buy_amount` |
| `buy` | Buy asset_out with `max_sell_amount` |
| `set_asset_tradable_state` | Per-asset tradability flags |
| `update_amplification` | Schedule amplification change |
| `update_pool_fee` | Governance fee update |
| `destroy_pool` | Governance pool removal |

## Hooks

Calls `StableswapHooks::on_liquidity_changed` and `on_trade` for every liquidity / trade action. No lifecycle hooks.

## Integration

- **Traits implemented:** `TradeExecution` (consumed by route-executor)
- **Traits consumed:** `MultiCurrency`, `AssetInspection`, `BlockNumberProvider`, `PegRawOracle`, `DustRemovalAccountWhitelist`, `StableswapHooks`
- **Pallets depended on:** [[wiki/pallet-broadcast\|pallet-broadcast]], [[wiki/pallet-asset-registry\|pallet-asset-registry]] (decimals), [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] (peg oracle via MMOracle)

## Key extrinsic: `sell`

```rust
// pallets/stableswap/src/lib.rs
pub fn sell(
    origin: OriginFor<T>,
    pool_id: T::AssetId,
    asset_in: T::AssetId,
    asset_out: T::AssetId,
    amount_in: Balance,
    min_buy_amount: Balance,
) -> DispatchResult {
    let who = ensure_signed(origin)?;
    ensure!(asset_in != asset_out, Error::<T>::NotAllowed);
    ensure!(
        Self::is_asset_allowed(pool_id, asset_in, Tradability::SELL)
            && Self::is_asset_allowed(pool_id, asset_out, Tradability::BUY),
        Error::<T>::NotAllowed
    );
    ensure!(amount_in >= T::MinTradingLimit::get(), Error::<T>::InsufficientTradingAmount);
    let pool = Pools::<T>::get(pool_id).ok_or(Error::<T>::PoolNotFound)?;
    let (amount_out, fee_amount) = Self::calculate_out_amount(pool_id, asset_in, asset_out, amount_in, true)?;
    ensure!(amount_out >= min_buy_amount, Error::<T>::BuyLimitNotReached);
    // ... transfer and emit Swapped via pallet_broadcast
}
```

## Gotchas

- `MAX_ASSETS_IN_POOL = 5`; pool creation requires 2+ assets.
- First LP must seed *all* assets; subsequent LPs may add a single asset at the current price (causes small slippage).
- Amplification changes are *scheduled*, applied linearly between start/end blocks; sudden changes rejected.
- Pegs can be `Value` (const), `Oracle` ([[wiki/ema-oracle\|ema-oracle]]), or `MMOracle` ([[hsm\|hsm]]); drifting peg alters the invariant target.
- Pool account derived deterministically from `pool_id` via `ShareAccountId`; share token is a separate fungible asset.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/stableswap\|stableswap]]
- [[wiki/hollar\|hollar]]
