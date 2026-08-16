---
{"dg-publish":true,"permalink":"/wiki/pallet-gigahdx/","title":"pallet-gigahdx","tags":["gigahdx","staking","liquid-staking","aave","evm","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-gigahdx","repo":"hydration-node","paths":["pallets/gigahdx/src/lib.rs","pallets/gigahdx/src/traits.rs","pallets/gigahdx/src/weights.rs","pallets/gigahdx/src/benchmarking.rs","traits/src/gigahdx.rs","runtime/hydradx/src/gigahdx.rs","precompiles/lock-manager/src/lib.rs"],"symbols":["Pallet","Config","StakeRecord","PendingUnstake","Stakes","TotalLocked","GigaHdxPoolContract","PendingUnstakes","giga_stake","giga_unstake","unlock","cancel_unstake","migrate","realize_yield","set_pool_contract","do_stake","do_unstake","do_realize_yield","refresh_lock","exchange_rate","total_staked_hdx","total_gigahdx_supply","locked_gigahdx","gigapot_account_id","calculate_gigahdx_given_hdx_amount","calculate_hdx_amount_given_gigahdx"],"traits_impl":["Seize","ExternalClaims","LegacyStakeMigrator","VotingCommitmentInspect","MoneyMarketOperations"],"depends_on":["pallet-staking","pallet-gigahdx-rewards","pallet-liquidation","pallet-evm-accounts","pallet-conviction-voting","pallet-asset-registry"],"runtime_index":86,"tags":["gigahdx","staking","liquid-staking","aave","evm","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
---


# pallet-gigahdx

**TL;DR:** Liquid-staking primitive for [[wiki/hdx\|hdx]] on top of an EVM money market (AAVE V3 fork). `giga_stake` locks HDX *in the user's own account*, mints stHDX (asset 670), supplies it to the money market, and the MM mints GIGAHDX aTokens (asset 67) to the user. Value accrues by HDX flowing into the "gigapot" account, which lifts the HDX↔GIGAHDX exchange rate. Runtime index = 86. See [[wiki/gigahdx\|gigahdx]] for the protocol-level view.

## Role

- Successor to the legacy NFT-based [[wiki/pallet-staking\|pallet-staking]]. Stake stays **voteable** (`LockableCurrency::max` lock semantics let `pyconvot` overlap `ghdxlock`) and **usable as collateral** (GIGAHDX is a listed AAVE reserve, so stakers can borrow HOLLAR against their stake).
- No per-position reward accounting: yield is a pure exchange-rate mechanism. The gigapot receives 15% of trade fees via `GigaHdxFeeReceiver` (`runtime/hydradx/src/assets.rs`).
- Governance rewards are a separate concern, owned by [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]].
- Provides the seize hooks for the protocol-funded gigahdx liquidation path in [[wiki/pallet-liquidation\|pallet-liquidation]].

## Token / account map

| Thing | Value | Where |
|---|---|---|
| stHDX (underlying, MM reserve) | `AssetId 670` | `StHdxAssetId` in `runtime/hydradx/src/assets.rs` |
| GIGAHDX (aToken) | `AssetId 67` | `GigaHdxAssetIdConst` |
| Lock id on user HDX | `*b"ghdxlock"` | `GigaHdxLockId` |
| Gigapot (yield holder) | `PalletId(*b"gigahdx!")` → `gigapot_account_id()` | `GigaHdxPalletId` |
| `MinStake` | `1 UNITS` (1 HDX) | `GigaHdxMinStake` |
| `CooldownPeriod` | `28 * DAYS` | `GigaHdxCooldownPeriod` |
| `MaxPendingUnstakes` | `10` | `GigaHdxMaxPendingUnstakes` |
| Lock-manager precompile | `0x0806` | `precompiles/lock-manager/src/lib.rs` |

## Config trait (excerpt)

```rust
// pallets/gigahdx/src/lib.rs
#[pallet::config]
pub trait Config: frame_system::Config<RuntimeEvent: From<Event<Self>>> {
    type NativeCurrency: LockableCurrency<Self::AccountId, Balance = Balance, Moment = BlockNumberFor<Self>>
        + ReservableCurrency<Self::AccountId, Balance = Balance>;
    type MultiCurrency: fungibles::Mutate<Self::AccountId, AssetId = AssetId, Balance = Balance>
        + fungibles::Inspect<Self::AccountId, AssetId = AssetId, Balance = Balance>;
    #[pallet::constant] type StHdxAssetId: Get<AssetId>;
    type MoneyMarket: MoneyMarketOperations<Self::AccountId, AssetId, Balance>;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    #[pallet::constant] type PalletId: Get<PalletId>;          // gigapot
    #[pallet::constant] type LockId: Get<LockIdentifier>;      // ghdxlock
    #[pallet::constant] type MinStake: Get<Balance>;
    #[pallet::constant] type CooldownPeriod: Get<BlockNumberFor<Self>>;
    #[pallet::constant] type MaxPendingUnstakes: Get<u32>;
    type ExternalClaims: crate::traits::ExternalClaims<Self::AccountId>;
    type LegacyStaking: crate::traits::LegacyStakeMigrator<Self::AccountId>;
    type VotingCommitment: crate::traits::VotingCommitmentInspect<Self::AccountId>;
    type WeightInfo: WeightInfo;
}
```

## Key types

```rust
// pallets/gigahdx/src/lib.rs
pub struct StakeRecord {
    /// HDX locked in the user's own account under `Config::LockId` (active principal).
    pub hdx: Balance,
    /// aToken (GIGAHDX) units this stake backs — the value *returned* by
    /// `MoneyMarketOperations::supply`, so it matches the on-EVM aToken balance.
    pub gigahdx: Balance,
    pub unstaking: Balance,      // sum over pending positions
    pub unstaking_count: u16,
}

pub struct PendingUnstake { pub amount: Balance }
```

## Storage

| Name | Kind | Key → Value | Notes |
|---|---|---|---|
| `Stakes` | StorageMap | `AccountId → StakeRecord` | `OptionQuery`; reaped when `is_empty()` |
| `TotalLocked` | StorageValue | `Balance` | Σ `Stakes[a].hdx` |
| `GigaHdxPoolContract` | StorageValue | `EvmAddress` (Option) | AAVE V3 Pool; no default, must be set |
| `PendingUnstakes` | StorageDoubleMap | `(AccountId, BlockNumber) → PendingUnstake` | key = originating block = `position_id`; same-block unstakes compound |

`STORAGE_VERSION = 1`.

## Extrinsics

| # | Name | Origin | Description |
|---|---|---|---|
| 0 | `giga_stake(amount)` | Signed | Admission check → lock HDX under `ghdxlock` → mint stHDX at current rate → `MoneyMarket::supply` → record actual aTokens minted. Emits `Staked`. |
| 1 | `giga_unstake(gigahdx_amount)` | Signed | Pre-decrements `Stakes.gigahdx` (so the aToken lock check passes) → `MoneyMarket::withdraw` → burn stHDX → open a pending position for `payout = rate × gigahdx_amount`. Returns `DispatchResultWithPostInfo` (refunds the unused vote-scan weight). Emits `Unstaked`. |
| 2 | `set_pool_contract(contract)` | `AuthorityOrigin` (`Root \| TechCommitteeMajority`) | Sets the AAVE Pool address. Refuses with `OutstandingStake` unless `total_gigahdx_supply() == 0`. Emits `PoolContractUpdated`. |
| 3 | `unlock(position_id)` | Signed | Releases one pending position after `CooldownPeriod`; shrinks the lock. Emits `Unlocked`. |
| 4 | `cancel_unstake(position_id)` | Signed | Folds a pending position back into active stake at *today's* rate (re-mints aTokens). No cooldown gate. Emits `UnstakeCancelled`. |
| 5 | `migrate()` | Signed | `LegacyStaking::force_unstake` (100% of legacy rewards, no sigmoid slash) then re-stakes the whole freed amount. Whole-position only. Emits `MigratedFromLegacy`. |
| 6 | `realize_yield()` | Signed | Moves `rate × gigahdx − Stakes.hdx` from the gigapot into the caller's locked principal. GIGAHDX balance and rate unchanged. Emits `YieldRealized`. |

## Exchange-rate math

```rust
// pallets/gigahdx/src/lib.rs
pub fn total_staked_hdx() -> Balance {
    TotalLocked::<T>::get().saturating_add(T::NativeCurrency::free_balance(&Self::gigapot_account_id()))
}
pub fn total_gigahdx_supply() -> Balance {          // live stHDX total_issuance
    <T::MultiCurrency as fungibles::Inspect<_>>::total_issuance(T::StHdxAssetId::get())
}
pub fn exchange_rate() -> Ratio {                   // Ratio { n: hdx, d: gigahdx }, floored at 1.0
    let supply = Self::total_gigahdx_supply();
    if supply == 0 { return Ratio::one(); }
    core::cmp::max(Ratio::new(Self::total_staked_hdx(), supply), Ratio::one())
}
```

- `calculate_gigahdx_given_hdx_amount(a) = a × rate.d / rate.n` (rounds down).
- `calculate_hdx_amount_given_gigahdx(g) = g × rate.n / rate.d` (rounds down).
- Rate is **floored at 1.0** — sub-1 is only reachable via privileged gigapot drains or migration bugs; the floor protects downstream AAVE oracle reads.
- Compare `Ratio`s with `cmp`/`partial_cmp`, not `==` (field-wise equality).

## Admission gate (`ensure_stakeable`)

```rust
// pallets/gigahdx/src/lib.rs
fn ensure_stakeable(who: &T::AccountId, amount: Balance) -> DispatchResult {
    ensure!(amount >= T::MinStake::get(), Error::<T>::BelowMinStake);
    ensure!(T::ExternalClaims::on(who) == 0, Error::<T>::BlockedByExternalLock);
    let stake = Stakes::<T>::get(who).unwrap_or_default();
    let own_claim = stake.hdx.saturating_add(stake.unstaking);
    let stakeable = T::NativeCurrency::free_balance(who).saturating_sub(own_claim);
    ensure!(stakeable >= amount, Error::<T>::InsufficientFreeBalance);
    Ok(())
}
```

Strict policy: **any** non-whitelisted lock blocks staking outright. The runtime's `HdxExternalClaims` (`runtime/hydradx/src/gigahdx.rs`) whitelists only `ghdxlock` (own) and `pyconvot` (conviction voting) — so a legacy `stk_stks` or `ormlvest` lock blocks `giga_stake`.

## Locking model

- One combined lock per account: `refresh_lock` sets `ghdxlock = Stakes.hdx + Stakes.unstaking` with `set_lock` (shrinks as well as grows); removed entirely at zero.
- FRAME locks overlay via `max()`, which is why `pyconvot` may coexist — the same HDX backs a stake *and* a conviction vote — and why every other lock is rejected.
- **Unstake freeze:** `do_unstake` requires `stake.hdx − payout >= committed`, where `committed` is `VotingCommitment::committed_with_count(who)` = the **max** (not the sum) of active per-referendum vote reservations, pulled lazily from [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]. Violation → `StakeFrozen`.

## EVM / aToken side

- The `LockableAToken.sol` contract calls the **lock-manager precompile at `0x0806`** (`getLockedBalance(token, account)`), which returns `Pallet::locked_gigahdx(who) = Stakes[who].gigahdx`. Since that equals the user's aToken balance while staked, `free = balanceOf − locked = 0` → user-initiated GIGAHDX transfers are blocked.
- `giga_unstake` pre-decrements `Stakes.gigahdx` *before* calling the MM so `aToken.burn` passes the `freeBalance` check; the whole dispatchable is `#[transactional]` so a revert rolls the pre-decrement back.
- The precompile answers only for the configured `GigaHdxATokenAddress`; any other token gets zero.
- stHDX invariants documented in `runtime/hydradx/src/assets.rs`: (1) mint/burn exclusive to this pallet (the rate denominator is global `total_issuance`); (2) stHDX must be **non-borrowable** on AAVE (zero borrow cap / IRM 0), otherwise `liquidityIndex > 1 RAY` breaks the `aToken : stHDX = 1 : 1` invariant and leaks unlocked aTokens past the lock manager.

## Unstake payout waterfall

`payout = rate × gigahdx_amount`, then:
1. `payout <= stake.hdx` → consumed from active principal only, `yield_share = 0`.
2. `payout > stake.hdx` → active principal drained to 0, remainder transferred from the gigapot as `yield_share`.
3. On a **full exit** (`new_gigahdx == 0`) any residual `hdx` above `committed` is unbacked rounding dust; it is folded into this position's payout so the record and lock reap cleanly at `unlock`.

## Errors

`BelowMinStake`, `InsufficientFreeBalance`, `BlockedByExternalLock`, `InsufficientStake`, `NoStake`, `ZeroAmount`, `StHdxMintFailed`, `MoneyMarketSupplyFailed`, `MoneyMarketWithdrawFailed`, `Overflow`, `CooldownNotElapsed`, `PendingUnstakeNotFound`, `TooManyPendingUnstakes`, `OutstandingStake`, `StakeFrozen`, `SeizeFailed`, `GigapotInsufficient`.

## Hooks / lifecycle

No `on_initialize` / `on_finalize`. All state transitions are extrinsic- or trait-driven:

| Entry point | Caller | Effect |
|---|---|---|
| `Pallet::do_stake` (`pub`) | `cancel_unstake`, `migrate`, `pallet-gigahdx-rewards::claim_rewards` | **No admission control** — trusted internal callers only |
| `Seize::realize_yield` | [[wiki/pallet-liquidation\|pallet-liquidation]] | Folds accrued yield before snapshotting; best-effort |
| `Seize::snapshot_stake` | [[wiki/pallet-liquidation\|pallet-liquidation]] | Reads `(hdx, gigahdx)` pre-mutation |
| `Seize::on_pre_seize` | [[wiki/pallet-liquidation\|pallet-liquidation]] | Zeroes `Stakes.gigahdx` so the precompile reports `locked = 0` and AAVE's internal aToken transfer succeeds |
| `Seize::on_seize` | [[wiki/pallet-liquidation\|pallet-liquidation]] | Moves `seize_hdx` borrower→recipient (clean transfer, falling back to `slash` + `slash_reserved` when a lock blocks it), restores `gigahdx = orig − seized`, refreshes both locks |

## Integration points

| Direction | Counterparty | Seam |
|---|---|---|
| calls | AAVE V3 fork (EVM) | `MoneyMarketOperations` → `AaveMoneyMarket` in `runtime/hydradx/src/gigahdx.rs` (`Pool.supply` / `Pool.withdraw`, `GAS_LIMIT = 500_000`, returns the *balance delta* not the requested amount) |
| calls | [[wiki/pallet-staking\|pallet-staking]] | `LegacyStakeMigrator` → `LegacyStakingMigrator` → `pallet_staking::force_unstake` |
| calls | [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] | `VotingCommitment = GigaHdxRewards` (`VotingCommitmentInspect`) |
| calls | `pallet-balances` locks | `ExternalClaims` → `HdxExternalClaims` |
| called by | [[wiki/pallet-liquidation\|pallet-liquidation]] | `hydradx_traits::gigahdx::Seize` via `GigaHdxLiquidationSupport` |
| called by | lock-manager precompile `0x0806` | `locked_gigahdx()` |
| called by | `pallet-fee-processor` | `GigaHdxFeeReceiver` sends **15%** of trade fees to `gigapot_account_id()` |
| uses | [[wiki/pallet-evm-accounts\|pallet-evm-accounts]] | resolves the staker's H160 for the MM calls |

Runtime wiring: `impl pallet_gigahdx::Config for Runtime` in `runtime/hydradx/src/assets.rs`; adapters in `runtime/hydradx/src/gigahdx.rs`; `GigaHdx: pallet_gigahdx = 86` in `runtime/hydradx/src/lib.rs`. The gigapot and both reward pots are on the `ExtendedDustRemovalWhitelist`.

## Gotchas

- `Stakes.gigahdx` stores the value **returned** by `supply`, not the requested amount — AAVE rounds scaled balances down and the stored value must equal the real aToken balance or the lock manager mis-reports.
- `set_pool_contract` checks `total_gigahdx_supply()` (stHDX issuance), **not** `TotalLocked` — a drained-principal user can still hold aTokens bound to the old pool.
- `do_stake` is `pub` and skips *all* admission checks; untrusted callers must replicate `giga_stake`'s gate.
- `realize_yield` can return `GigapotInsufficient` from cross-user floor-rounding; the tripwire `MAX_GIGAPOT_ROUNDING_SHORTFALL = 1_000_000` (1 µHDX) `debug_assert`s under test but returns the error in release.
- `position_id` is the **originating block number**, not a sequence — unstakes in the same block compound into one position.
- Behavioural coverage: `integration-tests/src/gigahdx.rs` (mainnet-snapshot tests against the deployed AAVE fork — staking, rate inflation, multi-position cooldowns, migration, liquidation e2e).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/gigahdx\|gigahdx]]
- [[wiki/hdx\|hdx]]
- [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]
- [[wiki/pallet-staking\|pallet-staking]]
- [[wiki/pallet-liquidation\|pallet-liquidation]]
- [[wiki/hydration-runtime\|hydration-runtime]]
