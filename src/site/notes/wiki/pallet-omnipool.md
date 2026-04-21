---
{"dg-publish":true,"permalink":"/wiki/pallet-omnipool/","title":"pallet-omnipool","tags":["amm","omnipool","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-omnipool","repo":"hydration-node","paths":["pallets/omnipool/src/lib.rs","pallets/omnipool/src/types.rs","pallets/omnipool/src/traits.rs","pallets/omnipool/src/router_execution.rs","pallets/omnipool/src/weights.rs"],"symbols":["Pallet","Config","AssetState","Position","Tradability","sell","buy","add_liquidity","remove_liquidity","add_token","set_asset_tradable_state","sacrifice_position","withdraw_protocol_liquidity","HubAssetId","HdxAssetId","Assets","Positions"],"traits_impl":["AMM","AssetPairSpotPrice","TradeExecution","RouteProvider"],"depends_on":["pallet-broadcast","pallet-nft","pallet-ema-oracle","pallet-circuit-breaker","pallet-asset-registry","pallet-dynamic-fees"],"runtime_index":59,"tags":["amm","omnipool","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-omnipool

**TL;DR:** Single-pool AMM where all assets trade against a hub asset (LRNA). LPs receive NFT positions; traders benefit from non-fragmented liquidity with any-to-any swaps. Construct_runtime index = 59.

## Role

Implements the [[omnipool\|omnipool]] core. Every listed asset has an [[wiki/lrna\|lrna]] hub reserve; trades compute deltas through the hub. LP positions are [[wiki/nft-lp-positions\|nft-lp-positions]] valued against the snapshot price at provision.

## Config trait (excerpt)

```rust
// pallets/omnipool/src/lib.rs
pub trait Config: frame_system::Config + pallet_broadcast::Config {
    type AssetId: Member + Parameter + Default + Copy + Ord + HasCompact + ...;
    type Currency: MultiCurrency<Self::AccountId, CurrencyId = Self::AssetId, Balance = Balance>;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type UpdateTradabilityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type AssetRegistry: RegistryInspect<AssetId = Self::AssetId>;
    #[pallet::constant] type HdxAssetId: Get<Self::AssetId>;
    #[pallet::constant] type HubAssetId: Get<Self::AssetId>;
    type Fee: GetDynamicFee<(Self::AssetId, Balance), Fee = (Permill, Permill)>;
    #[pallet::constant] type MinWithdrawalFee: Get<Permill>;
    #[pallet::constant] type MinimumTradingLimit: Get<Balance>;
    #[pallet::constant] type MinimumPoolLiquidity: Get<Balance>;
    #[pallet::constant] type MaxInRatio: Get<u128>;
    #[pallet::constant] type MaxOutRatio: Get<u128>;
    type PositionItemId: Member + Parameter + Default + Copy + HasCompact + AtLeast32BitUnsigned;
    type NFTHandler: Mutate + Create + Inspect;
    type OmnipoolHooks: OmnipoolHooks<...>;
    type PriceBarrier: ShouldAllow<Self::AccountId, Self::AssetId, EmaPrice>;
    type ExternalPriceOracle: ExternalPriceProvider<Self::AssetId, EmaPrice>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Assets` | StorageMap | `AssetId → AssetState` |
| `Positions` | StorageMap | `PositionItemId → Position` |
| `NextPositionId` | StorageValue | `PositionItemId` |
| `HubAssetTradability` | StorageValue | `Tradability` |
| `SlipFee` | StorageValue | `SlipFeeConfig` |
| `SlipFeeHubReserveAtBlockStart` | StorageMap | `AssetId → Balance` |
| `SlipFeeDelta` | StorageMap | `AssetId → SignedBalance` |

## Events

`TokenAdded`, `TokenRemoved`, `LiquidityAdded`, `LiquidityRemoved`, `ProtocolLiquidityRemoved`, `SellExecuted`, `BuyExecuted`, `PositionCreated`, `PositionDestroyed`, `PositionUpdated`, `TradableStateUpdated`, `AssetRefunded`, `AssetWeightCapUpdated`, `SlipFeeSet`.

## Errors (selected)

`InsufficientBalance`, `AssetAlreadyAdded`, `AssetNotFound`, `MissingBalance`, `InvalidInitialAssetPrice`, `BuyLimitNotReached`, `SellLimitExceeded`, `PositionNotFound`, `InsufficientShares`, `NotAllowed`, `AssetWeightCapExceeded`, `AssetNotRegistered`, `InsufficientLiquidity`, `InsufficientTradingAmount`, `SameAssetTradeNotAllowed`, `HubAssetUpdateError`, `InvalidSharesAmount`, `InvalidHubAssetTradableState`, `MaxOutRatioExceeded`, `MaxInRatioExceeded`, `PriceDifferenceTooHigh`, `InvalidOraclePrice`, `InvalidWithdrawalFee`, `SharesRemaining`, `SlippageLimit`, `ProtocolFeeNotConsumed`.

## Extrinsics

| Name | Description |
|------|-------------|
| `add_token` | Add asset with initial liquidity + weight cap (authority) |
| `add_liquidity` | Add liquidity, mint NFT position |
| `add_liquidity_with_limit` | Add with min/max share limits |
| `add_all_liquidity` | Add user's entire balance as liquidity |
| `remove_liquidity` | Remove partial liquidity from position |
| `remove_liquidity_with_limit` | Remove with min/max amount limits |
| `remove_all_liquidity` | Close position entirely |
| `sell` | Sell asset_in for asset_out with `min_buy_amount` guard |
| `buy` | Buy asset_out with `max_sell_amount` guard |
| `set_asset_tradable_state` | Update SELL/BUY/ADD_LIQUIDITY/REMOVE_LIQUIDITY flags |
| `refund_refused_asset` | Refund initial liquidity if token rejected |
| `sacrifice_position` | Destroy position; shares become protocol-owned |
| `withdraw_protocol_liquidity` | Governance withdraws sacrificed liquidity |

## Hooks

Dispatches `OmnipoolHooks::on_liquidity_changed`, `on_trade`, `on_trade_fee` on every liquidity / trade action. No lifecycle hooks (`on_initialize`, etc.) beyond standard pallet machinery.

## Integration

- **Traits implemented:** partial `AMM`, `AssetPairSpotPrice`, `TradeExecution` (consumed by route-executor), `RouteProvider` via storage
- **Traits consumed:** `MultiCurrency`, `RegistryInspect`, `GetDynamicFee`, `OmnipoolHooks`, `ShouldAllow`, `ExternalPriceProvider`, `NFTHandler`
- **Pallets depended on:** [[wiki/pallet-broadcast\|pallet-broadcast]] (Swapped events), [[wiki/pallet-nft\|pallet-nft]] (position storage), [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] (via PriceBarrier + fee computation), [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] (limits), [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]] (Fee)

## Key extrinsic: `sell`

```rust
// pallets/omnipool/src/lib.rs
pub fn sell(
    origin: OriginFor<T>,
    asset_in: T::AssetId,
    asset_out: T::AssetId,
    amount: Balance,
    min_buy_amount: Balance,
) -> DispatchResult {
    let who = ensure_signed(origin.clone())?;
    ensure!(asset_in != asset_out, Error::<T>::SameAssetTradeNotAllowed);
    ensure!(amount >= T::MinimumTradingLimit::get(), Error::<T>::InsufficientTradingAmount);
    ensure!(
        T::Currency::ensure_can_withdraw(asset_in, &who, amount).is_ok(),
        Error::<T>::InsufficientBalance
    );
    // Special case: Hub asset can only be sold, never bought
    if asset_in == T::HubAssetId::get() {
        return Self::sell_hub_asset(origin, &who, asset_out, amount, min_buy_amount);
    }
    // ... general two-leg trade through hub reserve
}
```

## Gotchas

- Hub asset ([[wiki/lrna\|lrna]]) is special: can only be *sold*, never *bought*; all trades compute hub-reserve deltas internally.
- NFT position captures the exact price at provision; [[wiki/impermanent-loss\|impermanent-loss]] is calculated against that snapshot.
- Withdrawal fee is dynamic, derived from oracle; can differ significantly from spot due to [[wiki/price-barrier\|price-barrier]].
- `SlipFee` storage cleared per-block, lazily populated on first trade, tracks net hub-reserve delta.
- `sacrifice_position` → `withdraw_protocol_liquidity` flow is the [[wiki/protocol-owned-liquidity\|protocol-owned-liquidity]] mechanism.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[omnipool\|omnipool]]
