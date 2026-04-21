---
{"dg-publish":true,"permalink":"/wiki/pallet-route-executor/","title":"pallet-route-executor","tags":["router","multi-hop","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-route-executor","repo":"hydration-node","paths":["pallets/route-executor/src/lib.rs","pallets/route-executor/src/types.rs","pallets/route-executor/src/weights.rs"],"symbols":["Pallet","Config","Route","Trade","PoolType","sell","buy","set_route","update_route","Routes","TradeExecution","RouteProvider"],"traits_impl":["RouteProvider"],"depends_on":["pallet-broadcast","pallet-omnipool","pallet-stableswap","pallet-xyk","pallet-lbp","pallet-ema-oracle"],"runtime_index":67,"tags":["router","multi-hop","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-route-executor

**TL;DR:** Multi-hop trade router across [[wiki/pallet-omnipool\|pallet-omnipool]] / [[wiki/pallet-stableswap\|pallet-stableswap]] / [[wiki/pallet-xyk\|pallet-xyk]] / [[wiki/pallet-lbp\|pallet-lbp]] / [[wiki/pallet-hsm\|pallet-hsm]]. Caches on-chain routes per asset pair; executes and validates via oracle prices. Runtime index = 67.

## Role

Entry point for all user trades. [[wiki/smart-order-router\|smart-order-router]] (off-chain) selects best path, then submits `sell`/`buy` via this pallet. Also stores curated on-chain routes used when a user provides an empty route.

## Config trait (excerpt)

```rust
// pallets/route-executor/src/lib.rs
pub trait Config: frame_system::Config + pallet_broadcast::Config {
    type AssetId: Parameter + Member + Copy + ...;
    type Balance: Parameter + Member + Copy + ...;
    #[pallet::constant] type NativeAssetId: Get<Self::AssetId>;
    type Currency: Inspect + Mutate;
    type AMM: TradeExecution<
        Self::RuntimeOrigin, Self::AccountId, Self::AssetId, Self::Balance,
        Error = DispatchError,
    >;
    type OraclePriceProvider: PriceOracle<Self::AssetId, Price = EmaPrice>;
    #[pallet::constant] type OraclePeriod: Get<OraclePeriod>;
    type DefaultRoutePoolType: Get<PoolType<Self::AssetId>>;
    type ForceInsertOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type WeightInfo: AmmTradeWeights<Trade<Self::AssetId>>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Routes` | StorageMap | `AssetPair → Route` |

## Events

`Executed`, `RouteUpdated`.

## Errors

`TradingLimitReached`, `MaxTradesExceeded`, `PoolNotSupported`, `InsufficientBalance`, `RouteCalculationFailed`, `InvalidRoute`, `RouteUpdateIsNotSuccessful`, `RouteHasNoOracle`, `InvalidRouteExecution`, `NotAllowed`.

## Extrinsics

| Name | Description |
|------|-------------|
| `sell` | Execute sell route with `min_amount_out`; falls back to default route if empty |
| `buy` | Execute buy route with `max_amount_in` |
| `set_route` | Set on-chain cached route (validated via dry-run) |
| `update_route` | Update route; oracle-validated and compared to existing |

## Hooks

None.

## Integration

- **Traits implemented:** `RouteProvider` (via `Routes` storage), `TradeExecution` (dispatches to inner AMM)
- **Traits consumed:** `TradeExecution` (inner AMMs), `PriceOracle`, `BroadcastContext`
- **Pallets depended on:** [[wiki/pallet-broadcast\|pallet-broadcast]] (sets `ExecutionType::Router`), any AMM pallet via `AMM` aggregator

## Key extrinsic: `sell`

```rust
// pallets/route-executor/src/lib.rs
pub fn sell(
    origin: OriginFor<T>,
    asset_in: T::AssetId,
    asset_out: T::AssetId,
    amount_in: T::Balance,
    min_amount_out: T::Balance,
    route: Route<T::AssetId>,
) -> DispatchResult {
    Self::do_sell(origin, asset_in, asset_out, amount_in, min_amount_out, route)
}

pub fn buy(
    origin: OriginFor<T>,
    asset_in: T::AssetId,
    asset_out: T::AssetId,
    amount_out: T::Balance,
    max_amount_in: T::Balance,
    route: Route<T::AssetId>,
) -> DispatchResult {
    let who = ensure_signed(origin.clone())?;
    ensure!(asset_in != asset_out, Error::<T>::NotAllowed);
    Self::ensure_route_size(route.len())?;
    // ... calculate intermediate amounts + slippage + execute per hop
}
```

## Gotchas

- `MAX_NUMBER_OF_TRADES = 9`; limits route depth for weight budgeting.
- Empty `route` → falls back to `DefaultRoutePoolType` (Omnipool) using the direct pair.
- Oracle validation requires every intermediate asset to have an oracle price — else `RouteHasNoOracle`.
- Execution context: sets [[wiki/pallet-broadcast\|pallet-broadcast]] `ExecutionType::Router` so downstream AMMs emit `Swapped` events tagged as a single router trade.
- `set_route`/`update_route` dry-runs in both directions and picks the better path.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/smart-order-router\|smart-order-router]]
- [[wiki/sdk-next\|sdk-next]]
