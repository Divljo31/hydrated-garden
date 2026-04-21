---
{"dg-publish":true,"permalink":"/wiki/xc-scan/","title":"xc-scan","tags":["sdk","cross-chain","monitoring","scanning"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"xc-scan","tags":["sdk","cross-chain","monitoring","scanning"],"source_count":1,"last_updated":"2026-04-13"}}
---


# xc-scan

**TL;DR:** `@galacticcouncil/xc-scan` provides cross-chain transaction scanning and journey tracking via indexer queries, monitoring transfers across chains with status, protocols, and USD values.

## Journey Tracking

Monitors transfers across chains via indexer queries:

```ts
XcJourneyBuilder.journeys()
  .address(addr)
  .status('sent', 'received')
  .build()
```

## Journey Data

Each journey includes:
- Transaction hashes (origin and destination)
- Status: sent, received, failed, timeout, waiting, unknown
- Origin/destination protocol: [[wiki/xcm\|xcm]], [[wiki/wormhole\|wormhole]], [[wiki/snowbridge\|snowbridge]]
- Asset operations with roles (transfer, swap, fee)
- USD value tracking

## Clients

Uses SSE (Server-Sent Events) and HTTP clients for real-time and polling-based monitoring.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
