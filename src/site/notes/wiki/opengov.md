---
{"dg-publish":true,"permalink":"/wiki/opengov/","title":"OpenGov","tags":["governance","polkadot","hdx"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"OpenGov","tags":["governance","polkadot","hdx"],"source_count":2,"last_updated":"2026-11-02"}}
---


# OpenGov

**TL;DR:** OpenGov is [[wiki/polkadot\|polkadot]]'s on-chain governance framework used by [[wiki/hydration\|hydration]] for all protocol decisions. [[wiki/hdx\|hdx]] holders vote on referenda via tiered tracks with no multisigs or council intermediaries.

## How It Works

All [[wiki/hdx\|hdx]] holders can vote on referenda. OpenGov uses tiered tracks (10, defined in `runtime/hydradx/src/governance/tracks.rs`: root, whitelisted_caller, referendum_canceller, referendum_killer, general_admin, treasurer, spender, tipper, omnipool_admin, economic_parameters), where the required approval threshold scales with the impact of the referendum — from small tips to Root-level chain upgrades. There are no multisigs or council intermediaries.

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

The founding team relinquished control at mainnet launch. The Technical Committee (`pallet_collective::Instance2`, index 25) holds the **fast emergency path**: TC-majority origins can flip [[wiki/tradability-flags\|tradability-flags]], set circuit-breaker limits and lockdowns, pause extrinsics, dispatch as the emergency-admin account, disable or redirect the ICE solver; **any single TC member** can `suspend_xcm` ([[wiki/pallet-parameters\|pallet-parameters]]) and force-remove finished conviction votes (`VoteRemovalOrigin`). Resuming XCM needs TC super-majority (2/3) or a referendum. Full list with origins: [[wiki/note-emergency-levers\|note-emergency-levers]].

## Gov v1

Only OpenGov exists. `pallet-democracy` (gov v1, index 19) was removed in the 2026-10 `govv1_cleanup`; the next runtime upgrade ships `RemoveDemocracy` to clear its storage. Do not look for `Democracy` calls, `DemocracyHooks`, or `governance/old.rs`.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
