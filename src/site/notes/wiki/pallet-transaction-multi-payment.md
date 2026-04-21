---
{"dg-publish":true,"permalink":"/wiki/pallet-transaction-multi-payment/","title":"pallet-transaction-multi-payment","tags":["fees","payment","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-transaction-multi-payment","repo":"hydration-node","paths":["pallets/transaction-multi-payment/src/lib.rs","pallets/transaction-multi-payment/src/types.rs","pallets/transaction-multi-payment/src/traits.rs"],"symbols":["Pallet","Config","AcceptedCurrencies","AccountCurrencyMap","TransferFees","set_currency","add_currency","remove_currency","dispatch_permit","CurrencyBalanceCheck"],"traits_impl":["OnChargeTransaction","PaymentSwapResult"],"depends_on":["pallet-transaction-payment","pallet-route-executor"],"runtime_index":9,"tags":["fees","payment","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-transaction-multi-payment

**TL;DR:** Lets users pay transaction fees in any accepted non-native asset. Converts fee from the user's chosen currency to HDX at trade-time (via [[wiki/pallet-route-executor\|pallet-route-executor]]) and burns/forwards to treasury. Runtime index = 9 (replaces stock `pallet-transaction-payment` for fee handling).

## Role

Removes the "you must hold HDX to transact" UX barrier. Every accepted currency has an on-chain price feed via [[wiki/pallet-ema-oracle\|pallet-ema-oracle]]; the pallet computes the per-extrinsic fee in native HDX then swaps from the user's balance.

## Config trait (excerpt)

```rust
// pallets/transaction-multi-payment/src/lib.rs
pub trait Config: frame_system::Config + pallet_transaction_payment::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AcceptedCurrencyOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type Currencies: MultiCurrency<Self::AccountId, CurrencyId = AssetId, Balance = Balance>
        + MultiCurrencyExtended<Self::AccountId>;
    type RouteProvider: RouteProvider<AssetId>;
    type OraclePriceProvider: PriceOracle<AssetId, Price = EmaPrice>;
    type SpotPriceProvider: SpotPriceProvider<AssetId, Price = FixedU128>;
    type WeightInfo: WeightInfo;
    #[pallet::constant] type WeightToFee: WeightToFee<Balance = Balance>;
    #[pallet::constant] type NativeAssetId: Get<AssetId>;
    type PolkadotNativeAssetId: Get<AssetId>;
    type EvmAssetId: Get<AssetId>;
    type InspectEvmAccounts: InspectEvmAccounts<Self::AccountId>;
    type EvmPermit: EvmPermitHandler<Self>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `AcceptedCurrencies` | StorageMap | `AssetId → Option<Price>` (fallback spot price) |
| `AccountCurrencyMap` | StorageMap | `AccountId → AssetId` (user's chosen fee currency) |
| `AcceptedCurrencyPrice` | StorageMap | `AssetId → Balance` (last-known fallback price) |
| `TransferFees` | StorageMap | `(AssetId, AccountId) → Balance` |

## Events

`CurrencySet`, `CurrencyAdded`, `CurrencyRemoved`, `FeeWithdrawn`.

## Errors

`UnsupportedCurrency`, `ZeroBalance`, `AlreadyAccepted`, `CoreAssetNotAllowed`, `ZeroPrice`, `FallbackPriceNotFound`, `MathOverflow`.

## Extrinsics

| Name | Description |
|------|-------------|
| `set_currency` | User selects their preferred fee currency |
| `add_currency` | Governance adds an accepted fee currency with fallback price |
| `remove_currency` | Governance removes an accepted fee currency |
| `reset_payment_currency` | Resets account's fee currency to native (HDX) |
| `dispatch_permit` | Dispatches an EVM permit transaction (consumed by EVM call-permit precompile) |

## Hooks

None.

## Integration

- **Traits implemented:** `OnChargeTransaction` (replaces default HDX-only fee handler)
- **Traits consumed:** `RouteProvider` ([[wiki/pallet-route-executor\|pallet-route-executor]]), `PriceOracle` ([[wiki/pallet-ema-oracle\|pallet-ema-oracle]]), `SpotPriceProvider`, `InspectEvmAccounts`, `EvmPermitHandler`
- **Pallets depended on:** [[pallet-transaction-payment\|pallet-transaction-payment]], [[wiki/pallet-route-executor\|pallet-route-executor]], [[wiki/pallet-ema-oracle\|pallet-ema-oracle]], [[wiki/pallet-evm-accounts\|pallet-evm-accounts]]

## Key logic: fee charging

```rust
// pallets/transaction-multi-payment/src/lib.rs
impl<T: Config> OnChargeTransaction<T> for Pallet<T> {
    fn withdraw_fee(who, call, info, fee, tip) -> Result<Self::LiquidityInfo, _> {
        let currency = AccountCurrencyMap::<T>::get(who).unwrap_or(T::NativeAssetId::get());
        if currency == T::NativeAssetId::get() {
            return <pallet_transaction_payment::Pallet<T>>::withdraw_fee(...);
        }
        let price = Self::price(currency)?;
        let fee_in_currency = fee.saturating_mul(price);
        T::Currencies::withdraw(currency, who, fee_in_currency)?;
        // swap via router to HDX, deposit to treasury
        Ok(LiquidityInfo { currency, amount: fee_in_currency })
    }
}
```

## Gotchas

- EVM transactions: fee is charged in WETH (the EVM gas asset in Hydration) unless user overrides.
- Fallback price is used when oracle is temporarily unavailable (new asset or oracle underfunded) — governance-set via `add_currency`.
- Per-user currency preference persists across sessions (stored in `AccountCurrencyMap`).
- Fee swap uses the router's default route (cached in `pallet-route-executor`'s `Routes` storage).
- `dispatch_permit` bridges EVM EIP-2612 permits into Substrate dispatch for gasless-style UX.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
