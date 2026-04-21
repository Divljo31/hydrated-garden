---
{"dg-publish":true,"permalink":"/wiki/source-hydration-general-context/","title":"Hydration — General Protocol Context","tags":["hydration","protocol","overview","polkadot","defi"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"source","title":"Hydration — General Protocol Context","author":"Hydration team (internal AI context document)","date_ingested":"2026-04-13","source_date":"2025-09-22","tags":["hydration","protocol","overview","polkadot","defi"]}}
---


# Source: Hydration — General Protocol Context

**Raw file:** `raw/hydration-general-context.md`

## Summary

**TL;DR:** Foundational reference document covering the entire [[wiki/hydration\|hydration]] protocol — architecture, products, governance, tokenomics, cross-chain design, and security — written as AI context for development tasks.

## Key Points

**Protocol identity:** [[wiki/hydration\|hydration]] (formerly HydraDX) is the largest DeFi protocol on Polkadot by TVL (~$330M+ at HOLLAR launch). It is a purpose-built Layer-1 parachain (appchain) built with Substrate, not a set of smart contracts on a general-purpose chain.

**Three pillars:** The protocol consolidates trading, lending/borrowing, and stable value into one vertically integrated environment. The three pillars are the [[omnipool\|omnipool]] (trading), [[wiki/hydration-borrow\|hydration-borrow]] (Aave v3 fork), and [[wiki/hollar\|hollar]] (stablecoin). All three interact at the runtime level, enabling composability that smart-contract systems cannot replicate.

**Trading products beyond the Omnipool:** The protocol also offers [[wiki/stableswap\|stableswap]] pools, [[wiki/xyk-pools\|XYK isolated pools]] (permissionless listing), [[wiki/dca\|DCA (Dollar-Cost Averaging)]], [[wiki/otc-trading\|OTC trading]], [[wiki/lbp\|Liquidity Bootstrapping Pools]], and the upcoming [[wiki/ice\|Intent Composing Engine (ICE)]].

**Strategy tokens:** [[wiki/gdot\|gdot]], [[wiki/geth\|geth]], and [[wiki/gsol\|gsol]] bundle staking yield, lending yield, and liquidity incentives into single tokens.

**Governance:** Fully on-chain via Polkadot's [[wiki/opengov\|OpenGov]] framework. All [[wiki/hdx\|hdx]] holders can vote. No multisigs or council intermediaries — the founding team relinquished control at mainnet launch. The Technical Committee retains only emergency powers.

**Tokenomics:** [[wiki/hdx\|hdx]] is the native token. Staking uses a [[wiki/bonding-curve\|bonding curve]] model where early claimers receive a fraction and the remainder redistributes. Governance participation (voting) accelerates the curve. A [[wiki/referrals\|referral system]] redirects 50% of asset fees from stakers to referrers.

**Treasury:** One of the most diversified on-chain treasuries in Polkadot. Seeded with ~22.5M DAI from the 2021 LBP. Holds BTC, ETH, DOT, stablecoins, tokenized gold. Actively deployed across Hydration's own products. All decisions via OpenGov referenda.

**Cross-chain:** Uses [[wiki/xcm\|XCM (Cross-Consensus Messaging)]] for interoperability within Polkadot. Supports remote swaps (swap and receive on a different chain in one flow). Bridges to Ethereum via [[wiki/snowbridge\|snowbridge]] (trustless native bridge) and [[wiki/wormhole\|wormhole]]. EVM compatibility via [[wiki/pallet-frontier\|pallet-frontier]].

**Security:** Runtime-level execution provides deeper guarantees than EVM. Active auditing, Immunefi bug bounty, circuit breakers, dynamic fees, liquidity caps.

## Entities Mentioned

- [[wiki/hydration\|hydration]] — the protocol itself
- [[omnipool\|omnipool]] — flagship AMM
- [[wiki/hollar\|hollar]] — native stablecoin
- [[wiki/hdx\|hdx]] — governance/incentive token
- [[wiki/lrna\|lrna]] — internal hub token
- [[wiki/hydration-borrow\|hydration-borrow]] — Aave v3 fork (money market)
- [[wiki/gdot\|gdot]], [[wiki/geth\|geth]], [[wiki/gsol\|gsol]] — strategy tokens
- [[wiki/ice\|ice]] — Intent Composing Engine
- [[wiki/polkadot\|polkadot]] — relay chain / ecosystem
- [[wiki/asset-hub\|asset-hub]] — Polkadot system chain

## Concepts Mentioned

- [[wiki/opengov\|opengov]] — on-chain governance framework
- [[wiki/xcm\|xcm]] — cross-chain messaging
- [[wiki/protocol-owned-liquidity\|protocol-owned-liquidity]] — POL
- [[wiki/snowbridge\|snowbridge]], [[wiki/wormhole\|wormhole]] — bridges
- [[wiki/pallet-frontier\|pallet-frontier]] — EVM compatibility layer
- [[wiki/stableswap\|stableswap]], [[wiki/xyk-pools\|xyk-pools]], [[wiki/lbp\|lbp]], [[wiki/dca\|dca]], [[wiki/otc-trading\|otc-trading]] — trading tools
- [[wiki/bonding-curve\|bonding-curve]] — staking mechanics
