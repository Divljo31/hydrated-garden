---
{"dg-publish":true,"permalink":"/wiki/pallet-duster/","title":"pallet-duster","tags":["dust","ed","treasury","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-duster","repo":"hydration-node","paths":["pallets/duster/src/lib.rs"],"symbols":["Pallet","Config","dust_account","whitelist_account","remove_from_whitelist","AccountWhitelist","DusterWhitelist"],"traits_impl":["OnDust","Contains","DustRemovalAccountWhitelist"],"depends_on":["pallet-asset-registry","pallet-currencies"],"runtime_index":61,"tags":["dust","ed","treasury","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-duster

**TL;DR:** Sweeps sub-ED balances to the Treasury. Anyone can dust any non-whitelisted account whose balance in a given asset is below that asset's `ExistentialDeposit`; the dusting caller's tx fee is refunded on success. Runtime index = 61.

## Role

Keeps chain storage clean by removing accounts holding economically-meaningless balances while preserving protocol-owned accounts (pool accounts, treasury, LM accounts) via a whitelist. Also handles special ERC-20 "AToken" (Aave aToken) dust, which must be withdrawn to the underlying asset rather than transferred.

## Config trait (excerpt)

```rust
// pallets/duster/src/lib.rs
pub trait Config: frame_system::Config {
    type AssetId: Parameter + Member + Copy + MaybeSerializeDeserialize + Ord;
    type MultiCurrency: Inspect<Self::AccountId, AssetId = Self::AssetId, Balance = Balance>
        + Mutate<Self::AccountId>;
    type ExistentialDeposit: GetByKey<Self::AssetId, Balance>;
    type WhitelistUpdateOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type Erc20Support: Erc20Inspect<Self::AssetId>
        + Erc20OnDust<Self::AccountId, Self::AssetId>;
    type ExtendedWhitelist: Get<Vec<Self::AccountId>>;
    #[pallet::constant] type TreasuryAccountId: Get<Self::AccountId>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `AccountWhitelist` | StorageMap | `AccountId → ()` (dynamic whitelist, genesis-configurable) |

## Events

`Dusted` (who, amount), `Added`, `Removed`.

## Errors

`AccountWhitelisted`, `AccountNotWhitelisted`, `ZeroBalance`, `NonZeroBalance`, `BalanceSufficient`, `ReserveAccountNotSet`.

## Extrinsics

| Name | Description |
|------|-------------|
| `dust_account` | Dust `(account, currency_id)` if balance < ED; caller is refunded tx fee |
| `whitelist_account` | Add account to dynamic whitelist (WhitelistUpdateOrigin) |
| `remove_from_whitelist` | Remove account from whitelist |

## Hooks

None.

## Integration

- **Traits implemented:** `OnDust<AccountId, AssetId, Balance>` (invoked by `orml_tokens` via `OnDust` hook), `Contains<AccountId>` (whitelist predicate), `DustRemovalAccountWhitelist<AccountId>` (API for other pallets to add/remove)
- **Traits consumed:** `Inspect` + `Mutate` (fungibles), `GetByKey` (ED provider), `Erc20Inspect`, `Erc20OnDust`
- **Pallets depended on:** [[wiki/pallet-asset-registry\|pallet-asset-registry]], [[wiki/pallet-currencies\|pallet-currencies]], [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]] (uses `DustRemovalAccountWhitelist` to whitelist farm accounts)

## Gotchas

- Dust is transferred to `TreasuryAccountId`, never burnt.
- AToken-backed assets call `Erc20OnDust::on_dust` which withdraws from Aave to the underlying, then transfers — bypasses the ERC-20 path that would lose liquidity.
- `ExtendedWhitelist` is a runtime-level compile-time list (e.g. Treasury itself) — cannot be removed via extrinsic.
- `Pays::No` on successful dust — incentivizes users to clean up sub-ED accounts.
- Two whitelist layers: dynamic `AccountWhitelist` (governance) + static `ExtendedWhitelist` (runtime); both checked via `Contains<AccountId>`.
- Dusting the Treasury itself is blocked.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
