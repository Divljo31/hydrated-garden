---
{"dg-publish":true,"permalink":"/wiki/pallet-dispenser/","title":"pallet-dispenser","tags":["faucet","evm","dispenser","signet","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-dispenser","repo":"hydration-node","paths":["pallets/dispenser/src/lib.rs","pallets/dispenser/src/types.rs"],"symbols":["Pallet","Config","request_fund","DispenserConfig","FaucetBalanceWei","UsedRequestIds","EvmTransactionParams"],"traits_impl":[],"depends_on":["pallet-signet"],"runtime_index":85,"tags":["faucet","evm","dispenser","signet","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-dispenser

**TL;DR:** Bridge-side ETH faucet. User pays a fee (in HDX) + the requested amount (in WETH) on Hydration; the pallet submits a signing request to [[wiki/pallet-signet\|pallet-signet]] which produces a signed EVM tx that is broadcast to Ethereum to actually send ETH from a shared faucet contract. Runtime index = 85.

## Role

Provide ETH gas to users who have Hydration assets but no Ethereum ETH — needed for bridging/withdrawal-fee-paying flows. The pallet doesn't control ETH directly; it orchestrates via SigNet (Hydration's off-chain signing service) + an on-chain EVM faucet contract.

## Config trait (excerpt)

```rust
// pallets/dispenser/src/lib.rs
pub trait Config: frame_system::Config + pallet_signet::Config {
    type Currency: Mutate<Self::AccountId, AssetId = AssetId, Balance = Balance>;
    #[pallet::constant] type MinimumRequestAmount: Get<Balance>;
    #[pallet::constant] type MaxDispenseAmount: Get<Balance>;
    #[pallet::constant] type DispenserFee: Get<Balance>;
    #[pallet::constant] type FeeAsset: Get<AssetId>;        // typically HDX
    #[pallet::constant] type FaucetAsset: Get<AssetId>;     // typically WETH
    #[pallet::constant] type FeeDestination: Get<Self::AccountId>;
    #[pallet::constant] type FaucetAddress: Get<EvmAddress>; // Ethereum faucet contract
    #[pallet::constant] type PalletId: Get<PalletId>;
    #[pallet::constant] type MinFaucetEthThreshold: Get<Balance>;
    type WeightInfo: crate::WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `DispenserConfig` | StorageValue | `DispenserConfigData` (paused flag) |
| `FaucetBalanceWei` | StorageValue | `Balance` (off-chain-tracked ETH balance) |
| `UsedRequestIds` | StorageMap | `Bytes32 → ()` (anti-replay) |

## Events

`Paused`, `Unpaused`, `FundRequested` (request_id, requester, to, amount), `FaucetBalanceUpdated`.

## Errors

`DuplicateRequest`, `Serialization`, `InvalidOutput`, `InvalidRequestId`, `Paused`, `AmountTooSmall`, `AmountTooLarge`, `InvalidAddress`, `FaucetBalanceBelowThreshold`, `NotEnoughFeeFunds`, `NotEnoughFaucetFunds`.

## Extrinsics

| Name | Description |
|------|-------------|
| `request_fund` | Caller pays fee + amount, pallet builds EVM tx & submits via `pallet_signet::sign_bidirectional` |

## Hooks

None.

## Integration

- **Traits implemented:** none external
- **Traits consumed:** `Mutate` (fungibles), signet's `sign_bidirectional` API
- **Pallets depended on:** [[wiki/pallet-signet\|pallet-signet]]

## Key extrinsic

```rust
// pallets/dispenser/src/lib.rs
pub fn request_fund(
    origin: OriginFor<T>, to: EvmAddress, amount: Balance,
    request_id: Bytes32, tx: EvmTransactionParams,
) -> DispatchResult {
    let requester = ensure_signed(origin)?;
    Self::ensure_not_paused()?;
    ensure!(to != EvmAddress::zero(), Error::<T>::InvalidAddress);
    ensure!(amount >= T::MinimumRequestAmount::get(), Error::<T>::AmountTooSmall);
    let observed = FaucetBalanceWei::<T>::get();
    ensure!(observed >= T::MinFaucetEthThreshold::get() + amount,
            Error::<T>::FaucetBalanceBelowThreshold);
    <T as Config>::Currency::transfer(T::FeeAsset::get(), &requester,
        &T::FeeDestination::get(), T::DispenserFee::get(), Preservation::Expendable)?;
    <T as Config>::Currency::transfer(T::FaucetAsset::get(), &requester,
        &T::FeeDestination::get(), amount, Preservation::Expendable)?;
    // build + submit signing request to SigNet...
    Ok(())
}
```

## Gotchas

- Does **not** actually send ETH from this chain — it submits a signing request to SigNet, which relays a signed tx to Ethereum.
- `FaucetBalanceWei` is updated off-chain (by an operator) via a privileged call path (not shown in public extrinsics here).
- Both `FeeAsset` (HDX) and `FaucetAsset` (WETH on Hydration) are charged upfront; the user's WETH is effectively swapped for real ETH delivered on Ethereum.
- `MinFaucetEthThreshold` ensures the faucet retains a reserve buffer.
- `UsedRequestIds` prevents replay; request_id is derived from pallet account + RLP-encoded tx params.
- Uses `alloy_primitives` (Rust Ethereum lib) for EVM tx construction.
- Pausing goes through `UpdateOrigin`-gated extrinsics (governance).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/pallet-signet\|pallet-signet]]
