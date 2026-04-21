---
{"dg-publish":true,"permalink":"/wiki/pallet-dynamic-evm-fee/","title":"pallet-dynamic-evm-fee","tags":["evm","fees","dynamic","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-dynamic-evm-fee","repo":"hydration-node","paths":["pallets/dynamic-evm-fee/src/lib.rs","pallets/dynamic-evm-fee/src/types.rs"],"symbols":["Pallet","Config","BaseFeePerGas","MinBaseFeePerGas","DefaultBaseFeePerGasValue"],"traits_impl":["FeeCalculator"],"depends_on":["pallet-transaction-payment"],"runtime_index":94,"tags":["evm","fees","dynamic","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-dynamic-evm-fee

**TL;DR:** Computes EVM `BaseFeePerGas` dynamically from the Substrate extrinsic length multiplier so EVM and non-EVM transactions pay comparable fees under congestion. Runtime index = 94.

## Role

Provides `FeeCalculator` for [[wiki/pallet-frontier\|pallet-frontier]] (`pallet-evm`). Bridges Substrate's fee-multiplier (used by `pallet-transaction-payment`) into an EVM-compatible gas price so economic fees are aligned across dispatch paths.

## Config trait (excerpt)

```rust
// pallets/dynamic-evm-fee/src/lib.rs
pub trait Config: frame_system::Config + pallet_transaction_payment::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    #[pallet::constant] type DefaultBaseFeePerGas: Get<u128>;
    #[pallet::constant] type MinBaseFeePerGas: Get<u128>;
    #[pallet::constant] type MaxBaseFeePerGas: Get<u128>;
    #[pallet::constant] type FeeMultiplier: Get<Multiplier>;
    #[pallet::constant] type WethAssetId: Get<Self::AssetId>;
    type NativePriceOracle: NativePriceOracle<Self::AssetId, EmaPrice>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `BaseFeePerGas` | StorageValue | `U256` |

## Events

`BaseFeePerGasChanged`.

## Errors

None (pure computation).

## Extrinsics

None — updated via `on_initialize` hook.

## Hooks

`on_initialize` reads Substrate fee-multiplier, computes new `BaseFeePerGas`, clamps to `[MinBaseFeePerGas, MaxBaseFeePerGas]`, stores it. Also emits `BaseFeePerGasChanged` when value changes.

## Integration

- **Traits implemented:** `FeeCalculator` (consumed by `pallet-evm`'s `FeeCalculator` config)
- **Traits consumed:** `NativePriceOracle`, `TransactionPayment` fee multiplier
- **Pallets depended on:** [[pallet-transaction-payment\|pallet-transaction-payment]], [[wiki/pallet-frontier\|pallet-frontier]] (indirectly)

## Key logic

```rust
// pallets/dynamic-evm-fee/src/lib.rs
fn on_initialize(_: BlockNumberFor<T>) -> Weight {
    let multiplier = <pallet_transaction_payment::NextFeeMultiplier<T>>::get();
    let base = T::DefaultBaseFeePerGas::get();
    let next = multiplier.saturating_mul_int(base);
    let clamped = next.clamp(T::MinBaseFeePerGas::get(), T::MaxBaseFeePerGas::get());
    BaseFeePerGas::<T>::put(U256::from(clamped));
    T::WeightInfo::on_initialize()
}
```

## Gotchas

- `FeeCalculator::min_gas_price()` returns the stored `BaseFeePerGas` — called by `pallet-evm` on every EVM transaction.
- EVM Chain ID = 222 (Hydration mainnet).
- WETH is the native-pair token used for pricing (see `WethAssetId`).
- Congestion on non-EVM side raises EVM fees, preserving fee parity.
- Min fee prevents gas-price from dropping below a dust threshold; max fee caps spikes.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/dynamic-fees\|dynamic-fees]]
- [[wiki/pallet-frontier\|pallet-frontier]]
