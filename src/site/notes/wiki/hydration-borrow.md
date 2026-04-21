---
{"dg-publish":true,"permalink":"/wiki/hydration-borrow/","title":"Hydration Borrow","tags":["lending","borrowing","aave","defi","core-product"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"Hydration Borrow","tags":["lending","borrowing","aave","defi","core-product"],"source_count":1,"last_updated":"2026-04-13"}}
---


# Hydration Borrow

**TL;DR:** Hydration Borrow is [[wiki/hydration\|hydration]]'s lending and borrowing product — an Aave v3 fork adapted for Polkadot, running as EVM smart contracts on the Frontier EVM layer ([[wiki/pallet-frontier\|pallet-frontier]]), enabling users to supply assets for yield or borrow against collateral.

## Functionality

Users can supply assets and earn yield, or borrow against collateral. Supported collateral includes DOT, ETH, BTC variants, and stablecoins.

## Native Integration

Hydration Borrow integrates natively with the [[omnipool\|omnipool]] and [[wiki/hollar\|hollar]]:
- Users can mint [[wiki/hollar\|hollar]] by borrowing against deposited collateral
- The vision includes using Omnipool LP positions as collateral for borrowing, enabled by the protocol's vertical integration at the runtime level

## Risk Model

Key risk factors include Loan-to-Value (LTV) ratio, health factor, and liquidation thresholds. Partial automated liquidations at the start of each block restore health factors incrementally rather than triggering full liquidation.

## Repository

The codebase lives at `galacticcouncil/aave-v3-deploy` — EVM smart contracts and deployment configurations.

## Frontend Integration

The Hydration Borrow UI is implemented in [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] and integrated into the main app's [[wiki/hydration-ui-modules\|hydration-ui-modules]] `borrow/` module.

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]], [[wiki/hydration-ui-money-market\|hydration-ui-money-market]]
