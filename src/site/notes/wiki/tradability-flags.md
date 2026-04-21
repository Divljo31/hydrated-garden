---
{"dg-publish":true,"permalink":"/wiki/tradability-flags/","title":"Tradability Flags","tags":["security","omnipool","governance"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"Tradability Flags","tags":["security","omnipool","governance"],"source_count":1,"last_updated":"2026-04-13"}}
---


# Tradability Flags

**TL;DR:** Each asset in the [[omnipool\|omnipool]] has a `Tradability` bitmask controlling SELL, BUY, ADD_LIQUIDITY, and REMOVE_LIQUIDITY permissions, with [[wiki/lrna\|lrna]] defaulting to SELL-only and the Technical Committee able to pause assets in emergencies.

## Flags

| Flag | Meaning |
|------|---------|
| `SELL` | Asset can be sold into the pool |
| `BUY` | Asset can be bought from the pool |
| `ADD_LIQUIDITY` | LPs can add liquidity for this asset |
| `REMOVE_LIQUIDITY` | LPs can remove liquidity for this asset |

[[wiki/lrna\|lrna]] has a hardcoded default of `SELL` only (`DefaultHubAssetTradability`).

## Safe Withdrawal

A `safe_withdrawal` state is inferred when both `SELL` and `BUY` are disabled. The protocol treats this as a graceful shutdown mode — the [[wiki/dynamic-fees\|dynamic withdrawal fee]] is waived, allowing LPs to exit without penalty.

## Emergency Use

The Technical Committee can invoke `set_asset_tradable_state` to selectively disable any combination of flags for any asset in an emergency. This extrinsic is a candidate for `Operational` dispatch class (highest priority, bypasses block weight limits), ensuring pausing can succeed even during high-throughput attacks. This is the primary tool for responding to an active exploit.

## Governance

Tradability flags are governed on-chain by [[wiki/hdx\|hdx]] holders via [[wiki/opengov\|opengov]].

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
