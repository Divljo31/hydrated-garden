---
{"dg-publish":true,"permalink":"/wiki/hdx/","title":"HDX","tags":["token","governance","staking","tokenomics","gigahdx"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"HDX","tags":["token","governance","staking","tokenomics","gigahdx"],"source_count":2,"last_updated":"2026-08-15"}}
---


# HDX

**TL;DR:** HDX is the native governance and incentive token of [[wiki/hydration\|hydration]], enabling on-chain voting via [[wiki/opengov\|opengov]] and staking for protocol revenue. Two staking systems coexist: the legacy NFT/bonding-curve system ([[wiki/pallet-staking\|pallet-staking]]) and its successor, the liquid-staking [[wiki/gigahdx\|gigahdx]] system ([[wiki/pallet-gigahdx\|pallet-gigahdx]]).

## Governance

HDX holders vote on all protocol decisions via [[wiki/opengov\|opengov]] — including [[omnipool\|omnipool]] asset listings, parameter changes (weight caps, fee rates, circuit breaker limits), and treasury management. There are no multisigs or council intermediaries.

## Staking

HDX can be staked to earn protocol revenue. Revenue sources include LP fees from Hydration's own HDX position in the [[omnipool\|omnipool]] and treasury subsidies.

### Legacy staking ([[wiki/pallet-staking\|pallet-staking]], runtime index 69)

NFT-position staking with a [[wiki/bonding-curve\|bonding-curve]] model:
- Early claimers receive a fraction of accumulated rewards
- The remainder redistributes to other stakers
- Governance participation (voting in referenda) accelerates the bonding curve, rewarding active governance participants

Still live, but on the way out: it now receives only 5% of trade fees, refuses to admit HDX carrying a `ghdxlock` lock, and exposes `force_unstake` purely so holders can migrate out to gigahdx without forfeiting rewards.

### GigaHDX liquid staking ([[wiki/gigahdx\|gigahdx]], [[wiki/pallet-gigahdx\|pallet-gigahdx]], runtime index 86)

The successor system. `giga_stake` locks HDX **in the staker's own account** (lock id `ghdxlock`, so it stays voteable), mints internal stHDX (asset 670), and supplies it to Hydration's AAVE V3 fork, which mints the yield-bearing **GIGAHDX** aToken (asset 67) to the staker.

| Property | Legacy staking | GigaHDX |
|---|---|---|
| Position | Non-transferable NFT | GIGAHDX aToken (non-transferable, lock-manager enforced) |
| Yield | Per-position RPS + sigmoid payable % | Global exchange rate: `(TotalLocked + gigapot) / stHDX issuance`, floored at 1.0 |
| Fee share | 5% | 15% to the gigapot + 25% to the governance rewards pot |
| Governance rewards | Action points | Pro-rata per referendum, `staked_vote × conviction multiplier` ([[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]) |
| Exit | `unstake` (forfeits unclaimed rewards) | `giga_unstake` → 28-day cooldown → `unlock`; cancellable |
| Collateral use | None | GIGAHDX is a listed money-market reserve — borrow HOLLAR against your stake |

Because FRAME locks overlay via `max()`, `ghdxlock` and `pyconvot` can share the same HDX, so staked HDX remains fully voteable — but every other lock is rejected (`BlockedByExternalLock`), and `giga_unstake` is frozen below the HDX backing the staker's active votes.

## Protocol Revenue Flow

Protocol fees from the [[omnipool\|omnipool]] are collected in [[wiki/lrna\|lrna]] and used for continuous HDX buybacks, which are distributed to stakers.

Current trade-fee split (`pallet_fee_processor::Config::FeeReceivers`, `runtime/hydradx/src/assets.rs`): 15% gigapot ([[wiki/gigahdx\|gigahdx]] exchange rate), 25% gigahdx governance-rewards accumulator, 5% legacy staking pot, 5% [[wiki/referrals\|referrals]].

## Referrals

The [[wiki/referrals\|referral system]] redirects 50% of asset fees (that would otherwise go to HDX stakers) to referral rewards, incentivizing user growth.

## Bonds

Users can purchase HDX bonds to grow [[wiki/protocol-owned-liquidity\|POL]] in exchange for fixed-rate yield over a set period.

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/gigahdx\|gigahdx]]
- [[wiki/pallet-gigahdx\|pallet-gigahdx]]
- [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]
- [[wiki/pallet-staking\|pallet-staking]]
