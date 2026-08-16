---
{"dg-publish":true,"permalink":"/wiki/snowbridge/","title":"Snowbridge","tags":["bridge","cross-chain","ethereum","polkadot"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"Snowbridge","tags":["bridge","cross-chain","ethereum","polkadot"],"source_count":1,"last_updated":"2026-08-15"}}
---


# Snowbridge

**TL;DR:** Snowbridge is a trustless native bridge between [[wiki/polkadot\|polkadot]] and Ethereum with no additional trust assumptions. [[wiki/hydration\|hydration]] uses it alongside [[wiki/wormhole\|wormhole]] NTT for Ethereum connectivity, while [[wiki/xcm\|xcm]] handles Polkadot interoperability.

## V1 and V2 side by side (Aug 2026)

The SDK ships both generations ([[wiki/xc-cfg\|xc-cfg]] `configs/polkadot/hydration/`):

| Tag | Template | Notes |
|---|---|---|
| `Tag.Snowbridge` | `viaSnowbridgeTemplate` | V2, registered **first** — the default resolution |
| `Tag.Snowbridge` + `Tag.SnowbridgeV1` | `viaSnowbridgeV1Template` | legacy, cheaper route for the same pairs; reachable only via an explicit UI switch |

New assets this cycle: `apyusd`, `wsol`. Note `usdc_eth` / `usdt_eth` (Snowbridge) and `usdc_wh` / `usdt_wh` (NTT) are **different asset keys for the same underlying token** — the destination asset key picks the bridge, not the chain pair.

## Gone from the UI

hydration-ui deleted its Snowbridge layer entirely — the GraphQL client, `schema.snowbridge.graphql`, the codegen script, `utils/helpers/snowbridge.ts` and the `api/provider.ts` wiring. Three GraphQL clients remain (indexer, squid, multix). Snowbridge routes are now driven purely through the XC stack. See [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], [[wiki/hydration-ui-modules\|hydration-ui-modules]].

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
- [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]]
