---
{"dg-publish":true,"permalink":"/wiki/pallet-dispatcher/","title":"pallet-dispatcher","tags":["dispatcher","batching","governance","evm","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-dispatcher","repo":"hydration-node","paths":["pallets/dispatcher/src/lib.rs","pallets/dispatcher/src/weights.rs"],"symbols":["Pallet","Config","dispatch_as_treasury","dispatch_as_aave_manager","dispatch_as_emergency_admin","dispatch_with_extra_gas","dispatch_evm_call","dispatch_with_fee_payer","note_aave_manager","AaveManagerAccount","ExtraGas","LastEvmCallExitReason","EvmCallChecker","MaybeEvmCall","EvmFeePayerSupport","ExtraGasSupport"],"traits_impl":["ExtraGasSupport"],"depends_on":["pallet-evm"],"runtime_index":40,"tags":["dispatcher","batching","governance","evm","runtime","rust","substrate"],"last_updated":"2026-05-14"}}
---


# pallet-dispatcher

**TL;DR:** Authorised-origin call dispatcher. Lets specific governance origins (Treasurer, AaveManager, EmergencyAdmin = TC majority) execute calls as a pallet-controlled signed account, plus EVM-specific dispatch paths: gas-augmented dispatch, single EVM call dispatch (with exit-reason validation), and fee-payer override. Runtime index = 40. (ISMP cleanup hook removed in spec 419 along with Hyperbridge.)

## Role

Three jobs rolled into one pallet:
1. **Dispatch-as**: A governance origin dispatches an inner call as if signed by a designated managed account (Treasury account, Aave manager account, Emergency Admin account).
2. **Extra gas / EVM call**: Wrap an inner call with declared extra EVM gas allowance (`dispatch_with_extra_gas`), or run a single EVM call and validate the exit reason (`dispatch_evm_call`).
3. **Fee payer override**: Temporarily override which account pays for an EVM call (`dispatch_with_fee_payer`) via `EvmFeePayerSupport`.

## Config trait (excerpt)

```rust
// pallets/dispatcher/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeCall: IsType<<Self as frame_system::Config>::RuntimeCall>
        + Dispatchable<RuntimeOrigin = Self::RuntimeOrigin, PostInfo = PostDispatchInfo>
        + GetDispatchInfo + FullCodec + TypeInfo
        + From<frame_system::Call<Self>> + Parameter;

    // Identifies whether a RuntimeCall is `pallet_evm::Call::call`.
    type EvmCallIdentifier: MaybeEvmCall<<Self as Config>::RuntimeCall>;

    type TreasuryManagerOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type AaveManagerOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type EmergencyAdminOrigin: EnsureOrigin<Self::RuntimeOrigin>;

    type TreasuryAccount: Get<Self::AccountId>;
    type DefaultAaveManagerAccount: Get<Self::AccountId>;
    type EmergencyAdminAccount: Get<Self::AccountId>;

    type GasWeightMapping: GasWeightMapping;
    type EvmFeePayer: EvmFeePayerSupport<AccountId = Self::AccountId>;
    type WeightInfo: WeightInfo;
}
```

Runtime wiring (`runtime/hydradx/src/governance/mod.rs`):

```rust
type TreasuryManagerOrigin = EitherOf<EnsureRoot<AccountId>, Treasurer>;
type AaveManagerOrigin     = EitherOf<EnsureRoot<AccountId>, EconomicParameters>;
type EmergencyAdminOrigin  = EitherOf<EnsureRoot<AccountId>, TechCommitteeMajority>;
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `AaveManagerAccount` | StorageValue | `AccountId` (ValueQuery, default = `DefaultAaveManagerAccount`) |
| `ExtraGas` | StorageValue | `u64` (ValueQuery, whitelist) — extra gas for the currently-running outer call |
| `LastEvmCallExitReason` | StorageValue | `Option<ExitReason>` (whitelist, unbounded) — captured during EVM execution, reset in `on_finalize` |

## Events

`TreasuryManagerCallDispatched { call_hash, result }`, `AaveManagerCallDispatched { call_hash, result }`, `EmergencyAdminCallDispatched { call_hash, result }`.

## Errors

`EvmCallFailed`, `NotEvmCall`, `EvmOutOfGas`, `EvmArithmeticOverflowOrUnderflow`, `AaveSupplyCapExceeded`, `AaveBorrowCapExceeded`, `AaveHealthFactorNotBelowThreshold`, `AaveHealthFactorLowerThanLiquidationThreshold`, `CollateralCannotCoverNewBorrow`, `AaveReservePaused`.

## Extrinsics

| call_index | Name | Description |
|---|---|---|
| 0 | `dispatch_as_treasury` | `TreasuryManagerOrigin` dispatches call as `TreasuryAccount` signed origin |
| 1 | `dispatch_as_aave_manager` | `AaveManagerOrigin` dispatches call as `AaveManagerAccount` signed origin |
| 2 | `note_aave_manager` | Root sets the `AaveManagerAccount` in storage (mainly for testnet) |
| 3 | `dispatch_with_extra_gas` | Any signed origin; sets `ExtraGas` for the duration of the inner call, then clears |
| 4 | `dispatch_evm_call` | Signed; requires inner call to be `pallet_evm::Call::call`; verifies `LastEvmCallExitReason ∈ {Succeed(Returned), Succeed(Stopped)}` or returns `EvmCallFailed` |
| 5 | `dispatch_as_emergency_admin` | `EmergencyAdminOrigin` (TC majority or Root) dispatches as `EmergencyAdminAccount` — fast path to react to incidents (e.g. pause an exploited market) without a full referendum |
| 7 | `dispatch_with_fee_payer` | Signed; sets the EVM fee payer to the signer for the duration of the inner call (via `EvmFeePayerSupport`); restores the previous payer afterwards |

(Call index 6 is unused.)

## Hooks

`on_finalize` — `LastEvmCallExitReason::kill()` resets the per-block storage. No `on_idle` cleanup any more; the previous ISMP cleanup hook was removed alongside the Hyperbridge pallets in spec 419.

## Integration

- **Traits implemented:** `ExtraGasSupport` for `Pallet<T>` (`set_extra_gas`, `clear_extra_gas`, `out_of_gas_error`)
- **Traits consumed:** `GasWeightMapping`, `MaybeEvmCall`, `EvmFeePayerSupport`
- **Pallets depended on:** `pallet_evm` (for `ExitReason`, `GasWeightMapping`)
- **Public API:** `Pallet::decrease_extra_gas(amount)`, `Pallet::set_last_evm_call_exit_reason(reason)` — called from the EVM runner wrapper

## Key extrinsic

```rust
// pallets/dispatcher/src/lib.rs
pub fn dispatch_as_emergency_admin(
    origin: OriginFor<T>, call: Box<<T as Config>::RuntimeCall>,
) -> DispatchResultWithPostInfo {
    T::EmergencyAdminOrigin::ensure_origin(origin)?;
    let call_hash = T::Hashing::hash_of(&call);
    let (result, actual_weight) = Self::do_dispatch(
        frame_system::Origin::<T>::Signed(T::EmergencyAdminAccount::get()).into(),
        *call,
    );
    Self::deposit_event(Event::<T>::EmergencyAdminCallDispatched { call_hash, result });
    Ok(actual_weight.map(|w| w.saturating_add(/* base */)).into())
}
```

## Gotchas

- **No more ISMP / Hyperbridge support.** The `on_idle` hook that called `pallet_ismp::cleanup_requests` was removed in spec 419 along with the deletion of Hyperbridge / ISMP / token-gateway pallets. Old wiki notes referencing `CLEANUP_LIMIT` and ISMP housekeeping are obsolete.
- **`dispatch_with_extra_gas`** is a gas inflation tool — be careful with weight accounting; post-dispatch weight equals declared-extra (via `GasWeightMapping`) + actual inner.
- **`dispatch_evm_call`** rejects any call that is not `pallet_evm::Call::call` (checked by `EvmCallIdentifier`). It also reads `LastEvmCallExitReason` set by the EVM runner wrapper and fails the extrinsic if the EVM execution returned anything other than `Succeed(Returned)` or `Succeed(Stopped)`. This is the only path where EVM call failures bubble up as proper substrate errors.
- **`dispatch_with_fee_payer`** sets `T::EvmFeePayer::set_fee_payer(signer)` for the duration of the inner call and restores the previous payer (or clears) afterwards. Used to let one account sponsor another's EVM gas.
- **`EmergencyAdminOrigin`** wired to TC majority — this is the chain's emergency response lever. Dispatched calls run as the `EmergencyAdminAccount` signed origin, so the inner call sees that account as the caller.
- Treasury / Aave manager / EmergencyAdmin accounts are dispatched as `Signed` — they can therefore hold balances, hold positions, be referenced as `from` in asset operations, etc.
- **`AaveManagerAccount`** uses `ValueQuery` with `DefaultAaveManagerAccount` — never panics on read, always returns the runtime-default if `note_aave_manager` has not been called.
- **No pause functionality** here — see [[wiki/pallet-transaction-pause\|pallet-transaction-pause]].

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hydration-runtime\|hydration-runtime]]
