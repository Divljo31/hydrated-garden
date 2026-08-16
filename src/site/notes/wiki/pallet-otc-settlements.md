---
{"dg-publish":true,"permalink":"/wiki/pallet-otc-settlements/","title":"pallet-otc-settlements","tags":["otc","settlement","arbitrage","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-otc-settlements","repo":"hydration-node","paths":["pallets/otc-settlements/src/lib.rs","pallets/otc-settlements/src/weights.rs"],"symbols":["Pallet","Config","settle_otc_order","settle_otc","try_find_trade_amount","otc_price","ensure_min_profit","account_id","ExistentialDepositMultiplier","NamedReserveId"],"traits_impl":[],"depends_on":["pallet-otc","pallet-route-executor"],"runtime_index":72,"tags":["otc","settlement","arbitrage","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
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

## Key logic — self-funded fill (no mint/burn)

The arb no longer mints `asset_a` up front and burns it afterwards. It calls [[wiki/pallet-otc\|pallet-otc]]'s `fill_order_with_deferred_delivery`, which hands it the maker's `asset_b` first; the router sell inside the closure buys the `asset_a` needed to complete the fill.

```rust
// pallets/otc-settlements/src/lib.rs
pallet_otc::Pallet::<T>::fill_order_with_deferred_delivery(otc_id, &pallet_acc, amount, |otc_amount_out| {
    T::Router::sell(RawOrigin::Signed(pallet_acc.clone()).into(),
                    asset_b, asset_a, otc_amount_out, 1, route.clone())
        // binary search retries with a smaller amount on any router failure
        .map_err(|_| Error::<T>::TradeAmountTooHigh)?;

    // Must be able to hand `amount` of asset_a back to the maker out of what we bought.
    let bought = Currency::balance(asset_a, &pallet_acc).saturating_sub(asset_a_balance_before);
    ensure!(bought >= amount, Error::<T>::TradeAmountTooHigh);
    Ok(())
})?;

// The shares bought to fill the order were already delivered to the maker, so
// whatever we hold above the initial balance is the arbitrage profit.
let profit = Currency::balance(asset_a, &pallet_acc).checked_sub(asset_a_balance_before)?;
Self::ensure_min_profit(otc.amount_in, profit)?;
```

## Gotchas

- **No more `mint_into` / `burn_from` of `asset_a`.** The old flow minted `amount` of `asset_a` into the pallet account, filled the order, sold `asset_b` back through the router, then burned the minted amount. Profit accounting dropped the `- amount` term accordingly.
- The `partially_fillable && amount != otc.amount_in` branch is gone — `fill_order_with_deferred_delivery` derives full-vs-partial from `amount_in == order.amount_in` itself.
- A short purchase (`bought < amount`) is signalled as `TradeAmountTooHigh` so the binary search retries smaller, rather than reverting the whole settlement.
- Binary search converges on profit-maximizing fill size — bounded by `MaxIterations` weight.
- Profit is distributed to `ProfitReceiver` (typically treasury); caller just pays fees.
- Off-chain worker dispatches unsigned extrinsics; runtime requires unsigned-validator to gate spam.
- Requires OTC order's asset pair to have a viable AMM route (otherwise reverse trade fails).
- Both the OTC fill and the router sell are `#[cfg(not(feature = "runtime-benchmarks"))]`-gated; their weights are accounted separately.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/otc-trading\|otc-trading]]
