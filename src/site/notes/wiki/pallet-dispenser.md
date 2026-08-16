---
{"dg-publish":true,"permalink":"/wiki/pallet-dispenser/","title":"pallet-dispenser","tags":["faucet","evm","dispenser","signet","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-dispenser","repo":"hydration-node","paths":["pallets/dispenser/src/lib.rs","pallets/dispenser/src/types.rs","pallets/dispenser/src/weights.rs"],"symbols":["Pallet","Config","request_fund","set_config","pause","unpause","DispenserConfig","DispenserConfigData","UsedRequestIds","EvmTransactionParams","IGasFaucet"],"traits_impl":[],"depends_on":["pallet-signet"],"runtime_index":85,"tags":["faucet","evm","dispenser","signet","runtime","rust","substrate"],"last_updated":"2026-05-14"}}
---


# pallet-dispenser

**TL;DR:** Bridge-side ETH faucet. User pays a fee (in HDX) + the requested amount (in WETH) on Hydration; the pallet builds an `IGasFaucet::fund(address,uint256)` EVM transaction and submits it to [[wiki/pallet-signet\|pallet-signet]] for MPC signing, which then broadcasts the signed tx to Ethereum. All runtime knobs (fee amount, faucet address, balance threshold, min/max request) live in a single storage value `DispenserConfig`, **settable on the fly** by `UpdateOrigin`. Runtime index = 85.

## Role

Provide ETH gas to users who hold Hydration assets but no Ethereum ETH (needed for bridging / withdrawal-fee paying flows). The pallet doesn't control ETH directly; it orchestrates via SigNet + an on-chain EVM faucet contract on the destination chain.

## Config trait

```rust
// pallets/dispenser/src/lib.rs
pub trait Config: frame_system::Config<RuntimeEvent: From<Event<Self>>>
    + pallet_signet::Config
{
    /// Origin allowed to call set_config / pause / unpause.
    type UpdateOrigin: EnsureOrigin<Self::RuntimeOrigin>;

    /// Multi-asset fungible currency implementation.
    type Currency: Mutate<Self::AccountId, AssetId = AssetId, Balance = Balance>;

    /// Asset used to charge the per-request fee (HDX).
    #[pallet::constant] type FeeAsset: Get<AssetId>;
    /// Asset deducted to back the faucet delivery (WETH).
    #[pallet::constant] type FaucetAsset: Get<AssetId>;
    /// Account that receives the collected fees and faucet asset.
    #[pallet::constant] type FeeDestination: Get<Self::AccountId>;
    /// Pallet ID for deriving the sovereign account.
    #[pallet::constant] type PalletId: Get<PalletId>;

    type WeightInfo: crate::WeightInfo;
}
```

All other knobs (fee amount, faucet address, balance threshold, min/max request) are **storage**, not constants — see below.

## Storage

| Name | Kind | Description |
|---|---|---|
| `DispenserConfig` | `StorageValue<_, DispenserConfigData, OptionQuery>` | All on-chain-mutable settings. `None` → `NotConfigured`. |
| `UsedRequestIds` | `StorageMap<_, Blake2_128Concat, Bytes32, ()>` | Anti-replay; insert on each `request_fund` |

```rust
pub struct DispenserConfigData {
    pub paused: bool,
    pub faucet_balance_wei: Balance,      // off-chain-tracked ETH reserve
    pub faucet_address: EvmAddress,       // destination-chain faucet contract
    pub min_faucet_threshold: Balance,    // wei floor that must remain after a request
    pub min_request: Balance,
    pub max_dispense: Balance,
    pub dispenser_fee: Balance,           // flat fee, in FeeAsset
}
```

## Events

- `ConfigUpdated { faucet_address, min_faucet_threshold, min_request, max_dispense, dispenser_fee, faucet_balance_wei }`
- `Paused`, `Unpaused`
- `FundRequested { request_id, requester, to: EvmAddress, amount }`

## Errors

`NotConfigured`, `DuplicateRequest`, `Serialization`, `InvalidOutput`, `InvalidRequestId`, `Paused`, `AmountTooSmall`, `AmountTooLarge`, `InvalidAddress`, `FaucetBalanceBelowThreshold`, `NotEnoughFeeFunds`, `NotEnoughFaucetFunds`, `InvalidConfig`.

## Extrinsics

| call_index | Name | Origin | Description |
|---|---|---|---|
| 0 | `request_fund` | Signed | Charge fee + faucet asset, build the EIP-1559 EVM call, submit to SigNet via `pallet_signet::sign_bidirectional` |
| 1 | `set_config` | `UpdateOrigin` | Set / update `DispenserConfigData`. Validates `faucet_address ≠ 0`, `max_dispense > 0`, `min_request ≤ max_dispense`. Preserves current `paused`. |
| 2 | `pause` | `UpdateOrigin` | Set `paused = true` (errors if not configured) |
| 3 | `unpause` | `UpdateOrigin` | Set `paused = false` |

## Hooks

None.

## Integration

- **Traits implemented:** none external
- **Traits consumed:** `Mutate` (fungibles)
- **Pallets depended on:** [[wiki/pallet-signet\|pallet-signet]] (calls `sign_bidirectional` to dispatch the EVM tx for MPC signing)

## Key extrinsic (sketch)

```rust
// pallets/dispenser/src/lib.rs (abridged)
pub fn request_fund(
    origin: OriginFor<T>,
    to: EvmAddress,
    amount: Balance,
    request_id: Bytes32,
    tx: EvmTransactionParams,
) -> DispatchResult {
    let requester = ensure_signed(origin)?;
    let cfg = DispenserConfig::<T>::get().ok_or(Error::<T>::NotConfigured)?;
    ensure!(!cfg.paused, Error::<T>::Paused);
    ensure!(to != EvmAddress::zero(), Error::<T>::InvalidAddress);
    ensure!(amount >= cfg.min_request,  Error::<T>::AmountTooSmall);
    ensure!(amount <= cfg.max_dispense, Error::<T>::AmountTooLarge);
    ensure!(cfg.faucet_balance_wei >= cfg.min_faucet_threshold + amount,
            Error::<T>::FaucetBalanceBelowThreshold);
    ensure!(!UsedRequestIds::<T>::contains_key(&request_id),
            Error::<T>::DuplicateRequest);

    // 1. Charge fee in FeeAsset (HDX)
    <T as Config>::Currency::transfer(T::FeeAsset::get(), &requester,
        &T::FeeDestination::get(), cfg.dispenser_fee, Preservation::Expendable)?;
    // 2. Take collateral in FaucetAsset (WETH)
    <T as Config>::Currency::transfer(T::FaucetAsset::get(), &requester,
        &T::FeeDestination::get(), amount, Preservation::Expendable)?;
    // 3. Build IGasFaucet::fund(to, amount) call data via alloy_sol_types
    // 4. Wrap in EIP-1559 tx params, RLP-encode, derive request_id, ensure match
    // 5. Forward to pallet_signet::sign_bidirectional
    UsedRequestIds::<T>::insert(&request_id, ());
    Self::deposit_event(Event::FundRequested { request_id, requester, to, amount });
    Ok(())
}
```

## EVM call ABI

```rust
sol! {
    #[sol(abi)]
    interface IGasFaucet {
        function fund(address to, uint256 amount) external;
    }
}
```

The pallet uses `alloy_primitives` + `alloy_sol_types` to encode the call data; transaction wrapping is handled by `pallet_signet::build_evm_tx`.

## Gotchas

- **Does not send ETH from this chain.** It submits a SigNet request; SigNet (off-chain) signs and broadcasts to Ethereum. There is no on-chain confirmation that the destination transaction actually included — only that the signing request was emitted.
- **Most config is on-chain mutable.** Fee amount, faucet address, threshold, min/max — all live in `DispenserConfigData` and can be changed by `UpdateOrigin` via `set_config` without a runtime upgrade. Previous wiki revisions documented these as `#[pallet::constant]` — that's no longer accurate.
- **`faucet_balance_wei` is an oracle field.** It's tracked off-chain and written into `DispenserConfigData` via `set_config`; the pallet does not poll real ETH balances. If this drifts from reality, `FaucetBalanceBelowThreshold` either over-rejects or over-accepts.
- **`UsedRequestIds`** prevents replay; `request_id` is deterministically derived from `(pallet account, RLP-encoded tx params)` and must match what the caller supplies (`InvalidRequestId` otherwise).
- **Both `FeeAsset` (HDX) and `FaucetAsset` (WETH on Hydration) are charged upfront** — the user effectively swaps WETH on Hydration for real ETH delivered on Ethereum.
- **Pausing** uses `set` of `paused` inside `DispenserConfigData` — gracefully errors if `DispenserConfig::get()` is `None`.
- The pallet implicitly trusts `pallet_signet` to forward the request faithfully; verification of the signed transaction happens off-chain at the relayer.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/pallet-signet\|pallet-signet]]
