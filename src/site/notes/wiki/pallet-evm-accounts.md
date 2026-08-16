---
{"dg-publish":true,"permalink":"/wiki/pallet-evm-accounts/","title":"pallet-evm-accounts","tags":["evm","accounts","ntt","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-evm-accounts","repo":"hydration-node","paths":["pallets/evm-accounts/src/lib.rs"],"symbols":["Pallet","Config","bind_evm_address","add_contract_deployer","approve_contract","claim_account","set_ntt_minter","clear_ntt_minter","AccountExtension","ContractDeployer","ApprovedContract","MarkedEvmAccounts","Allowances","NttMinters","MESSAGE_PREFIX"],"traits_impl":["InspectEvmAccounts","ValidateUnsigned"],"depends_on":["pallet-asset-registry","pallet-transaction-multi-payment"],"runtime_index":93,"tags":["evm","accounts","ntt","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
---


# pallet-evm-accounts

**TL;DR:** Binds Substrate AccountId32 to Ethereum H160. Manages contract-deployer allowlist, approved-contract list, ERC-20-style allowances, and signature-based account claiming. Runtime index = 93.

## Role

Identity bridge between Substrate and EVM worlds on Hydration. Needed because:
1. Substrate uses 32-byte sr25519 accounts; EVM uses 20-byte addresses.
2. Deploying contracts requires allowlist (governance control over smart-contract permissioning).
3. Unsigned `claim_account` lets an Ethereum-signature holder prove ownership of a truncated Substrate account.

## Config trait (excerpt)

```rust
// pallets/evm-accounts/src/lib.rs
pub trait Config: frame_system::Config {
    type EvmNonceProvider: EvmNonceProvider;
    #[pallet::constant] type FeeMultiplier: Get<u32>;
    type ControllerOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    /// Origin that can clear an NTT minter. Meant to be faster than
    /// `ControllerOrigin` so it can act as an emergency stop for NTT mint/burn.
    type NttEmergencyOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type AssetId: Parameter + Member + Copy + ... + AtLeast32BitUnsigned;
    type Currency: Inspect<Self::AccountId, AssetId = Self::AssetId, Balance = Balance>;
    type ExistentialDeposits: GetByKey<Self::AssetId, Balance>;
    type FeeCurrency: AccountFeeCurrency<Self::AccountId, AssetId = Self::AssetId>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `AccountExtension` | StorageMap | `EvmAddress → AccountIdLast12Bytes` |
| `ContractDeployer` | StorageMap | `EvmAddress → ()` (whitelist) |
| `ApprovedContract` | StorageMap | `EvmAddress → ()` (balance-managing contracts) |
| `MarkedEvmAccounts` | StorageMap | `AccountId → ()` (permanent sufficient counter) |
| `Allowances` | StorageNMap | `(AssetId, owner, spender) → Balance` (ERC-20 style) |
| `NttMinters` | StorageMap | `AssetId → EvmAddress` — the **only** EVM address allowed to mint/burn that asset via the MultiCurrency precompile (NTT spoke manager) |

## Events

`Bound`, `DeployerAdded`, `DeployerRemoved`, `ContractApproved`, `ContractDisapproved`, `AccountClaimed`, `NttMinterSet { asset_id, minter }`, `NttMinterCleared { asset_id }`.

## Errors

`TruncatedAccountAlreadyUsed`, `AddressAlreadyBound`, `BoundAddressCannotBeUsed`, `AddressNotWhitelisted`, `InvalidSignature`, `AccountAlreadyExists`, `InsufficientAssetBalance`, `InvalidMinterAddress` (zero address rejected).

## Extrinsics

| Name | Description |
|------|-------------|
| `bind_evm_address` | Link caller's Substrate AccountId to their derived EVM address |
| `add_contract_deployer` | Whitelist EVM address for contract deployment (ControllerOrigin) |
| `remove_contract_deployer` | Revoke deployer permission |
| `renounce_contract_deployer` | Self-revoke deployer permission |
| `approve_contract` | Approve contract for protocol balance management |
| `disapprove_contract` | Revoke contract approval |
| `claim_account` | Unsigned — prove ownership of truncated account via Ethereum signature |
| `set_ntt_minter` | Call index 7. `ControllerOrigin`. Set the NTT spoke-manager address permitted to mint/burn `asset_id` via the MultiCurrency precompile. Rejects the zero address. |
| `clear_ntt_minter` | Call index 8. `NttEmergencyOrigin` (deliberately *faster* than `ControllerOrigin`). Emergency stop: removes the entry, halting all NTT mint/burn for that asset. |

## Hooks

`integrity_test` (size checks: EvmAddress = 20 bytes, AccountId = 32 bytes).

## Integration

- **Traits implemented:** `InspectEvmAccounts` (hydradx_traits), `ValidateUnsigned` (for `claim_account`)
- **Traits consumed:** `Inspect` (fungibles), `GetByKey` (ED provider), `AccountFeeCurrency`
- **Pallets depended on:** [[wiki/pallet-asset-registry\|pallet-asset-registry]], [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]]

## Gotchas

- EVM-derived Substrate account = `[0..20 bytes of H160][b"ETH\0"][00 * 8]` — 32 bytes total with fixed `"ETH\0"` marker.
- `MarkedEvmAccounts` permanently bumps the `sufficients` counter — marked accounts can never be reaped, even with zero balance.
- `claim_account` uses Ethereum personal-message format with prefix `MESSAGE_PREFIX = b"EVMAccounts::claim_account"`.
- Contract deployment goes through `ControllerOrigin` permission check in the frontier call path — bypassable only by whitelisted deployers.
- `Allowances` storage is used by the multi-currency precompile to implement ERC-20 `approve`/`transferFrom`.
- `FeeMultiplier` scales the weight for binding — prevents griefing.
- **NTT minter is per-asset and single-valued.** `set_ntt_minter` overwrites without a prior-value check; an unset asset simply cannot be minted/burned through the precompile. The asymmetric origins are intentional: setting is slow (`ControllerOrigin`), clearing is fast (`NttEmergencyOrigin`) so the mint path can be halted in one governance action.
- `set_ntt_minter` / `clear_ntt_minter` weights are hand-copied from `approve_contract` / `disapprove_contract` (same shape: one map write) rather than benchmarked.

## Runtime wiring

```rust
// runtime/hydradx/src/evm/mod.rs
impl pallet_evm_accounts::Config for Runtime {
    type Currency = FungibleCurrencies<Runtime>;
    type FeeCurrency = MultiTransactionPayment;
    // ...
}
```

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
