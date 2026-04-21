---
{"dg-publish":true,"permalink":"/wiki/route-suggester/","title":"route-suggester","tags":["rust","routing","algorithm","performance"],"dg-note-properties":{"type":"entity","entity_kind":"product","title":"route-suggester","tags":["rust","routing","algorithm","performance"],"source_count":1,"last_updated":"2026-04-13"}}
---


# route-suggester

**TL;DR:** A high-performance Rust crate in the SDK monorepo for DEX route discovery, using BFS to enumerate all valid multi-hop trading routes with cycle prevention and trusted/isolated pool classification.

## Algorithm

Uses BFS to discover all acyclic paths up to 9 hops (`MAX_NUMBER_OF_TRADES`). Prevents cycles by tracking asset revisits (no asset appears twice in a route) and pool reuse (same pool never used twice).

## Routing Strategy

Same classification as the TypeScript [[wiki/smart-order-router\|smart-order-router]]:
- **Trusted pools:** [[omnipool\|omnipool]], [[wiki/stableswap\|stableswap]], [[wiki/lbp\|lbp]], Aave, HSM
- **Isolated pools:** [[wiki/xyk-pools\|XYK]] only

Three modes: trusted-only, isolated-only, or mixed — depending on which pool types contain the source and destination assets.

## Integration

Consumers implement the `PoolProvider` trait to query runtime pool state, then call:
```rust
RouteSuggester::<YourPoolProvider>::get_routes(asset_in, asset_out, state)
```

Returns `Vec<Route<AssetId>>` — bounded vectors of Trade structs.

## Testing

Includes unit tests, mainnet snapshot tests with real pool data, and PoolProvider integration tests.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
