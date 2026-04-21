---
{"dg-publish":true,"permalink":"/wiki/pallet-otc/","title":"pallet-otc","tags":["otc","orderbook","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-otc","repo":"hydration-node","paths":["pallets/otc/src/lib.rs","pallets/otc/src/types.rs"],"symbols":["Pallet","Config","Order","NextOrderId","Orders","place_order","fill_order","partial_fill_order","cancel_order"],"traits_impl":["NamedMultiReservableCurrency"],"depends_on":[],"runtime_index":64,"tags":["otc","orderbook","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
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

## Hooks

None.

## Integration

- **Traits implemented:** none external
- **Traits consumed:** `NamedMultiReservableCurrency`, `RegistryInspect` ([[wiki/pallet-asset-registry\|pallet-asset-registry]])
- **Pallets depended on:** [[wiki/pallet-asset-registry\|pallet-asset-registry]]

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
- `Fee` is charged to the taker in `asset_in`, routed to `FeeReceiver`.
- Orders cannot be modified — only cancelled and re-placed.
- Consumed by [[wiki/pallet-otc-settlements\|pallet-otc-settlements]] for arbitrage against [[wiki/pallet-omnipool\|pallet-omnipool]].

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/otc-trading\|otc-trading]]
