---
{"dg-publish":true,"permalink":"/wiki/sdk-descriptors/","title":"descriptors (SDK)","tags":["sdk","metadata","papi","types"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"descriptors (SDK)","tags":["sdk","metadata","papi","types"],"source_count":1,"last_updated":"2026-04-13"}}
---


# descriptors (SDK)

**TL;DR:** `@galacticcouncil/descriptors` provides type-safe polkadot-api (papi) metadata descriptors for [[wiki/hydration\|hydration]] and related chains, generated from a source whitelist. Built on [[wiki/papi-codegen\|papi-codegen]] and [[wiki/papi-typed-api\|papi-typed-api]] (see [[wiki/papi\|papi]] for context).

## Generation

Descriptors are generated from a whitelist (`src/whitelist.ts`) via `papi whitelist`. The whitelist specifies which runtime APIs, constants, events, queries, and transactions to include. **Do not edit the generated output** in `.papi/descriptors/` — edit the whitelist instead.

## Coverage

The whitelist includes descriptors for:
- **Runtime APIs:** AaveTradeExecutor, CurrenciesApi, EthereumRuntimeRPC, EvmAccounts, Metadata
- **Pallets:** Omnipool, Stableswap, XYK, LBP, DCA, HSM, DynamicFees, CircuitBreaker, Staking, OTC, Router, EmaOracle, and many more
- **Hub-specific:** Assets, ForeignAssets, AssetConversion
- **Common:** DryRunApi, XcmPaymentApi, System, PolkadotXcm

## Role in the Stack

Used as a peer dependency by [[wiki/sdk-next\|sdk-next]] and [[wiki/xc-core\|xc-core]]. Provides type-safe access to all on-chain storage, transactions, and runtime APIs that the SDK needs to interact with.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
