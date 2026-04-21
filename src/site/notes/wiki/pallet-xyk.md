---
{"dg-publish":true,"permalink":"/wiki/pallet-xyk/","title":"pallet-xyk","tags":["amm","xyk","constant-product","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-xyk","repo":"hydration-node","paths":["pallets/xyk/src/lib.rs","pallets/xyk/src/types.rs","pallets/xyk/src/trade_execution.rs","pallets/xyk/src/impls.rs","pallets/xyk/src/weights.rs"],"symbols":["Pallet","Config","AssetPair","create_pool","add_liquidity","remove_liquidity","sell","buy","ShareToken","TotalLiquidity","PoolAssets"],"traits_impl":["OnCreatePoolHandler","OnTradeHandler","OnLiquidityChangedHandler","TradeExecution"],"depends_on":["pallet-broadcast","pallet-asset-registry"],"runtime_index":74,"tags":["amm","xyk","constant-product","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-xyk

**TL;DR:** Classic constant-product (x·y = k) AMM with two-asset pools and ERC20-style share tokens. Simple and gas-efficient; used as long tail listing venue. Runtime index = 74.

## Role

Implements [[wiki/xyk-pools\|xyk-pools]]. Intended for long-tail / low-cap asset pairs that don't belong in the [[omnipool\|omnipool]]. Listed by governance (or permissioned `CanCreatePool`).

## Config trait (excerpt)

```rust
// pallets/xyk/src/lib.rs
pub trait Config: frame_system::Config + pallet_broadcast::Config {
    type AssetRegistry: Create<Balance, AssetId = AssetId, Error = DispatchError>;
    type AssetPairAccountId: AssetPairAccountIdFor<AssetId, Self::AccountId>;
    type Currency: MultiCurrencyExtended<Self::AccountId, CurrencyId = AssetId, Balance = Balance>;
    #[pallet::constant] type NativeAssetId: Get<AssetId>;
    type WeightInfo: WeightInfo;
    #[pallet::constant] type GetExchangeFee: Get<(u32, u32)>;
    #[pallet::constant] type MinTradingLimit: Get<Balance>;
    #[pallet::constant] type MinPoolLiquidity: Get<Balance>;
    #[pallet::constant] type MaxInRatio: Get<u128>;
    #[pallet::constant] type MaxOutRatio: Get<u128>;
    #[pallet::constant] type OracleSource: Get<Source>;
    type CanCreatePool: CanCreatePool<AssetId>;
    type AMMHandler: OnCreatePoolHandler + OnTradeHandler + OnLiquidityChangedHandler;
    type NonDustableWhitelistHandler: DustRemovalAccountWhitelist<Self::AccountId>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `ShareToken` | StorageMap | `AccountId (pool account) → AssetId (share token)` |
| `TotalLiquidity` | StorageMap | `AccountId (pool account) → Balance` |
| `PoolAssets` | StorageMap | `AccountId (pool account) → (AssetId, AssetId)` |

## Events

`LiquidityAdded`, `LiquidityRemoved`, `PoolCreated`, `PoolDestroyed`, `SellExecuted`, `BuyExecuted`.

## Errors (selected)

`CannotCreatePoolWithSameAssets`, `InsufficientLiquidity`, `InsufficientTradingAmount`, `ZeroLiquidity`, `InvalidMintedLiquidity`, `InvalidLiquidityAmount`, `AssetAmountExceededLimit`, `AssetAmountNotReachedLimit`, `InsufficientAssetBalance`, `InsufficientPoolAssetBalance`, `InsufficientNativeCurrencyBalance`, `TokenPoolNotFound`, `TokenPoolAlreadyExists`, `SellAssetAmountInvalid`, `BuyAssetAmountInvalid`, `MaxOutRatioExceeded`, `MaxInRatioExceeded`, `CannotCreatePool`, `SlippageLimit`.

## Extrinsics

| Name | Description |
|------|-------------|
| `create_pool` | Create XYK pool with asset pair + initial liquidity |
| `add_liquidity` | Add liquidity for asset_a + min_share guard |
| `add_liquidity_with_limits` | Add with both min/max limits on amount_b |
| `remove_liquidity` | Burn shares, withdraw proportional liquidity |
| `remove_liquidity_with_limits` | Remove with min/max received amounts |
| `sell` | Sell asset_in for asset_out with `max_limit` price cap |
| `buy` | Buy asset_out with `max_limit` on asset_in spent |

## Hooks

None (`Hooks` impl is empty); `AMMHandler` callbacks deliver events to [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] and [[wiki/pallet-xyk-liquidity-mining\|pallet-xyk-liquidity-mining]] but there is no `on_initialize` / `on_finalize`.

## Integration

- **Traits implemented:** `OnCreatePoolHandler`, `OnTradeHandler`, `OnLiquidityChangedHandler`, `AMM` (via `AMMHandler`), `TradeExecution`
- **Traits consumed:** `MultiCurrencyExtended`, `AssetRegistry`, `AssetPairAccountIdFor`, `CanCreatePool`, `DustRemovalAccountWhitelist`
- **Pallets depended on:** [[wiki/pallet-broadcast\|pallet-broadcast]] (Swapped), [[wiki/pallet-asset-registry\|pallet-asset-registry]] (share token creation)

## Key extrinsic: `sell`

```rust
// pallets/xyk/src/lib.rs
pub fn sell(
    origin: OriginFor<T>,
    asset_in: AssetId,
    asset_out: AssetId,
    amount: Balance,
    max_limit: Balance,
    _discount: bool,
) -> DispatchResult {
    let who = ensure_signed(origin)?;
    Self::execute_sell(&Self::validate_sell(
        &who,
        AssetPair { asset_in, asset_out },
        amount,
        max_limit,
    )?)?;
    Ok(())
}
```

## Gotchas

- Pool account is deterministically derived from the *ordered* asset pair via `AssetPairAccountId`; ordering matters for lookups.
- Share token created per pool; minted/burned atomically with liquidity changes.
- Fee is a constant ratio (`GetExchangeFee` tuple); not dynamic per asset (unlike [[wiki/pallet-omnipool\|pallet-omnipool]]).
- `MaxInRatio` / `MaxOutRatio` prevent extreme slippage / flash-loan style manipulation.
- `_discount` parameter is vestigial; kept for API compatibility.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/xyk-pools\|xyk-pools]]
