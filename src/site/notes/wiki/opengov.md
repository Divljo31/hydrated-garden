---
{"dg-publish":true,"permalink":"/wiki/opengov/","title":"OpenGov","tags":["governance","polkadot","hdx"],"dg-note-properties":{"type":"concept","title":"OpenGov","tags":["governance","polkadot","hdx"],"source_count":2,"last_updated":"2026-04-13"}}
---


# OpenGov

**TL;DR:** OpenGov is [[wiki/polkadot\|polkadot]]'s on-chain governance framework used by [[wiki/hydration\|hydration]] for all protocol decisions. [[wiki/hdx\|hdx]] holders vote on referenda via tiered tracks with no multisigs or council intermediaries.

## How It Works

All [[wiki/hdx\|hdx]] holders can vote on referenda. OpenGov uses tiered tracks (9 tracks), where the required approval threshold scales with the impact of the referendum — from small tips to Root-level chain upgrades. There are no multisigs or council intermediaries.

## What Is Governed

On [[wiki/hydration\|hydration]], OpenGov controls:
- [[omnipool\|omnipool]] asset listings (token listing process requires a referendum)
- Per-asset weight caps (`set_asset_weight_cap`)
- [[wiki/tradability-flags\|tradability-flags]] for any asset
- [[wiki/circuit-breaker\|circuit-breaker]] volume limits
- Protocol and asset fee rates
- Treasury management and [[wiki/protocol-owned-liquidity\|POL]] diversification

## Token Listing Process

1. Project submits an on-chain referendum proposing to list their asset in the [[omnipool\|omnipool]]
2. [[wiki/hdx\|hdx]] holders vote; approval requires meeting the threshold for the relevant track
3. If approved, initial liquidity is transferred to the pool account
4. `add_token` is called with the agreed initial price and weight cap
5. If rejected, `refund_refused_asset` returns the pre-transferred liquidity

## Technical Committee

The founding team relinquished control at mainnet launch. The Technical Committee retains only emergency powers — specifically, the ability to pause asset operations via [[wiki/tradability-flags\|tradability-flags]].

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
