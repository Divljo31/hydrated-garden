---
{"dg-publish":true,"permalink":"/wiki/source-onchain-monitoring-landscape/","title":"Onchain monitoring & incident-response vendor landscape (Nov 2026)","tags":["security","monitoring","incident-response","vendors","research"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"source","title":"Onchain monitoring & incident-response vendor landscape (Nov 2026)","source_kind":"article","raw_path":"(web research, 2026-11-02 — no raw copy; links in Sources)","produces_pages":["note-security-hardening-plan","note-dependency-aware-monitoring","runbook-collateral-damage-playbooks","note-emergency-levers","note-dependency-registry"],"tags":["security","monitoring","incident-response","vendors","research"],"last_updated":"2026-11-02"}}
---


# Source — Onchain monitoring & incident-response vendor landscape (Nov 2026)

**TL;DR:** Web research done 2026-11-02 after a Blockaid "onchain monitoring" tweet and a Zealynx reply to Jakub Gregus ("plan for collateral damage first: list every asset and oracle you lean on, decide now what you pause if one gets hit"). Maps the vendors, the common detect→respond pipeline, and the three places a tool can intervene. Almost every vendor is **EVM-first**; none of them understands Hydration's native pallets (Omnipool, Stableswap, HSM, XCM) out of the box. This is the input to [[wiki/note-dependency-aware-monitoring\|note-dependency-aware-monitoring]].

## Trigger

- Tweet (Bloqarl / Zealynx, reply to @GregusJakub, 2026-11-02): collateral damage is the thing to plan first — write down every asset + oracle, pre-decide the pause per dependency, so nobody improvises it at night with Telegram on fire.
- Concrete precedent the advice maps to: **Hyperbridge, 2026-04-13** — attacker exploited the Ethereum gateway and minted 1B bridged DOT in a single tx (mint-authority failure).

## The common pipeline

1. **Ingest** blocks (+ mempool on EVM) across chains.
2. **Detect** — rules (event fired, role changed, threshold crossed) + ML anomaly models + tx simulation.
3. **Triage** — severity scoring, false-positive suppression.
4. **Alert** — Slack / Telegram / PagerDuty / webhooks.
5. **Respond** — pre-signed pause / block / withdraw tx.

## Where a tool intervenes (the axis that matters)

| Mode | What it means | Stops the 1st attack tx? | Integration cost |
|---|---|---|---|
| Alert only | Tells you after funds moved | No | None |
| Detect + auto-respond / front-run | Fires a pre-signed pause after the first exploit tx; front-runs the follow-ups | No — caps loss to tx #1 | Pre-signed calls, narrow-scope signer module |
| Firewall / pre-execution screening | Tx is screened before it executes | Yes | Contract hooks, sequencer or RPC integration |

## Vendors

| Vendor | Category | How it works (as described publicly) | Hydration relevance |
|---|---|---|---|
| **Blockaid** — Onchain Monitoring | Managed detect + respond | Real-time ML, then agents that inspect bytecode, resolve proxies, analyse permissions, trace calls and simulate to confirm; feeds onchain circuit breakers + 24/7 war room. Sept 2026 expansion added exposure discovery, **third-party dependency mapping**, ecosystem-wide threat detection. July 2026: **Ecosystem Incidents Feed**. May 2026: Risk Exposure / DeFi Toxicity Monitors (AML-style exposure thresholds). Claims an Aztec exploit was flagged 6 min before execution. | Dependency mapping + incident feed is the closest product to what we want; EVM-centric |
| **Hypernative** | Managed detect + respond | AI models, graph analysis, heuristics, simulation; 300+ risk types; alerts to Slack/TG/PagerDuty/webhooks; can pause contracts / block txs. Also Guardian (pre-tx simulation), Security Oracle / Firewall (onchain policy). Has Stellar/Solana docs → non-EVM integration exists case-by-case. | Most likely vendor to do a custom Substrate integration; ask |
| **Chainalysis Hexagate** | Managed detect + respond | Acquired Dec 2024 (reported $60M). Detection mixes monitoring DSL rules, statistical/anomaly models, ML signals, threat intel. Offered free to selected chains (Base, Avalanche, Polygon, MegaETH, …) since Dec 2025. Wallet-compromise kit. | Ecosystem-program model — worth asking whether Polkadot/Hydration qualifies |
| **Cyvers** | Managed detect | Geometric ML / anomaly + behavioural analysis; skews to exchanges, custodians, trading firms. | Low |
| **Guardrail** (now part of Rain) | Managed detect + respond | Sub-second detection, ~28–30 chains, 285+ pre-built guards, can pause contracts. | Low (EVM) |
| **BlockSec Phalcon** | Detect + auto-respond / front-run | Watches mempool + chain; on first attack tx fires a **pre-signed pause** with priority bidding; Safe Module restricted to exactly that pause; LP auto-withdraw mode. | Design pattern to copy: pre-signed narrow-scope calls |
| **Forta** | Detection network → Firewall | Detection bots (custom TS) on a node network; **Firewall** screens txs with the FORTRESS ML model, runs on Forta Chain (L3). Does not cover key compromise. | Pattern only |
| **Venn (Ironblocks)** | Pre-execution screening | Pending txs go through security operators (EigenLayer) who freeze suspicious ones before they hit the chain; protected-RPC deployment option. | Pattern only (a runtime can do this natively) |
| **Tenderly** Alerts + Web3 Actions | DIY | Triggers (function call, state change, blocklisted callers, …) → serverless JS/TS that can send txs. | EVM side of Hydration only |
| **OpenZeppelin Monitor** | DIY, self-hosted | Rust, YAML config; replaced hosted Defender (sunset 2026-07-01). | EVM side only |
| **Hacken Extractor** | DIY-ish | Triggers + detectors per address; bundles Forta Attack Detector. | Low |

Vendor-reported numbers (Hypernative "0.001% FP", Forta "stops 99%") are marketing, not verified.

## What this means for Hydration

- **The native runtime already has the response levers** the vendors bolt on from outside: tradability flags, circuit breaker lockdowns, global withdraw lockdown, tx pause, emergency-admin dispatch. See [[wiki/note-emergency-levers\|note-emergency-levers]].
- **Upstream already has a monitor**: snakewatch raises `AssetLockdown` alerts, triaged by `ai_skills/circuit-breaker-incident` (see [[wiki/note-ai-skills\|note-ai-skills]]). The gap is not "an alert exists" — it is **mapping an external incident to our exposure and having the response pre-decided**.
- Trend across vendors: losses are shifting from contract bugs to **infrastructure / key / dependency compromise** (Blockaid: 74% of H1 2026 stolen value). For a chain whose assets are mostly bridged, the dependency graph *is* the attack surface.


## Bridge & protocol emergency patterns (added 2026-11-02)

Second research pass for [[wiki/note-security-hardening-plan\|note-security-hardening-plan]] — how other systems limit, detect and halt:

- **Wormhole Governor** — guardians delay token-bridge VAAs when a chain's notional outflow in a sliding 24 h window exceeds its limit; large single transfers delayed 24 h. Status queryable via Wormholescan `/api/v1/governor/*`.
- **Wormhole Global Accountant** — guardians won't attest a transfer that would break the cross-chain supply invariant; available to NTT deployments on request.
- **NTT rate limits** — per-chain outbound + per-peer inbound token buckets, refilled continuously over the window (default 24 h); over-limit transfers queue or revert; `pause` = owner or pauser, `unpause` = owner only.
- **Chainlink CCIP** — token pools enforce per-lane inbound/outbound rate limits; the Risk Management Network can "curse" a lane or everything, halting flows.
- **Chainlink Secure Mint / Proof of Reserve** — mint reverts if supply + amount would exceed attested reserves; staleness threshold on the reserve feed.
- **Snowbridge RFC-0166** — permissionless, deposit-gated (~100k DOT) end-to-end halt; Fellowship resolves genuine (refund, stays halted) vs malicious (slash, auto-resume); 7-day backstop. Rejects FRAME safe-mode as too blunt for a bridge.
- **Snowbridge RFC-0167** — per-asset rolling 24 h net-flow caps on the Ethereum Gateway and Asset Hub frontend, token-denominated (no oracle), auto-lift after 24 h — pattern explicitly lifted from Hydration's circuit-breaker net-volume tracker.
- **FRAME safe-mode / tx-pause** — chain-wide deposit-triggered call filter / per-call pause; proposed for Polkadot system chains (fellows runtimes #1089, #1164).
- **XCMP queue** — `suspend_xcm_execution` / `resume_xcm_execution` (Operational, `ControllerOrigin`).
- **Lido GateSeal → CircuitBreaker** — fast committee can pause but not unpause; GateSeals were one-time and expiring.
- **Aave Protocol Guardian** — 4-of-7 standing emergency multisig (freeze, pause reserves, LTV 0), quarterly readiness checks; froze rsETH/wrsETH after the Kelp bridge exploit (2026-04-18, 1-of-1 DVN, ~$292M unbacked rsETH).
- **Hyperbridge** (2026-04-13) — forged message changed token admin on the Ethereum gateway, 1B bridged DOT minted; contained by pausing the bridge after detection.
- **Acala aUSD** (2022) — maintenance mode, transfers and XCM-out disabled; most error mints never left the chain.
- **XChainWatcher** — open-source logic-based anomaly detector for bridges; a flow-balance invariant flags every known attack in its dataset (Ronin, Nomad).

Links: https://wormhole.com/docs/products/native-token-transfers/configuration/rate-limiting · https://docs.wormhole.com/protocol/security/ · https://docs.chain.link/ccip/evm/api-reference/v2.0.0/token-pool · https://chain.link/blog/secure-mint · https://paritytech.github.io/RFCs/new/0166-snowbridge-emergency-pause-pallet.html · https://paritytech.github.io/RFCs/new/0167-snowbridge-circuit-breakers.html · https://github.com/paritytech/substrate/pull/12092 · https://docs.rs/cumulus-pallet-xcmp-queue/0.24.0/src/cumulus_pallet_xcmp_queue/lib.rs.html · https://gh.hhj520.top/lidofinance/gate-seals · https://governance.aave.com/t/aave-emergency-guardian-protocol-signer-rotation/24944 · https://defiprime.com/kelpdao-rseth-exploit · https://range.org/blog/1-billion-tokens-minted-inside-the-hyperbridge-gateway-exploit · https://medium.com/acalanetwork/acala-incident-report-14-08-2022-392089588642 · https://arxiv.org/abs/2410.02029


## Blockaid deep dive (added 2026-11-02)

Checked after the live chain read ([[wiki/note-live-chain-check\|note-live-chain-check]]). (V) = vendor claim, not independently confirmed.

**How it works**
- **Detection pipeline:** an ML model scores every transaction. Only transactions above a risk threshold go to LLM agents.
  - The agents are read-only. They work through raw RPC (`eth_getStorageAt`, `eth_getCode`, `eth_call`, `eth_getLogs`, **`debug_traceTransaction` with `callTracer`**), and also resolve EIP-1967 proxies, check `hasRole` / owner, simulate in a sandbox and decompile bytecode.
  - A verdict takes "a few minutes", not milliseconds.
  - The tooling assumes an EVM-style RPC.
- **Monitor types:** market (oracle deviation, collateral ratios), admin (upgrades, role grants, pauses), fraud / compliance, and ecosystem (bridge incidents, dependency exploits). Alerts go to a portal, Slack, Telegram or email; PagerDuty has been used by a customer.
- **Dependency mapping:** inferred from a seed you give (an asset, wallet or protocol). Blockaid then walks the contracts it calls, and what those depend on, across chains it indexes. It does not cover cross-chain positions outside indexed chains.
- **Response options:**
  1. *Call Contract* (since 2026-05-07): a pre-uploaded transaction is **proposed** to Fireblocks (normal approval flow) or Fordefi (policy engine). The customer keeps the keys.
  2. Direct pause execution at some customers (Scallop). Who holds that key is not public.
  3. Cosigner withholding its co-signature.
  4. A 24/7 war room that has coordinated chain-level freezes (Sui, Stellar).

  No Safe-module or guardian-role pattern is documented; the docs sit behind a login.
- **Coverage:**
  - 30–50+ chains, with no public list.
  - Solana since 2026-03, Sui, HyperEVM/HyperCore, Tron, Kaia and Robinhood Chain.
  - **No Polkadot / Substrate support found**, and no precedent for onboarding chain 222222.
  - Bridge Sentinel (2026-10) is a verifier for **LayerZero only**.
- **Customers** (Onchain Monitoring): Scallop, Mellow, IPOR Fusion, Concrete, Midas, Kaia and others.
- **Documented catches:**
  - Aztec Connect (2026-06, $2.19M): attacker setup was flagged ~6 min early, but nobody could pause. rekt.news credits Blockaid and criticises the lack of response.
  - Scallop (2026-04): ~$142k was lost before the first alert, with more saved via a pause plus a Sui freeze (V).
- **Pricing:** not public; demo only.
- **Limits:** no published recall or false-positive numbers for monitoring, and IPOR's auto-pause was still in development in 2026-08. SEAL recommends two independent monitors, one self-hosted.

**Fit for Hydration**
- **What it would cover:** the EVM side only — the 3 Aave markets, HOLLAR, oracle contracts and admin roles — and only after custom chain onboarding.
- **What it would not cover:** native pallets, XCM, Snowbridge, the circuit breaker, and the NTT ↔ Hydration supply invariant.
- **Onboarding blockers seen today:** the public RPC returns *Method not found* for `debug_traceTransaction`, and `eth_getLogs` times out beyond ~100k blocks. Onboarding would need a tracing-enabled, log-indexed node.
- **Response mismatch:** Blockaid's response proposes transactions to Fireblocks / Fordefi, while our levers are TC motions and referenda (see [[wiki/note-emergency-levers\|note-emergency-levers]]). Detection alone does nothing (cf. Aztec) unless a TC-held fast path exists — and today the TC has **no role on the main market** ([[wiki/note-live-chain-check\|note-live-chain-check]]).
- **Alternatives:** Hypernative covers Polkadot Hub (EVM) since 2025-09 via the DeFi Builders Program, with no mention of parachains or XCM. Hexagate shows no Polkadot program.
- **Likely shape:** a vendor for the EVM side and the NTT counterpart chains (Ethereum, Solana, Sui, HyperEVM), plus our own Substrate monitors.

Sources: https://blockaid.io/onchain-monitoring · https://blockaid.io/blog/building-a-harness-that-lets-an-llm-explore-the-blockchain · https://blockaid.io/blog/blockaid-adds-automated-transaction-proposals-to-onchain-monitoring-for-fireblocks-and-fordefi · https://blockaid.io/blog/how-scallop-secures-its-lending-protocol-with-blockaids-onchain-monitoring · https://blockaid.io/blog/how-blockaid-detected-a-smart-contract-exploit-on-scallop-in-real-time-and-prevented-loss-on-sui-ecosystem · https://blockaid.io/blog/blockaid-expands-onchain-monitoring-support-for-solana · https://blockaid.io/blog/blockaid-partners-with-ipor-fusion-to-bring-onchain-threat-monitoring-to-its-vault-ecosystem · https://crypto-economy.com/blockaid-unveils-bridge-sentinel-dvn-for-real-time-threat-monitoring-on-layerzero/ · https://rekt.news/aztec-connect-rekt · https://hypernative.io/insights/polkadot-integrates-hypernative-to-secure-polkadot-hub-and-the-ecosystem · https://frameworks.securityalliance.org/monitoring/tools

## Sources

- Blockaid Ecosystem Incidents Feed — https://x.com/blockaid_/status/2079552936389038216
- Blockaid onchain monitoring expansion — https://bitcoinethereumnews.com/tech/blockaid-expands-onchain-security-monitoring/ , https://www.kucoin.com/news/flash/blockaid-expands-onchain-monitoring-as-infrastructure-failures-drive-74-of-crypto-losses
- Blockaid Risk Exposure — https://chainwire.org/2026/05/14/blockaid-launches-real-time-risk-infrastructure-to-help-institutions-enforce-compliance-at-defi-speed/
- Blockaid blog — https://blockaid.io/blog
- Hypernative — https://hypernative.io/products/onchain-monitoring , https://thegrid.id/profiles/hypernative
- Hexagate — https://cointelegraph.com/news/chainalysis-acquires-web3-security-firm-hexagate , https://blockchain.news/news/chainalysis-unveils-hexagate-enhancing-blockchain-security
- BlockSec Phalcon — https://docs.blocksec.com/phalcon/phalcon , https://blocksec.com/blog/how-can-block-sec-phalcon-prevent-hacker-attacks-on-de-fi-protocols-by-using-front-running-techniques
- Forta Firewall — https://bitget.cloud/news/detail/12560604359541
- Venn — https://dev.coindesk.com/nl/tech/2024/10/30/venn-network-rolls-out-new-economy-for-crypto-security
- Tenderly Alerts — https://docs.tenderly.co/alerts
- OZ Defender sunset — https://www.openzeppelin.com/news/doubling-down-on-open-source-and-phasing-out-defender
- Guardrail — https://linkedin.com/company/GuardrailAI
- Hyperbridge exploit — https://www.autheo.com/blog/bridge-security-lessons-hyperbridge-exploit-2026
