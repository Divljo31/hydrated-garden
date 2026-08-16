---
{"dg-publish":true,"permalink":"/wiki/pallet-otc/","title":"pallet-otc","tags":["otc","orderbook","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-otc","repo":"hydration-node","paths":["pallets/otc/src/lib.rs","pallets/otc/src/weights.rs"],"symbols":["Pallet","Config","Order","NextOrderId","Orders","place_order","fill_order","partial_fill_order","cancel_order","fill_order_with_deferred_delivery","partial_amount_out","ensure_remaining_order_valid","release_reserved_asset_out","deposit_fill_events","calculate_fee","NAMED_RESERVE_ID"],"traits_impl":[],"depends_on":["pallet-broadcast"],"runtime_index":64,"tags":["otc","orderbook","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
---


# pallet-otc

**TL;DR:** Lightweight on-chain order book: makers escrow `asset_out` at fixed price; takers fill partially or fully. No matching engine — orders sit until taken or cancelled. Runtime index = 64.

## Role

Implements [[wiki/otc-trading\|otc-trading]]. Complements AMM venues for large-size trades that would otherwise incur slippage on [[wiki/pallet-omnipool\|pallet-omnipool]] or [[wiki/pallet-stableswap\|pallet-stableswap]].

## Config trait (excerpt)

```rust
// pallets/otc/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    type AssetRegistry: Inspect<AssetId = Self::AssetId>;
    type Currency: NamedMultiReservableCurrency<Self::AccountId, CurrencyId = Self::AssetId, Balance = Balance>;
    type ExistentialDeposits: GetByKey<Self::AssetId, Balance>;
    #[pallet::constant] type Fee: Get<Permill>;
    #[pallet::constant] type FeeReceiver: Get<Self::AccountId>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Orders` | StorageMap | `OrderId → Order` |
| `NextOrderId` | StorageValue | `OrderId` |

## Events

`Placed`, `PartiallyFilled`, `Filled`, `Cancelled`.

## Errors

`AssetNotRegistered`, `OrderNotFound`, `SameAssets`, `AmountTooLow`, `InsufficientReservedAmount`, `InsufficientBalance`, `InvalidPartialFillAmount`, `OrderNotPartiallyFillable`, `Forbidden`, `Overflow`, `MathError`.

## Extrinsics

| Name | Description |
|------|-------------|
| `place_order` | Maker escrows `amount_out` of `asset_out`, offers at fixed rate for `amount_in` of `asset_in` |
| `fill_order` | Taker fills entire order |
| `partial_fill_order` | Taker fills part of a partially-fillable order |
| `cancel_order` | Maker cancels and reclaims escrow |

## Public API (non-extrinsic)

`fill_order_with_deferred_delivery` — **collect-before-deliver** fill, added for [[wiki/pallet-otc-settlements\|pallet-otc-settlements]].

```rust
// pallets/otc/src/lib.rs
/// Fill an order (fully or partially) where the filler provides `asset_in`
/// *after* receiving `asset_out`, instead of before.
#[require_transactional]
pub fn fill_order_with_deferred_delivery<F>(
    order_id: OrderId,
    filler: &T::AccountId,
    amount_in: Balance,
    deliver: F,                       // F: FnOnce(amount_out_without_fee) -> DispatchResult
) -> DispatchResult
```

Order of operations: unreserve + pay the maker's `asset_out` (minus fee) to `filler` → run `deliver(amount_out_without_fee)` so the filler can source `asset_in` from it (e.g. a router sell) → pull `amount_in` of `asset_in` from `filler` to the maker. Because the maker's `asset_out` is drained before `asset_in` comes back, the pallet holds a `frame_system::inc_providers` reference on the maker across the gap so a maker holding only `asset_out` is not reaped (nonce reset). Must run inside a transaction — any failure rolls the whole fill back.

Refactor helpers extracted from `fill_order` / `partial_fill_order` and shared with it: `partial_amount_out`, `ensure_remaining_order_valid`, `release_reserved_asset_out`, `deposit_fill_events`.

## Hooks

None.

## Integration

- **Traits implemented:** none external
- **Traits consumed:** `NamedMultiReservableCurrency`, `RegistryInspect` ([[wiki/pallet-asset-registry\|pallet-asset-registry]])
- **Pallets depended on:** [[wiki/pallet-asset-registry\|pallet-asset-registry]], [[wiki/pallet-broadcast\|pallet-broadcast]] (`Swapped` events via `deposit_trade_event`)

## Key extrinsic: place_order

```rust
// pallets/otc/src/lib.rs
pub fn place_order(
    origin: OriginFor<T>,
    asset_in: T::AssetId, asset_out: T::AssetId,
    amount_in: Balance, amount_out: Balance,
    partially_fillable: bool,
) -> DispatchResult {
    let who = ensure_signed(origin)?;
    ensure!(asset_in != asset_out, Error::<T>::SameAssets);
    T::Currency::reserve_named(&T::NamedReserveId::get(), asset_out, &who, amount_out)?;
    let id = NextOrderId::<T>::mutate(|i| { let id = *i; *i += 1; id });
    Orders::<T>::insert(id, Order { owner: who, asset_in, asset_out, amount_in, amount_out, partially_fillable });
    // ...
}
```

## Gotchas

- Orders are fixed-price (no book) — matching happens off-chain.
- Maker's `asset_out` stays reserved (via named-reserve) until filled or cancelled.
- `Fee` is charged out of `amount_out` (the maker's escrowed asset) and routed to `FeeReceiver` — the filler receives `amount_out - fee`.
- Orders cannot be modified — only cancelled and re-placed.
- Consumed by [[wiki/pallet-otc-settlements\|pallet-otc-settlements]] for arbitrage against [[wiki/pallet-omnipool\|pallet-omnipool]] — it now uses `fill_order_with_deferred_delivery`, not `partial_fill_order`/`fill_order`.
- **Residual-order validation changed:** `ensure_remaining_order_valid` checks the leftover `asset_out` leg net of the fee the *remaining* order would pay when filled, matching `place_order`. Previously the check subtracted the fee of the *current* fill, so an order could be fillable through one entry point and not another.
- The maker is paid **first** in `fill_order` / `partial_fill_order` (filler → maker `asset_in`, then unreserve `asset_out`) so the maker's account keeps a live balance while its reserve is released. `fill_order_with_deferred_delivery` inverts this and compensates with a provider reference.
- Broadcast `Swapped` events report swapper/filler in **opposite orders** for full vs partial fills (full: `(filler, owner)`; partial: `(owner, filler)`). This asymmetry is pre-existing and deliberately preserved in `deposit_fill_events`.
- `Filled` / `PartiallyFilled` events are marked deprecated in favour of the broadcast `Swapped` event.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/otc-trading\|otc-trading]]
