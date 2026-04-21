---
{"dg-publish":true,"permalink":"/wiki/sdk-common/","title":"common (SDK)","tags":["sdk","utilities","shared"],"dg-note-properties":{"type":"entity","entity_kind":"product","title":"common (SDK)","tags":["sdk","utilities","shared"],"source_count":1,"last_updated":"2026-04-13"}}
---


# common (SDK)

**TL;DR:** `@galacticcouncil/common` provides shared utilities across the SDK monorepo including big number operations, XCM transformations, ERC20 helpers, and Substrate RPC abstractions.

## Modules

- **big** — BigInt/decimal conversions, power-of-10 calculations
- **xcm** — XCM transformations: AccountId32/AccountKey20 conversion, location encoding
- **erc20** — ERC20 token utilities
- **h160 / hex** — Ethereum address and hex utilities
- **account** — Account/address operations
- **SubstrateApis** — Substrate API abstraction with endpoint probing and RPC utilities (WS→HTTP conversion, legacy RPC detection)
- **log** — Logging utilities

## Role in the Stack

Used as a peer dependency by [[wiki/sdk-next\|sdk-next]], [[wiki/xc-core\|xc-core]], and other packages. Provides the foundation layer below domain-specific logic. Consumers should check downstream impact before modifying.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
