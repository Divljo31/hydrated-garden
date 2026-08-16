---
{"dg-publish":true,"permalink":"/wiki/hollar/","title":"HOLLAR","tags":["stablecoin","defi","lending","core-product"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"HOLLAR","tags":["stablecoin","defi","lending","core-product"],"source_count":1,"last_updated":"2026-08-15"}}
---


# HOLLAR

**TL;DR:** HOLLAR is [[wiki/hydration\|hydration]]'s native over-collateralized decentralized stablecoin pegged to $1, built on Aave's GHO framework and backed by DOT, ETH, BTC variants, and stablecoins.

## Collateral

Backed by DOT, ETH, BTC variants, USDT, and USDC. Users can mint HOLLAR by borrowing against collateral deposited in [[wiki/hydration-borrow\|hydration-borrow]].

## Stability Mechanism — HSM

The HOLLAR Stability Module (HSM) uses an asymmetric price support mechanism:
- **Caps upside:** users can mint at predictable rates
- **Defends downside:** intelligent buybacks when HOLLAR trades below $1

## Liquidation

Partial automated liquidations occur at the start of each block — restoring health factors incrementally rather than triggering full liquidation. This mechanism also applies to the broader [[wiki/hydration-borrow\|hydration-borrow]] money market.

## Parameters (at launch)

- Initial supply cap: 2,000,000 HOLLAR
- Initial borrow rate: 5% APR
- Revenues from minting flow back into yield strategies

## Integration

HOLLAR integrates with all three pillars of the [[wiki/hydration\|hydration]] protocol: trading (four dedicated [[wiki/stableswap\|stableswap]] pools outside the [[omnipool\|omnipool]]), lending, and staking. It introduces a new smart contract attack surface (HSM, liquidation engine) that extends [[wiki/hydration\|hydration]]'s security considerations.

## Stable bonds and strategies (Aug 2026)

The UI's new `strategies/` module ([[wiki/hydration-ui-modules\|hydration-ui-modules]]) added two HOLLAR products:

- **Hollar stable bonds** — fixed-yield bonds are **sold via [[wiki/otc-trading\|OTC]] offers, not minted by a bond pallet**: `strategies/stable-bonds/config/bonds.ts` pins `HOLLAR_BOND_25_08_26_ID` to `otcOfferIds: ["1488", "1489"]`, accepting USDT / USDC, fixed yield 1.725%. Buying is `pallet-otc` `fill_order` against those offers; holdings surface in the wallet's `MyBonds`.
- **BIL vault** — the RWA strategy supplies BIL as collateral and **borrows HOLLAR** against it, with instant redeem through a BIL/HOLLAR [[wiki/stableswap\|stableswap]] pool (id `10055`).

[[wiki/pallet-hsm\|pallet-hsm]] itself is unchanged this cycle apart from an arbitrage balance-snapshot ordering fix.

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]]
