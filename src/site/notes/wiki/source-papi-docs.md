---
{"dg-publish":true,"permalink":"/wiki/source-papi-docs/","title":"papi.how documentation","tags":["papi","documentation","typescript","sdk"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"source","title":"papi.how documentation","source_kind":"document","raw_path":"raw/papi-docs/","upstream":"https://github.com/polkadot-api/polkadot-api-docs","cloned_at":"2026-04-14","produces_pages":["papi","papi-getting-started","papi-client","papi-typed-api","papi-providers","papi-signers","papi-codegen","papi-types","papi-typed-codecs","papi-ink","papi-sdks","papi-offline","papi-unsafe-static","papi-recipes"],"tags":["papi","documentation","typescript","sdk"],"last_updated":"2026-04-14"}}
---


**TL;DR:** Official papi.how documentation (36 markdown pages under `docs/pages/`), structured as a complete user guide from getting started through advanced recipes. Sidebar config at `vocs.config.tsx` defines page hierarchy.

## Page Structure

Sidebar organization (from `vocs.config.tsx`):
- **Introduction:** getting-started, requirements, codegen, types
- **Providers:** introduction, smoldot, websocket, enhancers
- **Signers:** introduction, extensions, raw, polkadot-signer interface
- **Top-level client:** PolkadotClient, Typed API (with sub-pages: constants, runtime APIs, view functions, storage, events, transactions), Unsafe API, Static APIs
- **Ink!:** ink! contract integration
- **Offline API:** signing without a client
- **Typed Codecs:** SCALE codec helpers
- **Recipes:** simple-transfer, multi-chain, upgrade, metadata-caching
- **PAPI SDKs:** Ink SDK, Accounts (Identity, Linked Accounts), Governance (Referenda, Bounties, Voting), Staking, Statement, Multisig
- **PAPI Apps & Built with PAPI:** external links (PAPI Console, bounties, Kheopswap, Multix, etc.)

## File Inventory

```
docs/pages/
├── getting-started.md
├── requirements.md
├── codegen.md
├── types.mdx
├── client.md
├── typed.md
├── typed/
│   ├── constants.md
│   ├── queries.md
│   ├── tx.md
│   ├── events.md
│   ├── apis.md
│   └── view.md
├── unsafe.md
├── static.md
├── offline.md
├── providers/
│   ├── index.md
│   ├── sm.md (smoldot)
│   ├── ws.md (websocket)
│   └── enhancers.md
├── signers/
│   ├── index.md
│   ├── extensions.md
│   ├── raw.md
│   └── polkadot-signer.md
├── ink.md
├── v2migration.md
├── typed-codecs.mdx
├── recipes/
│   ├── simple-transfer.md
│   ├── connect-to-multiple-chains.md
│   ├── upgrade.md
│   └── metadata-caching.md
└── sdks/
    ├── index.md
    ├── ink-sdk.md
    ├── staking-sdk.md
    ├── multisig-sdk.md
    ├── statement.md
    └── accounts/ & governance/ (subpages)
```

## Key Concepts Covered

- **Light-client first:** emphasis on smoldot for browser/lightweight deployments
- **Type-safe metadata:** codegen produces TypeScript types from runtime metadata
- **Promise/Observable duality:** both sync and async APIs available
- **Storage queries, constants, transactions, runtime APIs, events:** comprehensive chain interaction coverage
- **Signers:** browser extensions and raw signers, PolkadotSigner interface
- **Ink! contracts:** type-safe contract deployment and message encoding
- **Plugin SDKs:** specialized domains (governance, staking, accounts)

## Sources

This is the source page itself.
