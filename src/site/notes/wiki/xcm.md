---
{"dg-publish":true,"permalink":"/wiki/xcm/","title":"XCM (Cross-Consensus Messaging)","tags":["cross-chain","polkadot","interoperability"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"XCM (Cross-Consensus Messaging)","tags":["cross-chain","polkadot","interoperability"],"source_count":1,"last_updated":"2026-11-02"}}
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
- **[[wiki/wormhole\|wormhole]] NTT** — direct Hydration ↔ Ethereum / Base / HyperEVM / Robinhood / Solana / Sui / NEAR (Hydration EVM, no XCM hop)
- **Omni (NEAR One)** — Zcash corridor via NEAR
- **[[wiki/xc-swap\|xc-swap]]** — Hydration → NEAR Intents swaps over the NTT rail (one Hydration EVM tx)

## Chain set

Substrate chains in [[wiki/xc-cfg\|xc-cfg]] `chainsMap` (d16ff85): `hydration`, `polkadot`, `assethub`, `assethub_cex` (CEX forwarding for the UI `onramp/` module; reuses AssetHub's DEX via `dexAlias`), `astar`, `bifrost`, `energywebx`, `mythos`, `neuroweb`, `pendulum`, `unique`; Kusama `assethub_kusama`, `basilisk`. Full list incl. EVM / Solana / Sui / NEAR / Zcash: [[wiki/xc-cfg\|xc-cfg]].

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
