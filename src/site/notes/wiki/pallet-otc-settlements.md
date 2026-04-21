---
{"dg-publish":true,"permalink":"/wiki/pallet-otc-settlements/","title":"pallet-otc-settlements","tags":["otc","settlement","arbitrage","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-otc-settlements","repo":"hydration-node","paths":["pallets/otc-settlements/src/lib.rs","pallets/otc-settlements/src/types.rs"],"symbols":["Pallet","Config","settle_otc_order","ExistentialDepositMultiplier","NamedReserveId"],"traits_impl":[],"depends_on":["pallet-otc","pallet-route-executor"],"runtime_index":72,"tags":["otc","settlement","arbitrage","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-otc-settlements

**TL;DR:** Arbitrage worker for [[wiki/pallet-otc\|pallet-otc]] orders. Any user can call `settle_otc_order`, which fills an OTC order and immediately reverse-trades through [[wiki/pallet-route-executor\|pallet-route-executor]]; if the round-trip nets a profit, the caller pockets it. Runtime index = 72.

## Role

Ensures OTC orders mis-priced relative to AMM venues get closed quickly — caller is rewarded from arbitrage profit; pool state stays consistent.

## Config trait (excerpt)

```rust
// pallets/otc-settlements/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type Currency: MultiCurrency<Self::AccountId, CurrencyId = AssetId, Balance = Balance>;
    type Router: TradeExecution<Self::RuntimeOrigin, Self::AccountId, AssetId, Balance>;
    type ProfitReceiver: Get<Self::AccountId>;
    #[pallet::constant] type MinProfitPercentage: Get<Perquintill>;
    type PriceOracle: PriceOracle<AssetId, Price = EmaPrice>;
    type MinTradingLimit: Get<Balance>;
    #[pallet::constant] type MaxIterations: Get<u32>;
    type WeightInfo: WeightInfo;
}
```

## Storage

None (stateless executor).

## Events

`Executed`.

## Errors

`NotProfitable`, `OrderNotFound`, `TradeAmountTooHigh`, `TradeAmountTooLow`, `MaxIterationsReached`.

## Extrinsics

| Name | Description |
|------|-------------|
| `settle_otc_order` | Fill OTC order + route reverse trade; require min profit |

## Hooks

`offchain_worker` — scans open OTC orders each block, dispatches `settle_otc_order` unsigned extrinsics when a profit opportunity exceeds `MinProfitPercentage`.

## Integration

- **Traits implemented:** none
- **Traits consumed:** `Router` ([[wiki/pallet-route-executor\|pallet-route-executor]]), `PriceOracle`
- **Pallets depended on:** [[wiki/pallet-otc\|pallet-otc]], [[wiki/pallet-route-executor\|pallet-route-executor]], [[wiki/pallet-ema-oracle\|pallet-ema-oracle]]

## Key logic

```rust
// pallets/otc-settlements/src/lib.rs (simplified)
pub fn settle_otc_order(origin: OriginFor<T>, order_id: OrderId, amount: Balance) -> DispatchResult {
    // 1. Find optimal fill amount via binary search over `MaxIterations`.
    let fill = Self::find_optimal_fill(order_id, amount)?;
    // 2. Fill OTC order (consumes pallet_otc::partial_fill_order)
    pallet_otc::Pallet::<T>::partial_fill_order(origin.clone(), order_id, fill)?;
    // 3. Reverse-trade via router in the opposite direction.
    T::Router::execute_sell(/* ... */)?;
    // 4. Ensure net profit >= MinProfitPercentage of input, else revert.
    ensure!(profit_bps >= T::MinProfitPercentage::get(), Error::<T>::NotProfitable);
}
```

## Gotchas

- Binary search converges on profit-maximizing fill size — bounded by `MaxIterations` weight.
- Profit is distributed to `ProfitReceiver` (typically treasury); caller just pays fees.
- Off-chain worker dispatches unsigned extrinsics; runtime requires unsigned-validator to gate spam.
- Requires OTC order's asset pair to have a viable AMM route (otherwise reverse trade fails).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/otc-trading\|otc-trading]]
