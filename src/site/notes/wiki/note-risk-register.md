---
{"dg-publish":true,"permalink":"/wiki/note-risk-register/","title":"Protocol risk register (draft)","tags":["security","risk","monitoring","incident-response","draft"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"note","title":"Protocol risk register (draft)","tags":["security","risk","monitoring","incident-response","draft"],"depends_on":["note-collateral-map","note-oracle-inventory","note-dependency-registry","note-emergency-levers","note-security-hardening-plan","runbook-collateral-damage-playbooks"],"last_updated":"2026-11-02"}}
---


# Protocol risk register (draft)

**TL;DR:** One list of everything that can hurt Hydration, grouped by layer. Each row: the risk, what exists today, the gap, and **the signal a monitor would watch** — this column is the input for the monitoring plan. Detail lives in the linked notes: collateral → [[wiki/note-collateral-map\|note-collateral-map]], oracles → [[wiki/note-oracle-inventory\|note-oracle-inventory]], dependencies → [[wiki/note-dependency-registry\|note-dependency-registry]], response → [[wiki/note-emergency-levers\|note-emergency-levers]] / [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]], roadmap → [[wiki/note-security-hardening-plan\|note-security-hardening-plan]]. Severity is a first guess (H/M/L) for triage, not a scored model. Several rows were confirmed or corrected by the 2026-11-02 chain read — [[wiki/note-live-chain-check\|note-live-chain-check]].

## A. Bridges & bridged supply

| # | Risk | Sev | Today | Gap | Signal |
|---|---|---|---|---|---|
| A1 | Forged / unbacked mint over **Wormhole NTT** (guardian quorum, single transceiver, threshold 1 on all 33 legs) | H | Deposit fuse on Hydration; hub outbound limits | Hydration side ∞; fuse auto-expires; no 2nd verifier / Global Accountant | Σ burning-leg supply vs Σ hub custody; mint without matching VAA + hub lock |
| A2 | **Snowbridge** light-client / Gateway bug → unbacked Asset Hub supply → XCM into Hydration | H | Deposit fuse; RFC-0166/0167 upstream (proposed) | Not watched | Gateway escrow vs AH supply vs Hydration holdings; Snowbridge operating mode / cap trips |
| A3 | NTT hub custody drained (4/7 Safe, Solana Squads, Sui multisig) | H | Hub limits | No fast pauser (`PAUSE.md` not executed); no admin-event watch | Safe owner / threshold / module / guard changes; `OwnershipTransferred`, `PauserTransferred`, `Upgraded`, `PeerUpdated`, limit changes; any unpause |
| A4 | XCM reserve chain compromised / halted (Asset Hub, Bifrost, Interlay, other paras) | M | Per-asset fuse; `force_suspension` (TC 2/3); `suspend_xcm` not live yet (spec 455) | Origin-chain liveness not watched | Reserve-chain block production; Asset Hub safe-mode; abnormal XCM inflow per origin |
| A5 | Basejump pool drained by forged fast message before settlement | M | `setAuthorizedBridge(false)` / `withdraw` (TC) | Not watched | Landing payouts vs NTT settlements; pending queue depth (`fulfillPending` has no keeper) |
| A6 | NEAR / ZEC corridors (NEAR MPC, Omni, Sputnik DAO owner) | M | — | Not watched | NEAR NTT custody vs Hydration supply; UTXO withdraw anomalies |
| A7 | Large legit flows look like attacks (false positives → griefing pauses) | L | — | No baselines | Per-asset flow baselines |

## B. Collateral issuers & custody (upstream)

| # | Risk | Sev | Signal |
|---|---|---|---|
| B1 | Stablecoin issuer freeze / blacklist of a bridge escrow (Circle USDC/EURC, Tether USDT) → wrapped units unbacked | H | Blacklist / freeze events on escrow addresses; issuer announcements |
| B2 | tBTC: signer collusion, malicious mint (attempted 2026-05-18), governance upgrade, optimistic-minting unpause | H | Threshold contract events (see [[wiki/note-collateral-map\|note-collateral-map]] §5); tBTC supply vs BTC reserves |
| B3 | WBTC (BitGo / custody change), PAXG (Paxos) custodial risk | M | Issuer announcements; supply vs reserve attestations |
| B4 | LST / LRT accounting or slashing: vDOT (Bifrost), wstETH (Lido), jitoSOL (Jito) | M | Rate jumps vs market; slashing events; redemption pauses |
| B5 | Yield-bearing / RWA shares: apyUSD (ERC-4626 share price, donation risk), sUSDS (Sky), sUSDe (Ethena), BIL vault NAV, PRIME | M | Share-price jumps; vault pauses; NAV vs pool price |
| B6 | Native-chain risk: SOL, HYPE, SUI, NEAR, ZEC halts / reorgs | M | Chain liveness; NTT queue spikes |
| B7 | Peer protocols react first (Aave / Morpho freezing an asset we list) | M | Peer governance / guardian actions on shared assets |

## C. Oracles (summary — full list: [[wiki/note-oracle-inventory\|note-oracle-inventory]])

| # | Risk | Sev | Signal |
|---|---|---|---|
| C1 | Stale feed (DIA signer, WHM broadcaster/relayer, Bifrost pusher down; SPY off-hours) | H | Age vs heartbeat per feed; last-update gaps |
| C2 | **Representation gap** — feed prices canonical asset / fiat (BTC, EUR/USD for EURC) while we hold a bridged copy | H | Bridge invariant (A1/A2) + oracle vs on-Hydration market price |
| C3 | **Manual / fixed oracles**: SIGIL fixed $1; PRIME, wstETH/ETH, jitoSOL/SOL, vDOT-discount have manual oracles; legacy-bridge-era copies still deployed with live owners | M | Which manual oracles are live sources; time since last `setPrice`; any call to a legacy oracle |
| C4 | HOLLAR-pool collateral priced in HOLLAR (no USD leg) → HOLLAR depeg invisible | M | HOLLAR/USD market price vs $1 |
| C5 | EMA manipulation on thin pools / Uniswap v3 / derived routes; 2 s blocks shrink windows | H | Spot vs EMA deviation; one-sided volume bursts on oracle-source pools |
| C6 | CheckedOracle rejects → price frozen (apyUSD **frozen ~47 days**: 1.3672 vs check 1.4247; also the peg of pool 146) | M | `checkPrice()` vs feed divergence > 150 bps (asked for in money-market docs) |
| C7 | Oracle source swap via `AaveOracle.setAssetSources` / owner of ManagedOracles = aave-manager (referendum) | M | Any `AssetSourceUpdated`, `FallbackOracleUpdated`, ManagedOracle owner change |
| C8 | Exported HDX price → Robinhood BandHook pools (unchecked oracle) | M | HDX EMA vs external HDX market; Robinhood pool behaviour |
| C9 | DIA feed owner = updater EOA (2 hot keys cover 13 feeds) | H | Any owner / admin tx on DIA feed contracts; price jump vs external market at update |
| C10 | SPY CheckedOracle has no check oracle; PRIME / apyUSD checks are reflexive (route through pools pegged by the same oracle) | M | Pushed vs independent external price; pool imbalance on 143 / 146 |

## D. Money market (3 markets)

| # | Risk | Sev | Today | Gap | Signal |
|---|---|---|---|---|---|
| D1 | Unbacked / mispriced collateral borrows good assets (Kelp → Aave pattern) | H | Caps, isolation ceilings (PRIME $12M, apyUSD $1M) | **Confirmed on chain:** TC holds no Aave role on main / BIL → freeze / LTV 0 is referendum-speed | Supply→borrow bursts; cap utilisation; new-account collateral |
| D2 | **eMode correlation shock** — DOT eMode LT 92% (DOT/vDOT/gDOT), ETH 90%, SOL 85%: LST depeg → mass liquidation | H | — | — | LST/underlying ratio vs peg; HF clustering per eMode |
| D3 | Liquidation engine failure (PEPL worker down; omniwatch off-chain dependency; oracle-fast-path acting on bad DIA tx) | H | Validators run PEPL | Not watched | Liquidatable debt outstanding > N blocks; PEPL RPC status; omniwatch freshness |
| D4 | Bad debt accrual (illiquid collateral, failed liquidations, price-unavailable reverts) | M | — | — | Debt > collateral accounts; revert rate on `liquidate_with_pool` |
| D5 | Self-referential markets (GIGAHDX: stHDX collateral, HOLLAR debt, HDX-EMA pricing) | M | LTV 40 | — | HDX Omnipool volume / price vs external; GIGAHDX borrow growth |
| D6 | SIGIL reserve live with **zero issuance**: fixed $1, LTV 85, cap 1.1M — whoever can mint SIGIL can borrow ~$0.9M | M | Borrow disabled for SIGIL; minting is governance-level | — | Treasury borrow activity |
| D7 | Aave-fork code / config error (rate strategies, caps, new listings like HYPE) | M | Audits, chopsticks dry-runs | — | Governance decoder on MM proposals |

## E. HOLLAR / HSM

| # | Risk | Sev | Signal |
|---|---|---|---|
| E1 | HOLLAR depeg (collateral loss, facilitator over-mint, HSM drained) | H | HOLLAR price on stablepools; facilitator bucket usage; HSM holdings |
| E5 | GIGAHDX facilitator at cap (999,998 / 1M HOLLAR); main 9.97M / 13M | L | Bucket level / capacity per facilitator |
| E2 | HSM collateral failure (sUSDS etc.) | M | Collateral peg; `max_in_holding` usage |
| E3 | Flash-minter / arbitrage path abuse | M | `ArbitrageExecuted` anomalies; flash-mint sizes |
| E4 | No TC fast path for HSM | M | (control gap — see [[wiki/note-emergency-levers\|note-emergency-levers]]) |

## F. AMMs (Omnipool, Stableswap, XYK, LBP, Uniswap v3)

| # | Risk | Sev | Signal |
|---|---|---|---|
| F1 | Price manipulation / LP drain on thin assets | M | Circuit-breaker near-limit; dynamic-fee spikes; spot vs EMA |
| F2 | Per-block limits only slow multi-block dumps | M | Multi-block one-sided net flow per asset |
| F3 | Stableswap peg source wrong / reflexive (pool → EMA → peg) | M | Peg vs external rate |
| F4 | Tradability / emergency calls are `Normal` dispatch class (crowded out under spam) | M | Block fullness during incidents |
| F5 | Gamma vault keeper / Uniswap v3 curated pools misconfigured | L | Rebalance anomalies |

## G. ICE / intents

| # | Risk | Sev | Signal |
|---|---|---|---|
| G1 | Malicious / faulty solver batch | M | Settlement vs oracle price deviation; `SolverMode` changes |
| G2 | Lazy-executor EVM forwards (post-settlement calls) abused | M | Forward targets outside allowlist |
| G3 | DCA → intent migration bugs | L | Cancelled / stuck schedules |

## H. Governance, origins & keys

| # | Risk | Sev | Signal |
|---|---|---|---|
| H1 | OpenGov root compromise → also controls WHM cross-chain executors on Ethereum / Robinhood / Base / HyperEVM | H | Referendum decoder: calls touching levers, oracles, NTT minters, registry, dispatcher, cross-chain executor |
| H2 | TC compromise / unavailability (7 members: majority 4, 2/3 = 5; controls most levers; single-member `suspend_xcm` arrives with spec 458) | H | TC membership changes; motions; time-to-close in drills |
| H3 | Dispatch accounts (`EmergencyAdmin` `0xaa7e…aa7e1`, `AaveManager` `0xaa7e…aa7e0`, Treasury) — owners of many EVM contracts | H | Every `dispatch_as_*` call |
| H4 | Off-chain keys: Ethereum 4/7 Safe (NTT hubs, WHM vetoer), Solana config owner (deployer key), DIA signers, Bifrost pusher, relayer / broadcaster keys, funds-unlocker account | H | Hot-key activity; signer-set changes |
| H5 | Runtime upgrade risk (bad migration, renamed calls breaking pause entries) | M | Upgrade referenda + chopsticks diff |
| H6 | `funds-unlocker` auto-releases lockdown excess | H | Releases during an open incident |

## I. EVM layer

| # | Risk | Sev | Signal |
|---|---|---|---|
| I1 | New malicious contracts interacting with MM / pools | M | snakewatch deployment intel (exists) + interaction with core contracts |
| I2 | Precompile edge cases (dispatch `0x401`, call-permit, flash-loan receiver) | M | Unusual precompile call patterns |
| I3 | UUPS proxies owned by EmergencyAdmin / AaveManager upgraded | M | `Upgraded` events on all owned proxies |

## J. Infrastructure & operations

| # | Risk | Sev | Signal |
|---|---|---|---|
| J1 | Collator / relay forks, liveness | M | forkwatch (exists) |
| J2 | RPC / indexer dependency (omniwatch, Neckwork, Subscan) feeding liquidations / UI | M | Endpoint health + head lag |
| J3 | Frontend supply chain (npm, IPFS deploy, DNS) → drainer UI | H | Deployed bundle hash vs release; DNS / domain changes |
| J4 | Bot / alert channel compromise (Discord / Slack / Telegram tokens) | L | — |
| J5 | No paging / on-call, no drills | H | (process gap) |

## K. Social & external signals

| # | Risk | Sev | Signal |
|---|---|---|---|
| K1 | Upstream incident is public on X before it is visible on chain (Hyperbridge, Kelp were first reported there) | H | Watchlist of security-alert accounts + official accounts of every upstream dependency; keyword search on our asset / bridge names |
| K2 | **Fake alerts** from compromised official accounts (CertiK's main X account was hijacked to post a fake Uniswap vulnerability) | M | Never auto-act on social; require on-chain corroboration |
| K3 | Our own official accounts compromised → phishing users | M | Account changes; posts with unknown links |

## Notes for the monitoring plan

- **Cerebro** (transaction monitor in the hydration-node repo — location to confirm) should be the on-chain detection core for Hydration-side signals; external chains + social need separate collectors.
- **Social collection cost (Nov 2026):** X API is pay-per-use for reads (~$0.005 per post returned), Basic = 7-day search only, filtered stream needs Pro ($5k/mo, closed to new sign-ups) or Enterprise; third-party X data APIs run ~$0.05–0.40 per 1k posts. A ~50-account watchlist polled by `since_id` is cheap; full keyword streaming is not.
- Existing pieces to reuse: snakewatch (alerts + feeds), breakers dashboard, governator (referenda), forkwatch, `_status.ts` (custody), money-market's request for CheckedOracle divergence + staleness alerts.

## Sources

- [[wiki/note-collateral-map\|note-collateral-map]], [[wiki/note-oracle-inventory\|note-oracle-inventory]], [[wiki/note-dependency-registry\|note-dependency-registry]], [[wiki/note-emergency-levers\|note-emergency-levers]], [[wiki/note-security-hardening-plan\|note-security-hardening-plan]], [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]]
- X API pricing (2026): https://www.postproxy.dev/blog/x-api-pricing-2026/ · https://www.xpoz.ai/blog/guides/understanding-twitter-api-pricing-tiers-and-alternatives/ · https://opentweet.io/how-to/x-api-pay-per-use-explained
- Compromised-account precedent: https://cryptobriefing.com/certiks-x-account-hacked-sends-fake-vulnerability-warning-uniswap/
