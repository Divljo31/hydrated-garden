---
{"dg-publish":true,"permalink":"/wiki/pallet-currencies/","title":"pallet-currencies","tags":["currency","erc20","native","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-currencies","repo":"hydration-node","paths":["pallets/currencies/src/lib.rs","pallets/currencies/src/fungibles.rs"],"symbols":["Pallet","Config","transfer","transfer_native_currency","update_balance","BalanceOf","CurrencyIdOf"],"traits_impl":["MultiCurrency","MultiCurrencyExtended","MultiLockableCurrency","MultiReservableCurrency","NamedMultiReservableCurrency"],"depends_on":["pallet-asset-registry","pallet-circuit-breaker"],"runtime_index":79,"tags":["currency","erc20","native","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-currencies

**TL;DR:** Hybrid currency adapter. Unifies native balance (`pallet_balances`), ORML multi-currency tokens (`orml_tokens`), and ERC-20-backed assets (`BoundErc20`) behind a single `MultiCurrency` interface. Runtime index = 79.

## Role

One API, many backends. Every pallet in the runtime that moves value uses `Currencies` so the same call works for HDX (native), USDT (orml token), or a bound ERC-20 — `GetNativeCurrencyId` (=HDX) switches between native and multi backends; bound ERC-20s route through `BoundErc20`.

## Config trait (excerpt)

```rust
// pallets/currencies/src/lib.rs
pub trait Config: frame_system::Config {
    type MultiCurrency: TransferAll<Self::AccountId>
        + MultiCurrencyExtended<Self::AccountId> /* + Lockable/Reservable/Named */;
    type NativeCurrency: BasicCurrencyExtended<Self::AccountId, ...> /* + Lockable/Reservable */;
    type Erc20Currency: MultiCurrency<Self::AccountId, ...>;
    type BoundErc20: BoundErc20<AssetId = CurrencyIdOf<Self>>;
    #[pallet::constant] type ReserveAccount: Get<Self::AccountId>;
    #[pallet::constant] type GetNativeCurrencyId: Get<CurrencyIdOf<Self>>;
    type RegistryInspect: hydradx_traits::registry::Inspect<...>;
    type EgressHandler: AssetWithdrawHandler<...>;
    type WeightInfo: WeightInfo;
}
```

## Storage

None — pure adapter; balances live in the underlying backends (`pallet_balances`, `orml_tokens`, ERC-20 contract storage).

## Events

`Transferred`, `BalanceUpdated`, `Deposited`, `Withdrawn`.

## Errors

`AmountIntoBalanceFailed`, `BalanceTooLow`, `DepositFailed`, `NotSupported`.

## Extrinsics

| Name | Description |
|------|-------------|
| `transfer` | Transfer any currency (native / orml / erc20) |
| `transfer_native_currency` | Transfer HDX explicitly (lighter weight) |
| `update_balance` | Root-only balance adjustment (for sudo / migrations) |

## Hooks

None.

## Integration

- **Traits implemented:** `MultiCurrency`, `MultiCurrencyExtended`, `MultiLockableCurrency`, `MultiReservableCurrency`, `NamedMultiReservableCurrency`, `TransferAll`, `fungibles::Inspect/Mutate/Transfer`
- **Traits consumed:** the three backends (native/multi/erc20) + `BoundErc20`, `RegistryInspect`, `EgressHandler`
- **Pallets depended on:** [[wiki/pallet-asset-registry\|pallet-asset-registry]] (asset lookup), [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] (egress tracking), `pallet_balances`, `orml_tokens`

## Key type aliases

```rust
// pallets/currencies/src/lib.rs
pub(crate) type BalanceOf<T> =
    <<T as Config>::MultiCurrency as MultiCurrency<T::AccountId>>::Balance;
pub(crate) type CurrencyIdOf<T> =
    <<T as Config>::MultiCurrency as MultiCurrency<T::AccountId>>::CurrencyId;
pub(crate) type AmountOf<T> =
    <<T as Config>::MultiCurrencyExtended as MultiCurrencyExtended<T::AccountId>>::Amount;
```

## Gotchas

- `AssetId == GetNativeCurrencyId::get()` (typically 0 = HDX) → routes through `NativeCurrency` (= `pallet_balances`).
- `BoundErc20::contract_address(id)` returning `Some` → routes through the EVM ERC-20 precompile via `Erc20Currency`.
- Otherwise → `orml_tokens` via `MultiCurrency`.
- `EgressHandler` notifies circuit-breaker on withdrawal paths for bridge/XCM rate limiting.
- `update_balance` accepts signed `Amount` (can credit or debit); root only.
- `fungibles.rs` provides a separate `fungibles::*` trait surface for FRAME-style code that expects `fungibles::Inspect` rather than ORML `MultiCurrency`.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
