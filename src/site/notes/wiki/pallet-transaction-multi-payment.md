---
{"dg-publish":true,"permalink":"/wiki/pallet-transaction-multi-payment/","title":"pallet-transaction-multi-payment","tags":["fees","payment","evm","paymaster","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-transaction-multi-payment","repo":"hydration-node","paths":["pallets/transaction-multi-payment/src/lib.rs","pallets/transaction-multi-payment/src/traits.rs","pallets/transaction-multi-payment/src/weights.rs"],"symbols":["Pallet","Config","AcceptedCurrencies","AccountCurrencyMap","TransferFees","set_currency","add_currency","remove_currency","dispatch_permit","do_dispatch_permit_unsigned","do_dispatch_permit_signed","restore_fee_payer","permit_fee_currency","account_currency","EvmPermitCallFailed","EvmFeePayer","EvmFeePayerSupport","FeeSponsored","TransactionCurrencyOverride","AcceptedCurrencyPrice","AddTxAssetOnAccount","RemoveTxAssetOnKilled"],"traits_impl":["OnChargeTransaction","PaymentSwapResult"],"depends_on":["pallet-transaction-payment","pallet-route-executor","pallet-evm-accounts"],"runtime_index":203,"tags":["fees","payment","evm","paymaster","runtime","rust","substrate"],"last_updated":"2026-11-02"}}
---


# pallet-transaction-multi-payment

**TL;DR:** Lets users pay transaction fees in a **curated set** of non-native assets (`AcceptedCurrencies`). The fee is priced in HDX and converted with the per-block `AcceptedCurrencyPrice`. An asset that is not in `AcceptedCurrencies` cannot be used for fees at all — there is no swap-to-DOT fallback. Runtime index = 203 (`MultiTransactionPayment`; stock `TransactionPayment` is 9).

## Role

Removes the "you must hold HDX to transact" UX barrier. Every accepted currency has an on-chain price feed via [[wiki/pallet-ema-oracle\|pallet-ema-oracle]]; the pallet computes the per-extrinsic fee in native HDX then swaps from the user's balance.

## Config trait (excerpt)

```rust
// pallets/transaction-multi-payment/src/lib.rs
pub trait Config: frame_system::Config + pallet_transaction_payment::Config {
    type AcceptedCurrencyOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type Currencies: MultiCurrency<Self::AccountId>;
    type RouteProvider: RouteProvider<AssetIdOf<Self>>;
    type OraclePriceProvider: PriceOracle<AssetIdOf<Self>, Price = EmaPrice>;
    type WeightInfo: WeightInfo;
    type WeightToFee: WeightToFee<Balance = BalanceOf<Self>>;
    type NativeAssetId: Get<AssetIdOf<Self>>;
    type EvmAssetId: Get<AssetIdOf<Self>>;
    type InspectEvmAccounts: InspectEvmAccounts<Self::AccountId>;
    type EvmPermit: EVMPermit;
    type TryCallCurrency<'a>: TryConvert<&'a <Self as frame_system::Config>::RuntimeCall, AssetIdOf<Self>>;
    type EvmFeePayer: EvmFeePayerSupport<AccountId = Self::AccountId>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `AcceptedCurrencies` | StorageMap | `AssetId → Price` — curated fee currencies, value = governance-set fallback price |
| `AccountCurrencyMap` | StorageMap | `AccountId → AssetId` (user's chosen fee currency) |
| `AcceptedCurrencyPrice` | StorageMap | `AssetId → Price` — oracle price (or fallback if unavailable), refreshed at the start of every block |
| `TransactionCurrencyOverride` | StorageMap | `AccountId → AssetId` — per-account override |

`TransferFees<T, MC, DF, FR, WF>` is the `OnChargeTransaction` **struct**, not storage.

## Events

`CurrencySet`, `CurrencyAdded`, `CurrencyRemoved`, `FeeWithdrawn`, `FeeSponsored { from: H160, fee_payer: AccountId }`, `EvmPermitCallFailed { from, to, nonce, reason: ExitReason, output }` (output bounded to `MAX_EVM_CALL_OUTPUT_LEN = 256` bytes — surfaces the inner EVM error of a failed permit).

## Errors

`UnsupportedCurrency`, `ZeroBalance`, `AlreadyAccepted`, `CoreAssetNotAllowed`, `ZeroPrice`, `FallbackPriceNotFound`, `Overflow`, `EvmAccountNotAllowed`, `EvmPermitExpired`, `EvmPermitInvalid`, `EvmPermitCallExecutionError`, `EvmPermitRunnerError`, `EvmPermitNonceInvariantViolated`.

## Extrinsics

| Name | Description |
|------|-------------|
| `set_currency` | User selects their preferred fee currency |
| `add_currency` | Governance adds an accepted fee currency with fallback price |
| `remove_currency` | Governance removes an accepted fee currency |
| `reset_payment_currency` | Resets account's fee currency to native (HDX) |
| `dispatch_permit` | Dispatches a pre-signed EVM permit. Dual-origin (unsigned / signed paymaster) — see below. Root is rejected. |

Origin for add/remove: `AcceptedCurrencyOrigin = Root | TechCommitteeMajority | GeneralAdmin`.

### `dispatch_permit` — unsigned vs signed (paymaster) branches

`dispatch_permit` branches on the origin:

| Branch | Origin | Who pays | Failure handling |
|---|---|---|---|
| `do_dispatch_permit_unsigned` | none | `permit.from`, via `withdraw_fee` in its chosen currency | Invalid permit → `on_dispatch_permit_error()` (auto-pause) and `Ok(default)`; runner error → `on_dispatch_permit_error()` |
| `do_dispatch_permit_signed` | signed (paymaster) | the **signer** pays both the Substrate extrinsic fee and the EVM gas | Invalid permit → `Err`; runner error → `Err` (`Pays::Yes`, so the paymaster still pays the extrinsic fee); revert → `Ok(post_info)` |

```rust
// pallets/transaction-multi-payment/src/lib.rs
pub(crate) fn do_dispatch_permit_signed(signer, from, to, value, data, gas_limit, deadline, v, r, s)
    -> DispatchResultWithPostInfo
{
    let previous_fee_payer = T::EvmFeePayer::set_fee_payer(signer.clone());
    if let Err(e) = T::EvmPermit::validate_permit(from, to, data.clone(), value, gas_limit, deadline, v, r, s) {
        Self::restore_fee_payer(previous_fee_payer);
        return Err(e.into());
    }
    let (gas_price, _) = T::EvmPermit::gas_price();
    let result = T::EvmPermit::dispatch_permit(from, to, data, value, gas_limit, gas_price, None, None, vec![]);
    // Restore AFTER the dispatch so the runner's post-execution refund
    // (`correct_and_deposit_fee`) also routes to the paymaster.
    Self::restore_fee_payer(previous_fee_payer);
    // ... FeeSponsored event on success
}
```

## Hooks

None.

## Integration

- **Traits implemented:** `OnChargeTransaction` (replaces default HDX-only fee handler)
- **Traits consumed:** `RouteProvider` ([[wiki/pallet-route-executor\|pallet-route-executor]]), `PriceOracle` ([[wiki/pallet-ema-oracle\|pallet-ema-oracle]]), `InspectEvmAccounts`, `EVMPermit`
- **Pallets depended on:** [[pallet-transaction-payment\|pallet-transaction-payment]], [[wiki/pallet-route-executor\|pallet-route-executor]], [[wiki/pallet-ema-oracle\|pallet-ema-oracle]], [[wiki/pallet-evm-accounts\|pallet-evm-accounts]]

## Key logic: fee currency resolution

```rust
// pallets/transaction-multi-payment/src/lib.rs — account_currency()
Pallet::<T>::get_currency(who)
    .filter(|c| *c == T::NativeAssetId::get() || AcceptedCurrencies::<T>::contains_key(c))
    .unwrap_or_else(|| if T::InspectEvmAccounts::is_evm_account(who.clone()) {
        T::EvmAssetId::get()      // WETH (20) for EVM accounts
    } else {
        T::NativeAssetId::get()   // HDX
    })
```

`permit_fee_currency(account, data)` does the same for `dispatch_permit`: decode the inner call (depth limit 32), try `TryCallCurrency`, keep it only if HDX or accepted, else fall back to `account_currency`. Fee amount = native fee × `AcceptedCurrencyPrice[currency]` (missing price → `InvalidTransaction::Payment`).

## Gotchas

- EVM transactions: fee is charged in WETH (the EVM gas asset in Hydration) unless user overrides.
- Fallback price (the `AcceptedCurrencies` value, set by `add_currency`) is used when the oracle price is unavailable.
- **Only `AcceptedCurrencies` (+ HDX) can pay fees.** A stored `AccountCurrencyMap` entry pointing at a removed currency is silently ignored — the account pays in HDX (or WETH if EVM). `remove_currency` is therefore safe to run without resetting accounts first, but those users must hold HDX/WETH.
- Error-path classification: a permit counts as "executed" (nonce consumed) only on `EvmPermitCallExecutionError` or `EvmPermitNonceInvariantViolated`; everything else is treated as not executed.
- Per-user currency preference persists across sessions (stored in `AccountCurrencyMap`).
- `dispatch_permit` bridges EVM EIP-2612 permits into Substrate dispatch for gasless-style UX.
- **Never call `on_dispatch_permit_error()` on the signed branch.** The unsigned branch auto-pauses `dispatch_permit` on a bad permit because there is no cost to the submitter; on the signed branch the paymaster's per-attempt extrinsic fee is what replaces that defence.
- Signed-branch **revert** returns `Ok(e.post_info)`, not `Err`. The EVM already consumed the nonce and charged gas — returning `Err` would roll the nonce back and make the permit replayable.
- The fee-payer override is restored *after* `dispatch_permit` so the runner's post-execution refund (`correct_and_deposit_fee`) also routes to the paymaster; `restore_fee_payer` clears the override when there was no previous payer.
- On the signed branch `permit.from` is touched only by the inner call's effects — it is not charged.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
