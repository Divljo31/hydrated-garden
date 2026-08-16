---
{"dg-publish":true,"permalink":"/wiki/gigahdx/","title":"GigaHDX","tags":["gigahdx","hdx","staking","liquid-staking","tokenomics","governance","aave","evm"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"protocol","title":"GigaHDX","repo":"hydration-node","paths":["pallets/gigahdx/src/lib.rs","pallets/gigahdx-rewards/src/lib.rs","traits/src/gigahdx.rs","runtime/hydradx/src/gigahdx.rs","runtime/hydradx/src/assets.rs","precompiles/lock-manager/src/lib.rs","integration-tests/src/gigahdx.rs","integration-tests/src/gigahdx_rewards.rs"],"symbols":["giga_stake","giga_unstake","unlock","cancel_unstake","migrate","realize_yield","claim_rewards","exchange_rate","gigapot_account_id","MoneyMarketOperations","Seize","ClearConflictingVotes","VotingCommitmentInspect"],"tags":["gigahdx","hdx","staking","liquid-staking","tokenomics","governance","aave","evm"],"last_updated":"2026-08-15"}}
---


# GigaHDX

**TL;DR:** GigaHDX is Hydration's liquid-staking layer for [[wiki/hdx\|hdx]] and the successor to the legacy NFT staking system. Stake HDX → receive GIGAHDX, a yield-bearing aToken on Hydration's own AAVE V3 fork. The HDX never leaves your account (it is locked in place, so it stays voteable), yield accrues via a rising HDX↔GIGAHDX exchange rate, and the GIGAHDX itself is usable as money-market collateral.

## The three tokens

| Token | Asset id | What it is |
|---|---|---|
| HDX | `0` | The native token. Locked in the staker's own account under lock id `ghdxlock`. |
| stHDX | `670` | Internal receipt minted/burned **exclusively** by [[wiki/pallet-gigahdx\|pallet-gigahdx]]. Never held by users in practice — it is immediately supplied to the money market. Its `total_issuance` is the exchange-rate denominator. |
| GIGAHDX | `67` | The AAVE aToken minted against the stHDX supply. This is the user-facing liquid-staking token, and a listed reserve you can borrow HOLLAR against. |

## Stake flow

```
giga_stake(amount)
  ├─ admission: amount >= MinStake (1 HDX), ExternalClaims::on(who) == 0,
  │             free_balance - (hdx + unstaking) >= amount
  ├─ mint stHDX = amount × rate.d / rate.n   (to the staker)
  ├─ MoneyMarket::supply  →  AAVE Pool.supply  →  mints GIGAHDX aToken to the staker's H160
  ├─ Stakes[who].gigahdx += *actual* aToken delta;  Stakes[who].hdx += amount
  └─ refresh_lock: ghdxlock = hdx + unstaking   (HDX stays in the staker's account)
```

Unstaking is the reverse, plus a **28-day cooldown**: `giga_unstake` burns aTokens, opens a pending position keyed by the current block, and `unlock` releases it once `CooldownPeriod` elapses. `cancel_unstake` folds a pending position back into active stake at today's rate at any point before `unlock`.

## Economic mechanism

Yield is **not** per-position bookkeeping. It is a single global exchange rate:

```
total_staked_hdx = TotalLocked + free_balance(gigapot)
exchange_rate    = max(Ratio { n: total_staked_hdx, d: total_stHDX_issuance }, 1.0)
```

- The **gigapot** (`PalletId(*b"gigahdx!")`) receives **15% of all trade fees** (`GigaHdxFeeReceiver`). Every HDX that lands there lifts the numerator and therefore every staker's GIGAHDX redemption value pro rata. A plain transfer into the gigapot is the only side effect needed.
- The rate is **floored at 1.0** — it only ever rises under user flows, and the floor shields downstream AAVE oracle reads from privileged-drain artefacts.
- `realize_yield()` lets a staker materialise `rate × gigahdx − Stakes.hdx` from the gigapot into their *locked principal* without touching their GIGAHDX balance or the rate. It matters because pro-rata liquidation math and vote-freeze checks read `Stakes.hdx`.
- On unstake, payout above the recorded principal is drawn from the gigapot as `yield_share`.

## Governance rewards

Separate pot, separate pallet ([[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]): **25% of trade fees** flow to an accumulator pot. When a referendum completes, a track-specific slice of that pot (10% root, 8% whitelisted_caller, 5% treasurer/economic_parameters, 2% default) is allocated and split pro rata by `staked_vote × conviction_multiplier` (0× for `None` up to 8× for `Locked6x`). `claim_rewards` compounds the payout straight back into the caller's gigahdx position.

Net: HDX holders are paid for *staking* through the exchange rate, and paid again for *voting with conviction* through the reward pots. Non-voters still earn the rate; zero-conviction voters earn nothing extra.

## Why the HDX stays in your account

FRAME balance locks overlay via `max()`, not addition. Keeping the stake as a lock on the user's own account means `pyconvot` (conviction voting) and `ghdxlock` can share the same HDX — so staked HDX remains fully voteable, which was the whole point of the legacy staking design too. The cost is that any *other* lock must be rejected, otherwise the same HDX could back two claims. `HdxExternalClaims` therefore whitelists only `ghdxlock` and `pyconvot`; a legacy `stk_stks` or `ormlvest` lock makes `giga_stake` fail with `BlockedByExternalLock`. The reverse guard was added to [[wiki/pallet-staking\|pallet-staking]] too — `ghdxlock` blocks legacy staking.

To stop the overlap from becoming a stake→vote→unstake exploit, `giga_unstake` enforces `Stakes.hdx − payout >= committed`, where `committed` is the **max** of the user's active per-referendum vote reservations (they overlap, so max not sum), read lazily from `UserVoteRecords`. Violation → `StakeFrozen`. Symmetrically, `lock_balance_on_unsuccessful_vote` opts *losing* votes into the conviction lock so a max-conviction losing vote can't collect the boosted multiplier and exit on the cooldown alone.

## Why GIGAHDX can't be transferred

GIGAHDX is a `LockableAToken` on the EVM side. Its `freeBalance = balanceOf − locked` check calls the **lock-manager precompile at `0x0806`** (`precompiles/lock-manager/src/lib.rs`), which returns `pallet_gigahdx::Stakes[who].gigahdx`. While staked those are equal, so `free = 0` and any user-initiated ERC20 transfer reverts. `giga_unstake` pre-decrements `Stakes.gigahdx` before invoking the money market so the legitimate `Pool.withdraw → aToken.burn` path passes; the whole call is transactional so a revert rolls the pre-decrement back.

Two stHDX invariants (documented at `runtime/hydradx/src/assets.rs`) hold this together:
1. stHDX mint/burn is exclusive to [[wiki/pallet-gigahdx\|pallet-gigahdx]] — any external mint dilutes every staker, since the rate denominator reads global `total_issuance`.
2. stHDX must be **non-borrowable** on AAVE (zero borrow cap / IRM returning 0). If `liquidityIndex` drifts above 1 RAY the `aToken : stHDX = 1 : 1` invariant breaks and unlocked aTokens leak past the lock manager.

## Collateral and liquidation

GIGAHDX is a listed reserve, so a staker can borrow HOLLAR against their stake while still earning yield and voting. That makes liquidation possible, and it is **protocol-funded** — see [[wiki/pallet-liquidation\|pallet-liquidation]] `liquidate_gigahdx`:

1. `Seize::realize_yield` (best-effort — a gigapot shortfall must not block liquidation).
2. `snapshot_stake` → `(orig_hdx, orig_gigahdx)`.
3. `on_pre_seize` zeroes `Stakes.gigahdx` so the precompile reports `locked = 0` and AAVE's internal aToken transfer can land.
4. The liquidation account (the treasury) borrows HOLLAR from the *main* money market, clamped to the borrower's actual pool debt, and runs `liquidationCall` with `receiveAToken = true`.
5. `seize_hdx = orig_hdx × seized_atoken / orig_gigahdx` (rounded down — residue stays with the borrower).
6. `clear_conflicting_votes` force-removes conviction votes whose reservation exceeds the residual stake, then resyncs `pyconvot` via conviction-voting's `unlock`.
7. `on_seize` moves the HDX (clean transfer, falling back to `slash` + `slash_reserved` if a foreign lock blocks it), restores `gigahdx = orig − seized`, refreshes both locks.
8. Surplus borrowed HOLLAR is repaid.

Debt asset is restricted to HOLLAR (`UnsupportedDebtAsset` otherwise).

## Migration from legacy staking

`migrate()` is an all-or-nothing bridge from the legacy NFT position in [[wiki/pallet-staking\|pallet-staking]]:

- Calls `pallet_staking::force_unstake`, which pays out **100% of rewards** — no sigmoid `PayablePercentage` slash, no `UnclaimablePeriods` early-exit penalty (that is the migration incentive).
- Refuses while **any** registered conviction vote survives (`ExistingVotes`): votes must be removed while the legacy position still backs them, so conviction-voting applies the lock to winning *and* losing votes before the position is destroyed.
- Admission (`ensure_stakeable`) runs *after* `force_unstake`, so the now-released `stk_stks` lock no longer counts against `ExternalClaims`.
- The full `stake + accumulated_locked_rewards + paid_rewards` is re-staked into gigahdx. Emits `ForceUnstaked` (staking) and `MigratedFromLegacy` (gigahdx).

## Fee split context

`pallet_fee_processor::Config::FeeReceivers` in `runtime/hydradx/src/assets.rs`:

| Receiver | Share | Destination |
|---|---|---|
| `GigaHdxFeeReceiver` | 15% | gigapot → lifts the exchange rate |
| `GigaHdxRewardsFeeReceiver` | 25% | rewards accumulator pot |
| `StakingFeeReceiver` / `HdxStakingFeeReceiver` | 5% | legacy [[wiki/pallet-staking\|pallet-staking]] pot |
| `ReferralsFeeReceiver` | 5% | [[wiki/pallet-referrals\|pallet-referrals]] pot (raw asset) |

## Code map

| Concern | Path |
|---|---|
| Core pallet | `pallets/gigahdx/src/lib.rs` |
| Pallet-side runtime hooks | `pallets/gigahdx/src/traits.rs` (`ExternalClaims`, `LegacyStakeMigrator`, `VotingCommitmentInspect`) |
| Shared traits | `traits/src/gigahdx.rs` (`MoneyMarketOperations`, `Seize`, `ClearConflictingVotes`) |
| Rewards pallet | `pallets/gigahdx-rewards/src/lib.rs`, `voting_hooks.rs`, `types.rs` |
| Runtime adapters | `runtime/hydradx/src/gigahdx.rs` (`AaveMoneyMarket`, `TrackRewardConfig`, `RuntimeReferenda`, `HdxExternalClaims`, `GigaHdxVoteClearance`, `GigaHdxLiquidationSupport`, `LegacyStakingMigrator`, `LegacyStakingExternalClaims`) |
| Config + constants | `runtime/hydradx/src/assets.rs` |
| Voting hook tuple | `runtime/hydradx/src/governance/mod.rs` (`CombinedVotingHooks`) |
| Lock-manager precompile `0x0806` | `precompiles/lock-manager/src/lib.rs` |
| Liquidation path | `pallets/liquidation/src/lib.rs` → `liquidate_gigahdx` |
| Behavioural tests | `integration-tests/src/gigahdx.rs`, `integration-tests/src/gigahdx_rewards.rs` |

Runtime pallet indices: `GigaHdx = 86`, `GigaHdxRewards = 87`.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/pallet-gigahdx\|pallet-gigahdx]]
- [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]
- [[wiki/pallet-staking\|pallet-staking]]
- [[wiki/pallet-liquidation\|pallet-liquidation]]
- [[wiki/hdx\|hdx]]
- [[wiki/hydration-runtime\|hydration-runtime]]
