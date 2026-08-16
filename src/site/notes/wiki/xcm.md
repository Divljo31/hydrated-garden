---
{"dg-publish":true,"permalink":"/wiki/xcm/","title":"XCM (Cross-Consensus Messaging)","tags":["cross-chain","polkadot","interoperability"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"XCM (Cross-Consensus Messaging)","tags":["cross-chain","polkadot","interoperability"],"source_count":1,"last_updated":"2026-08-15"}}
---


# XCM (Cross-Consensus Messaging)

**TL;DR:** XCM (Cross-Consensus Messaging) is [[wiki/polkadot\|polkadot]]'s native cross-chain communication protocol enabling parachain interoperability without external bridges, used by [[wiki/hydration\|hydration]] for cross-chain transfers and remote swaps.

## Relevance to Hydration

[[wiki/hydration\|hydration]] uses XCM for:
- **Cross-chain asset transfers** with [[wiki/asset-hub\|asset-hub]] and other parachains
- **Remote swaps** — composing XCM instructions so users swap and receive assets on a different chain in one UX flow (e.g., DOT on relay chain → USDT on Asset Hub)
- **DAO treasury management** — DAOs across Polkadot can manage treasury operations on Hydration using XCM, no multisigs required
- **Cross-chain assets** — supports vDOT (Bifrost), USDT, USDC, wrapped BTC, and other parachain tokens

## Bridges Beyond Polkadot

For connectivity outside the Polkadot ecosystem, [[wiki/hydration\|hydration]] uses:
- **[[wiki/snowbridge\|snowbridge]]** — trustless native bridge between Polkadot and Ethereum; V1 and V2 routes ship side by side
- **[[wiki/wormhole\|wormhole]] NTT** — direct Hydration ↔ Ethereum / Base / Solana / Sui. **No XCM hop:** the old MRL model that routed through Moonbeam via XCM `Transact` is gone, and so is `AssetRoute.transact`
- **[[wiki/xc-swap\|xc-swap]]** — NEAR Intent Routing; a sibling of the XCM/bridge transfer stack, not a layer on it (one Hydration EVM tx, no transfer stack at all)

## Chain set (Aug 2026)

[[wiki/xc-cfg\|xc-cfg]] `chainsMap` **removed 5 chains** — `moonbeam`, `interlay`, `crust`, `ajuna`, `laos` — and **added `assethub_cex`**, an AssetHub variant used by the UI `onramp/` module for CEX forwarding (reads every asset incl. DOT from the `assets` pallet, reuses AssetHub's DEX via `dexAlias`).

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
