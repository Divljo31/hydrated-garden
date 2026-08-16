---
{"dg-publish":true,"permalink":"/wiki/pallet-liquidation/","title":"pallet-liquidation","tags":["lending","liquidation","money-market","gigahdx","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-liquidation","repo":"hydration-node","paths":["pallets/liquidation/src/lib.rs","pallets/liquidation/src/traits.rs","pallets/liquidation/src/weights.rs"],"symbols":["Pallet","Config","liquidate","liquidate_with_pool","set_borrowing_contract","do_liquidate","liquidate_gigahdx","BorrowingContract","GigaHdxSupport","Function","encode_borrow_call_data","encode_repay_call_data","encode_liquidation_call_data","MAX_UNSIGNED_LIQUIDATION_PRIORITY"],"traits_impl":["ValidateUnsigned"],"depends_on":["pallet-route-executor","pallet-evm","pallet-currencies","pallet-gigahdx","pallet-gigahdx-rewards"],"runtime_index":76,"tags":["lending","liquidation","money-market","gigahdx","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
---


# pallet-liquidation

**TL;DR:** Liquidator adapter for [[wiki/hydration-borrow\|hydration-borrow]] (Aave-fork money market on Hydration EVM). Calls the pool's `liquidationCall` via EVM and routes the received collateral through [[wiki/pallet-route-executor\|pallet-route-executor]] for profit. Now multi-money-market aware (`liquidate_with_pool`) and carries a **protocol-funded [[wiki/gigahdx\|gigahdx]] liquidation path** for GIGAHDX-collateral positions. Runtime index = 76.

## Role

Bridges Substrate-native liquidation bots to the EVM lending protocol. Three dispatch paths, all funnelled through `do_liquidate`:

| Condition | Path |
|---|---|
| `collateral_asset == GigaHdx::gigahdx_asset_id()` | **gigahdx** — `liquidate_gigahdx` (protocol-funded, `route` unused) |
| `debt_asset == HollarId` | HOLLAR flash-mint (`Function::FlashLoan` on `FlashMinter`) |
| otherwise | mint debt asset into the pallet account → `liquidate_position_internal` → router sell → burn |

## Config trait (excerpt)

```rust
// pallets/liquidation/src/lib.rs
pub trait Config: frame_system::Config {
    type Currency: Mutate<Self::AccountId, AssetId = AssetId, Balance = Balance>;
    type Evm: EVM<CallResult>;
    type Router: RouteProvider<AssetId>
        + RouterT<Self::RuntimeOrigin, AssetId, Balance, Trade<AssetId>, AmountInAndOut<Balance>>;
    type EvmAccounts: InspectEvmAccounts<Self::AccountId>;
    type Erc20Mapping: Erc20Mapping<AssetId>;
    type GasWeightMapping: GasWeightMapping;
    #[pallet::constant] type GasLimit: Get<u64>;
    #[pallet::constant] type ProfitReceiver: Get<Self::AccountId>;
    type RouterWeightInfo: AmmTradeWeights<Trade<AssetId>>;
    type WeightInfo: WeightInfo;
    #[pallet::constant] type HollarId: Get<AssetId>;
    type FlashMinter: Get<Option<(EvmAddress, EvmAddress)>>;
    type EvmErrorDecoder: Convert<CallResult, DispatchError>;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    /// Single integration seam for the protocol-funded gigahdx liquidation path.
    type GigaHdx: crate::traits::GigaHdxSupport<Self::AccountId>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `BorrowingContract` | StorageValue | `EvmAddress`, `ValueQuery` with `DefaultBorrowingContract` = `0x1b02E051683b5cfaC5929C25E84adb26ECf87B38` |

## Events

`Liquidated { user, collateral_asset, debt_asset, profit }`, `GigaHdxLiquidated { user, debt_repaid, hdx_seized, gigahdx_seized }`.

## Errors

`AssetConversionFailed`, `LiquidationCallFailed`, `InvalidRoute`, `NotProfitable`, `FlashMinterNotSet`, `InvalidLiquidationData`, plus the gigahdx set: `UnsupportedDebtAsset`, `NoGigaHdxPosition`, `RealizeYieldFailed`, `LiquidationAccountNotBound`, `ClearVotingLocksFailed`, `BorrowFailed`, `SeizeFailed`, `GigaHdxPoolNotSet`, `NoPoolDebt`, `RepayFailed`, `PoolAddressMismatch`.

## Extrinsics

| Name | Index | Origin | Description |
|------|-------|--------|-------------|
| `liquidate` | 0 | **any** (origin ignored) | Liquidate a MM position. Permissionless and caller-supplied `route` are intentional — see gotchas. |
| `set_borrowing_contract` | 1 | `AuthorityOrigin` | Set/update the money-market pool address |
| `liquidate_with_pool` | 2 | **none** (unsigned, local only) | Same body as `liquidate` plus an explicit `pool` consistency assertion and an `unsigned_priority` hint |

## Hooks

None. `ValidateUnsigned` gates `liquidate` and `liquidate_with_pool`:

```rust
// pallets/liquidation/src/lib.rs
pub const MAX_UNSIGNED_LIQUIDATION_PRIORITY: Priority = u64::MAX - 2;
const BASE_UNSIGNED_LIQUIDATION_PRIORITY: Priority = MAX_UNSIGNED_LIQUIDATION_PRIORITY - 10_000_000;

fn valid_tx(provides: impl Encode, priority: Priority) -> TransactionValidity {
    ValidTransaction::with_tag_prefix("liquidate_unsigned")
        .priority(BASE_UNSIGNED_LIQUIDATION_PRIORITY
            .saturating_add(priority)
            .min(MAX_UNSIGNED_LIQUIDATION_PRIORITY))
        .and_provides(provides)
        .longevity(1).propagate(false).build()
}
// liquidate           -> valid_tx(user, 0)                    (legacy, byte-identical)
// liquidate_with_pool -> valid_tx((user, pool), unsigned_priority.unwrap_or(0))
```

`unsigned_priority` is meant to carry *collateral at risk* (max `10_000_000.0` BASE). Signed user tx priority is capped by the SDK fork at `u64::MAX - 1_000_000_000`, which is below `BASE_UNSIGNED_LIQUIDATION_PRIORITY` — users cannot frontrun a liquidation with a tip.

## GigaHDX liquidation (`liquidate_gigahdx`)

Protocol-funded path for positions whose collateral is the GIGAHDX aToken. See [[wiki/pallet-gigahdx\|pallet-gigahdx]] / [[wiki/gigahdx\|gigahdx]] for the staking side; from the liquidation side the sequence is:

1. `GigaHdx::realize_yield(borrower)` — fold accrued yield into locked stake. **Best effort**: a `GigapotInsufficient` failure logs and continues (liquidation outranks the yield fold; the cost is a smaller `seize_hdx`).
2. `GigaHdx::snapshot_stake(borrower)` → `(orig_hdx, orig_gigahdx)`; require `orig_gigahdx > 0` else `NoGigaHdxPosition`.
3. `GigaHdx::on_pre_seize(borrower)` — zeroes `Stakes[borrower].gigahdx` so the lock-manager precompile lets Aave's internal aToken transfer through.
4. Assert the liquidation account is bound to its EVM address (`EvmAccounts::account_id(liq_evm) == liq_account`), else `LiquidationAccountNotBound` — otherwise the seized aToken lands in a different account than the ledger update.
5. Clamp: `capped = min(debt_to_cover, GigaHdx::borrower_pool_debt(borrower, debt_asset))`; `capped > 0` else `NoPoolDebt`.
6. `Function::Borrow` on the **main** borrowing pool as `liq_evm` — the liquidation account borrows `capped` HOLLAR (variable rate, `interestRateMode = 2`).
7. `Function::LiquidationCall` on the **GIGAHDX** pool with the *underlying* `sthdx_asset_id()` as collateral and `receiveAToken = true`, delivering the seized aToken straight to `liq_account`. Balance deltas measure `actual_seized_atoken` and `consumed` HOLLAR.
8. `seize_hdx = orig_hdx * actual_seized_atoken / orig_gigahdx` (rounding **down**; residue stays with the borrower).
9. `GigaHdx::clear_conflicting_votes(borrower, residual_hdx)` — drops conviction votes no longer backed by stake (also drops the matching `UserVoteRecord`).
10. `GigaHdx::on_seize(borrower, liq_account, seize_hdx, actual_seized_atoken, orig_gigahdx)` — reconciles the gigahdx ledger and refreshes locks.
11. `Function::Repay` of `surplus = capped - consumed` back to the borrowing pool, so the protocol carries debt only for what actually cleared.

Whole function is `#[frame_support::transactional]`.

### `GigaHdxSupport` — the integration seam

```rust
// pallets/liquidation/src/traits.rs
pub trait GigaHdxSupport<AccountId>: hydradx_traits::gigahdx::Seize<AccountId> {
    fn gigahdx_asset_id() -> AssetId;
    fn sthdx_asset_id() -> AssetId;
    fn liquidation_account() -> AccountId;
    fn pool_contract() -> Option<EvmAddress>;   // read live from pallet_gigahdx storage
    fn borrower_pool_debt(borrower: &AccountId, debt_asset: AssetId) -> Result<Balance, DispatchError>;
    fn clear_conflicting_votes(borrower: &AccountId, max_remaining_hdx: Balance) -> Result<u32, DispatchError>;
    fn clear_weight_for(user: EvmAddress) -> Weight;
}
```

Bundles six knobs into one `Config` item; the runtime wires it to `pallet_gigahdx` / `pallet_gigahdx_rewards` via an adapter. The supertrait `Seize` supplies `realize_yield`, `snapshot_stake`, `on_pre_seize`, `on_seize`, `seize_weight`.

## Integration

- **Traits implemented:** `ValidateUnsigned`
- **Traits consumed:** `EVM`, `InspectEvmAccounts`, `Erc20Mapping`, `RouterT`/`RouteProvider` ([[wiki/pallet-route-executor\|pallet-route-executor]]), `fungibles::Mutate`, `GigaHdxSupport` + `Seize`
- **Pallets depended on:** [[wiki/pallet-route-executor\|pallet-route-executor]], [[wiki/pallet-frontier\|pallet-frontier]], [[wiki/pallet-evm-accounts\|pallet-evm-accounts]], [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]

## EVM selectors (`Function`)

| Variant | Signature |
|---|---|
| `LiquidationCall` | `liquidationCall(address,address,address,uint256,bool)` |
| `FlashLoan` | `flashLoan(address,address,uint256,bytes)` |
| `Borrow` | `borrow(address,uint256,uint256,uint16,address)` |
| `Repay` | `repay(address,uint256,uint256,address)` |

`encode_borrow_call_data` / `encode_repay_call_data` share `encode_pool_debt_call`; `borrow` carries an extra `referralCode = 0` word.

## Gotchas

- **Permissionless `liquidate` + caller-chosen `route` is by design, not a bug.** The call exists to keep the money market solvent; a caller may pick a route that leaves little or no profit for `ProfitReceiver`. Anyone can already call Aave's `liquidationCall` directly and keep the whole bonus. Reports framing the open origin or the route as fund redirection are explicitly out of scope (see the doc comment on `liquidate`).
- `liquidate_with_pool` is **not publicly dispatchable** — `ensure_none` plus `ValidateUnsigned` rejecting `TransactionSource::External` means only a collator's own liquidation worker can submit it. The `pool` argument is an assertion (`PoolAddressMismatch`), never a router.
- GIGAHDX routing is **unconditional on collateral**: the gigahdx reserve lists HOLLAR as its only borrowable asset, so the `debt_asset == HollarId` check inside `liquidate_gigahdx` is a fail-closed guard. A gigahdx position must never fall through to the generic path — the locked aToken needs the `on_pre_seize`/`on_seize` dance.
- Without the `borrower_pool_debt` clamp an attacker-chosen `debt_to_cover` would borrow unbounded HOLLAR onto the liquidation account while Aave only consumes the close-factor slice.
- Weight is two-branch: gigahdx branch = `liquidate()` + `seize_weight()` + exact `clear_weight_for(user)`; generic branch = `liquidate()` + `RouterWeightInfo::sell_weight(route)`. Both add **4×** `GasLimit` (borrow + liquidationCall + repay + view calls).
- `BorrowingContract` is `ValueQuery` with a hard-coded default address — it is never `None`.
- Liquidation penalty / bonus is configured on the EVM contract, not in this pallet.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hydration-borrow\|hydration-borrow]]
