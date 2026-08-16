---
{"dg-publish":true,"permalink":"/wiki/hydration-borrow/","title":"Hydration Borrow","tags":["lending","borrowing","aave","defi","core-product"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"Hydration Borrow","tags":["lending","borrowing","aave","defi","core-product"],"source_count":1,"last_updated":"2026-08-15"}}
---


# Hydration Borrow

**TL;DR:** Hydration Borrow is [[wiki/hydration\|hydration]]'s lending and borrowing product — an Aave v3 fork adapted for Polkadot, running as EVM smart contracts on the Frontier EVM layer ([[wiki/pallet-frontier\|pallet-frontier]]), enabling users to supply assets for yield or borrow against collateral.

## Functionality

Users can supply assets and earn yield, or borrow against collateral. Supported collateral includes DOT, ETH, BTC variants, and stablecoins.

## Three markets (Aug 2026)

There is no longer a single pool. `CustomMarket` (`packages/money-market/src/ui-config/marketsConfig.ts`) defines three live Aave v3 deployments, each with its own `POOL_ADDRESSES_PROVIDER`:

| Market | Title | Role |
|---|---|---|
| `hydration_v3` | Hydration | the main money market (default) |
| `gigahdx_v3` | GIGAHDX | isolated pool where **GIGAHDX is the collateral** and [[wiki/hollar\|hollar]] the debt asset — see [[wiki/gigahdx\|gigahdx]] |
| `bil_v3` | BIL | isolated pool for the BIL RWA vault strategy |

(plus `hydration_testnet_v3`.) **Breaking:** `MoneyMarketProvider` now takes `market: CustomMarket` instead of `env: MoneyMarketEnv` ([[wiki/hydration-ui-money-market\|hydration-ui-money-market]]).

## Native Integration

Hydration Borrow integrates natively with the [[omnipool\|omnipool]] and [[wiki/hollar\|hollar]]:
- Users can mint [[wiki/hollar\|hollar]] by borrowing against deposited collateral
- The vision includes using Omnipool LP positions as collateral for borrowing, enabled by the protocol's vertical integration at the runtime level

## Risk Model

Key risk factors include Loan-to-Value (LTV) ratio, health factor, and liquidation thresholds. Partial automated liquidations at the start of each block restore health factors incrementally rather than triggering full liquidation.

[[wiki/pallet-liquidation\|pallet-liquidation]] gained two paths this cycle: `liquidate_with_pool` (unsigned, explicit pool) and **`liquidate_gigahdx`** — a protocol-funded liquidation of GIGAHDX-collateral positions where one liquidation account borrows HOLLAR from the main market, runs `liquidationCall` on the GIGAHDX pool with `receiveAToken=true`, and the pallet seizes the borrower's locked HDX pro rata (`realize_yield` → `snapshot_stake` → `on_pre_seize`). Off-chain execution moved to [[wiki/pepl-worker\|pepl-worker]] (multi-money-market).

## Repository

The codebase lives at `galacticcouncil/aave-v3-deploy` — EVM smart contracts and deployment configurations.

## Frontend Integration

The Hydration Borrow UI is implemented in [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] and integrated into the main app's [[wiki/hydration-ui-modules\|hydration-ui-modules]] `borrow/` module.

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]], [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
