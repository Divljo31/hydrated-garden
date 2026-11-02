---
{"dg-publish":true,"permalink":"/wiki/snowbridge/","title":"Snowbridge","tags":["bridge","cross-chain","ethereum","polkadot"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"Snowbridge","tags":["bridge","cross-chain","ethereum","polkadot"],"source_count":1,"last_updated":"2026-11-02"}}
---


# Snowbridge

**TL;DR:** Snowbridge is a trustless native bridge between [[wiki/polkadot\|polkadot]] and Ethereum with no additional trust assumptions. [[wiki/hydration\|hydration]] uses it alongside [[wiki/wormhole\|wormhole]] NTT for Ethereum connectivity, while [[wiki/xcm\|xcm]] handles Polkadot interoperability.

## V1 and V2 side by side

The SDK ships both generations ([[wiki/xc-cfg\|xc-cfg]] `configs/polkadot/hydration/`):

| Tag | Template | Notes |
|---|---|---|
| `Tag.Snowbridge` | `viaSnowbridgeTemplate` | V2, registered **first** — the default resolution |
| `Tag.Snowbridge` + `Tag.SnowbridgeV1` | `viaSnowbridgeV1Template` | legacy, cheaper route for the same pairs; reachable only via an explicit UI switch |

Assets on Snowbridge routes (d16ff85): `eth`, `aave`, `apyusd`, `cfg_new`, `ena`, `paxg`, `susde`, `tbtc`, `trac`, `lbtc`, `ldo`, `link`, `sky`, `wsteth`, `usdc_eth`, `usdt_eth`. Note `usdc_eth` / `usdt_eth` (Snowbridge) and `usdc_wh` / `usdt_wh` (NTT) are **different asset keys for the same underlying token** — the destination asset key picks the bridge, not the chain pair.

## In the UI

[[wiki/hydration-ui\|hydration-ui]] drives Snowbridge purely through the XC stack (no Snowbridge GraphQL client). Kill switches in `apps/main/src/config/env.ts`, applied in `apps/main/src/api/xcm.ts` by filtering routes by tag: `VITE_SNOWBRIDGE_DISABLED` (drops every `Tag.Snowbridge` route, V1 included) and `VITE_SNOWBRIDGE_INBOUND_DISABLED` (drops only Snowbridge routes **into** Hydration; Hydration → Ethereum keeps working). Siblings: `VITE_WORMHOLE_DISABLED` (also drops Basejump), `VITE_BASEJUMP_DISABLED`.

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
- [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]]
