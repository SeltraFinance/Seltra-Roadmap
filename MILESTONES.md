# Seltra — Milestones

## Programmable order execution on Soroban — SCF Build submission (Open Track)

---

**Total budget:** **$94,000**
**Timeline:** December 15, 2026 → April 13, 2027 (**4 months**)
**Budget split:** T1 $31,200 (33 %) / T2 $32,400 (35 %) / T3 $30,400 (32 %)
**Integrations delivered:** Aquarius and Soroswap (T1), DeFindex (T2), Stellar Wallets Kit (T2).

Every component below is specified in [ARCHITECTURE.md](./ARCHITECTURE.md) (section references in each deliverable). Each tranche is self-contained and testable on its own; each completion criterion is quantitative and verifiable from public sources: the GitHub repository and its CI, Stellar testnet and mainnet explorers, and the public orderbook and indexer endpoints. No criterion depends on user acquisition, volume targets or third-party commercial agreements.


---

## Tranche 1 — Settlement core, both venues and threat model (MVP) — $31,200

> Outcome: on-chain orders and the custody path exist, are proven against the invariants in ARCHITECTURE.md §7, and execute routed fills on testnet through both venue adapters. The threat model, including yield mode, is published before any public deployment.
> Period: December 15 → January 29, 2027.
> **Distribution channels.** Public repository and threat model shared in Soroban developer channels; existing Seltra users on Avalanche informed through the application and the project's X account.

### [Deliverable 1.1] SeltraSettlement — on-chain orders and custody path
**Description:** The single custody contract in Rust against the Soroban SDK: `place_orders` (batch placement, maker auth, order ids `(maker, nonce)` with no global counter, orders and maker books in temporary storage with TTL covering expiry, maker book per spend token bounded at 64 open orders), `cancel_orders` (per-order, batchable, never pausable), `fill` (DEX path, one order per call) and `cross` (P2P path: mirrored pairs, whole default-mode orders, per-side surplus with no price conversion, batch bounded by `MAX_CROSSES`, any invalid pair reverting the whole call), both pulling input with `transfer_from` under the contract's allowance, read-only `get_order` / `open_orders`, guardian `pause` / `unpause`. The contract computes and sets its own SEP-41 allowance on each maker's spend token to the sum of open orders with the latest expiry; the client never builds an `approve`. Output measured by `token_out` balance delta, surplus split by `surplus_bps`, keeper reward, `__constructor` initialisation, no upgrade entrypoint. Each order spends exactly `amount_in` or does not fill; partial execution is expressed as slices (ARCHITECTURE.md §2, §3, §4, §8).
**Completion criteria:**
- Public repository with reproducible build instructions; the built WASM is verified to expose no upgrade entrypoint (documented check in CI)
- Both settlement paths (DEX fill through a mock adapter, P2P cross) execute in the local Soroban test environment; ≥ 55 unit tests covering invariants 1-11 of ARCHITECTURE.md §7, each mapped to at least one named test
- Line coverage > 90 % on the settlement crate (coverage report published by CI)
- Fuzz and property tests on conservation, allowance accounting and surplus arithmetic: ≥ 100,000 iterations over randomised sequences of placements, cancellations, fills and expiries, with randomised amounts, decimals and `surplus_bps`; after every step the allowance equals the sum of open orders (invariant 3); zero failures
- Negative tests pass in 100 % of cases: filled, cancelled or expired order submitted again, cancellation by a non-maker, 65th open order on a spend token, fill missing `min_out` by one unit, two fills of one order, a fill after the maker emptied the balance or revoked the allowance; for `cross`, one test per validity rule of ARCHITECTURE.md §4.3 (repeated id within and across pairs, non-mirrored tokens, yield-mode side, expired side, each floor missed by one unit, batch above `MAX_CROSSES`), each leaving every order and balance unchanged
- P2P conservation: over ≥ 100,000 fuzzed `cross` batches with randomised amounts, decimals and `surplus_bps`, maker payout plus keeper share equals the counterparty's input exactly in each token and the settlement contract's balances are unchanged after every call; zero failures
- Placement and cancellation transactions signed with a plain account key (no authorization-entry signing) succeed in the test environment and on testnet (hashes); ≥ 10 placements by different makers applied in the same testnet ledger all succeed (no shared hot key)
- Measured resource budget for a single P2P cross, for a `cross` call at `MAX_CROSSES`, for the deepest allowed DEX route and for a 20-order ladder placement, each at most 50 % of every per-transaction limit (`MAX_CROSSES` fixed from this measurement), and measured placement cost (fee and rent) for a seven-day and a maximum-duration order, published against live network settings
**Estimated date:** January 3, 2027
**Budget:** $13,200

### [Deliverable 1.2] SeltraRouter + Adapter Registry + Aquarius and Soroswap adapters
**Description:** Router with `route` / `quote`, holding no funds at rest and no authorization; registry with timelocked `propose_adapter` / `activate_adapter`, guardian `pause_adapter`, `is_active`; two venue adapters behind the common interface, wrapping the Aquarius router (`swap_chained`) and the Soroswap router (`swap_exact_tokens_for_tokens`), each pre-authorizing the venue's token pull through `authorize_as_current_contract` (ARCHITECTURE.md §3).
**Completion criteria:**
- ≥ 10 routed fills on Stellar testnet through each adapter (≥ 20 in total), each with a public transaction hash and the maker receiving ≥ `min_out` (explorer links published), on a test pair seeded on both venues by the published, re-runnable deployment script
- Route through an unknown or paused adapter reverts before any token movement in 100 % of negative tests; pausing one adapter leaves fills through the other unaffected (hashes)
- Adapter activation impossible before the 72-hour timelock elapses (test with ledger advance); guardian pause blocks an adapter immediately
- Malicious-adapter test suite (keeps tokens, under-reports output, returns the wrong token) run against both adapters: 100 % revert, maker balances unchanged; router and adapter balances are zero after every fill
- Router, registry and adapter crates with line coverage > 90 %
**Estimated date:** January 15, 2027
**Budget:** $11,600

### [Deliverable 1.3] Threat model and invariant map
**Description:** STRIDE threat model with identifiers, covering the contract-managed allowance (computation, expiry, revocation, exposure of open orders to a fill-path defect), on-chain order storage and spam bounds, placement authorization and the scope of the maker's invocation tree, venue and adapter compromise, keeper behaviour (griefing, racing, surplus extraction), the "creates orders and pulls liquidity" scenario, AMM price manipulation, programmable execution against thin AMM liquidity, yield mode (vault compromise, lending-pool oracle risk, redemption liquidity, exposure caps), storage archival, off-chain services and frontend supply chain, classic-asset issuer actions. Each threat maps to a mitigation and to a named test or to an explicit residual-risk statement; the trust-assumption table per phase is published alongside (ARCHITECTURE.md §6, §7).
**Completion criteria:**
- Published document in the repository with ≥ 45 identified threats, each with: identifier, affected component, impact class (loss of funds / forgone surplus / availability / operator cost), mitigation, and either a named test or a residual-risk statement
- 100 % of the 13 invariants of ARCHITECTURE.md §7 traced to at least one test identifier (invariants 12-13 to tests delivered in Deliverable 2.3)
- Trust-assumption table per phase (testnet, mainnet launch, post-award), with default mode and yield mode stated separately
- Reviewed and signed off by one engineer outside the contract authors (name and date in the document)
**Estimated date:** January 29, 2027
**Budget:** $6,400

---

## Tranche 2 — Public testnet, open keeper network, yield mode — $32,400

> Outcome: the protocol runs on Stellar testnet with keepers that anyone can operate from a published image and that read orders directly from the contract, users place orders, splits and ladders from any Stellar wallet through a public application, and capital behind resting orders can opt into an allowlisted DeFindex vault and still settle atomically.
> Period: January 30 → March 1, 2027.
> **Distribution channels.** Public testnet application; open-source keeper image and operator documentation offered to keeper operators already contacted by the team; MCP server published for AI-assistant users.

### [Deliverable 2.1] Testnet deployment, open-source reference keeper and monitoring
**Description:** Deployment of SeltraSettlement, SeltraRouter, Adapter Registry and both venue adapters to Stellar testnet with `stellar-cli`; guardian and timelock configured; storage bump job running. Reference keeper published open source as a one-command Docker image with a configuration file (pairs, size bounds, venues, minimum surplus, RPC endpoints, channel accounts) and operator documentation; it reads open orders from the contract, quotes both venues, simulates through Stellar RPC and submits through channel accounts. Indexer ingesting contract events; adversarial scenarios from the threat model run against the deployment; monitors for keeper liveness and concentration, fills executed exactly at `min_out`, adapter and vault proposals and activations, storage TTL headroom and guardian actions (ARCHITECTURE.md §6, §7, §9, §10).
**Completion criteria:**
- Testnet contract addresses published; ≥ 100 filled orders on testnet across both paths and both venues, including ≥ 10 P2P crosses of which ≥ 2 are batched `cross` calls, listed with transaction hashes by the public indexer
- Reference keeper repository public under an OSI licence; a fresh operator following only the documentation runs a keeper that executes ≥ 5 testnet fills (documented walkthrough with addresses and hashes)
- With the order API switched off, the reference keeper discovers open orders from contract events through a third-party Stellar RPC provider, verifies them against contract state and executes ≥ 10 testnet fills (hashes, configuration published)
- Team keeper runs unattended for ≥ 7 consecutive days with uptime > 99 % (logs published) using ≥ 3 channel accounts; keeper selects the better venue per fill in a documented sample of ≥ 20 fills on the test pair seeded on both venues, with both quotes
- Adversarial suite covering 100 % of default-mode threats classified "loss of funds" or "forgone surplus" in Deliverable 1.3 passes, each test named by threat identifier; guardian pause on testnet blocks fills and placements while `cancel_orders` and expiry still succeed (hashes)
- ≥ 8 monitors live with documented trigger, baseline, severity and response; an injected event triggers its monitor within < 60 s in 100 % of ≥ 10 injections; public indexer endpoint exposes distinct keeper addresses, largest keeper's share of fills, last fill age and fills at exactly `min_out`
**Estimated date:** February 12, 2027
**Budget:** $12,600

### [Deliverable 2.2] Order pipeline — placement flow for every wallet, order API, Order Builder, MCP server and testnet application
**Description:** Order placement and cancellation through Stellar Wallets Kit (`signTransaction`, implemented by every kit wallet), with the allowance the transaction will set, the storage rent for the chosen expiry and the trustline status shown before signing; read-only order API over REST and WebSocket built from contract state and events, flagging orders whose maker balance or allowance is below `amount_in`; Order Builder compiling split (same price) and ladder (stepped prices) configurations into one `place_orders` batch, with a $10 minimum per order and per slice enforced by the application; MCP server (quote, read orders, prepare unsigned placement and cancellation transactions, no signing capability); public Next.js testnet application with expiry presets and market-based default `min_out`, fill gauge, per-slice and whole-order cancellation, and self-fill (ARCHITECTURE.md §4.2, §5, §9).
**Completion criteria:**
- Orders placed and cancelled on testnet from each of Freighter, Lobstr, xBull, Albedo, Hana Wallet and a Ledger device through the kit in clear-signing mode (≥ 3 placements and 1 cancellation per wallet, hashes listed); tested support matrix published for every Stellar Wallets Kit wallet
- A placed order appears on the order API WebSocket feed within < 2 s of ledger close (measured over ≥ 50 placements); an order whose maker balance or allowance drops below `amount_in` is flagged within one ledger of the indexer observing it
- A ladder of ≥ 20 orders and a split of ≥ 10 slices are each placed in one transaction and have ≥ 50 % of their orders filled on testnet by keepers; one slice of a partially filled ladder is cancelled alone, then the rest in one transaction, and the on-chain allowance matches the remaining open orders after each step (hashes, allowance reads)
- Public testnet URL from which an order is placed, seen resting and cancelled from ≥ 10 distinct testnet accounts (addresses listed); self-fill executed from the application in ≥ 5 documented cases (hashes)
- An AI assistant connected to the MCP server prepares ≥ 3 orders and 1 cancellation that the user signs in their wallet on testnet (hashes); the server holds no key (documented check)
- An order or slice worth less than $10 at the current quote is refused by the application before signing in 100 % of tests
- OpenAPI specification published; frontend builds reproducibly with pinned dependencies and subresource integrity, and checks the network passphrase and contract addresses before signing (documented checks in CI)
**Estimated date:** February 20, 2027
**Budget:** $10,900

### [Deliverable 2.3] Yield adapter on DeFindex vaults (opt-in)
**Description:** Yield adapter integrated against the DeFindex vault contract: `deposit` (called inside `place_orders`; vault `deposit` with `from` = maker, `invest = true`, shares minted to the maker's own account), `redeem` (called by SeltraSettlement inside `fill` after moving the order's shares under the contract's allowance on the share token; vault `withdraw` with the adapter as `from`), `preview_redeem` for keeper simulation, `exposure_cap`; fill input `min(redeemed, amount_in)` with accrued yield returned to the maker; registry accepting only single-asset, non-upgradable DeFindex vaults behind the timelock, pausable by the guardian; first allowlisted single-asset DeFindex vault with a Blend strategy on testnet; application flow to opt in per order with the vault's fees, roles and added risk stated at the moment of choosing it, and a "waiting on liquidity" state (ARCHITECTURE.md §3 Yield adapter, §7).
**Completion criteria:**
- ≥ 15 yield-mode orders on testnet whose placement transaction deposits into the allowlisted DeFindex vault, which accrue and settle through `fill` with redemption in the same invocation (hashes), including ≥ 3 filled within 10 ledgers of placement; in 100 % of them `min_out` holds and the value redeemed above `amount_in` is returned to the maker and absent from the keeper's share (balance report published)
- No Seltra contract holds vault shares between invocations: share balances of the yield adapter and settlement contract are unchanged before and after every tested fill; cancelling a yield-mode order leaves the shares in the maker's account and lowers the share allowance (hashes)
- Yield tests against the DeFindex vault and Blend strategy contracts in the local test environment pass in 100 % of cases: constrained Blend liquidity makes `preview_redeem` fail and the keeper skip rather than revert, with the order shown as waiting on liquidity; deposit above the exposure cap reverts; paused, multi-asset or upgradable vault rejected; vault value loss makes the fill revert with the maker's shares unchanged
- Invariants 12 and 13 of ARCHITECTURE.md §7 covered by named tests; adapter crate coverage > 90 %
**Estimated date:** March 1, 2027
**Budget:** $8,900

---

## Tranche 3 — Mainnet, verified checkpoint by checkpoint — $30,400

> Outcome: the protocol is live on Stellar mainnet with each surface verified on its own: contracts hardened against the findings of the independent review and deployed, the production application with its wallet integration and yield mode, and a keeper network in which two of the three operators are outside the team.
> Period: March 2 → April 13, 2027.
> **Distribution channels.** Production application; listing requests to the Stellar wallets integrated in Tranche 2 and to the venues routed (Aquarius, Soroswap, DeFindex) for their user communities; two outside keeper operators named at launch.

### [Deliverable 3.1] Hardened contracts and mainnet deployment
**Description:** Closure of the findings raised on the contracts delivered in Tranches 1-2 by the independent review obtained through Stellar's security review programme for SCF-funded protocols or an equivalent independent firm (core contracts on the commit frozen at the end of Tranche 1, the yield adapter on the commit frozen at the end of Tranche 2; the review itself is not funded by this award), then deployment of SeltraSettlement, SeltraRouter, Adapter Registry, both venue adapters and the yield adapter to Stellar mainnet with verified source and published addresses; adapters and the first DeFindex vault activated through the timelock with a published exposure cap; guardian multisig configured and published (ARCHITECTURE.md §3, §6, §7, §10).
**Completion criteria:**
- Public hardening report mapping every finding to its severity, the commit that fixes it and the test that covers it; 100 % of critical and high findings closed before deployment, 100 % of medium findings closed or documented with rationale
- Full test suite, fuzz suite, adversarial suite and yield fallback suite green on the hardened code; coverage thresholds of Tranches 1-2 maintained (CI report)
- Mainnet contract addresses published with reproducible build instructions; a third party rebuilds the WASM and matches the on-chain hash (documented procedure and one external confirmation)
- Aquarius and Soroswap adapters and the DeFindex vault activated on mainnet through the registry timelock, exposure cap readable on-chain (hashes); guardian is a 2-of-3 multisig account verifiable on the explorer, with one signer outside the Seltra team (named)
**Estimated date:** March 24, 2027
**Budget:** $12,400

### [Deliverable 3.2] Production application, wallet integration and yield mode on mainnet
**Description:** Production application with Stellar Wallets Kit against mainnet, for every kit wallet, with default and yield-mode placement, splits and ladders, the order API and the MCP server pointed at mainnet; monitors from Deliverable 2.1 running against mainnet; operational runbooks (incident, pause, vault pause, migration) exercised on the live deployment (ARCHITECTURE.md §5, §6, §9).
**Completion criteria:**
- ≥ 25 mainnet fills across both paths and both venues, including ≥ 3 P2P crosses and ≥ 5 yield-mode fills, of orders placed through the production application from ≥ 4 different wallet applications including Lobstr and xBull (hashes listed by the public indexer)
- A ladder of ≥ 10 orders placed from the production application in one mainnet transaction, one slice cancelled alone, the on-chain allowance matching the remaining open orders after each step (hashes, allowance reads)
- Pause drill on mainnet: fills and placements paused and unpaused, `cancel_orders` executed during the pause, vault paused and unpaused (hashes); runbooks published in the repository
- ≥ 8 monitors live against mainnet with documented trigger, baseline, severity and response; public indexer endpoint serving mainnet fills, distinct keeper addresses and fills at exactly `min_out`
**Estimated date:** April 4, 2027
**Budget:** $9,500

### [Deliverable 3.3] Production keeper network and independent operators
**Description:** Production keeper infrastructure (self-hosted Stellar RPC with provider fallback, channel accounts, alerting) operated by the team as a backstop, plus two keepers operated by parties outside the Seltra team from the published image, each operator named (any related party declared as such) and its keeper address published (ARCHITECTURE.md §6).
**Completion criteria:**
- ≥ 2 keepers operated by named parties outside the Seltra team each execute ≥ 20 fills on testnet and ≥ 5 fills on mainnet from their own keeper addresses (addresses and hashes published; operator statements linked)
- Indexer metrics on mainnet over the first 10 days after deployment show ≥ 3 distinct keeper addresses with fills, with the largest keeper's share of fills reported publicly
- Team backstop keeper uptime > 99.5 % over the same period; RPC failover exercised at least once (incident log)
- Operator documentation updated with mainnet configuration; walkthrough from zero to first mainnet fill reproduced by each outside operator in < 1 day (their confirmations linked)
**Estimated date:** April 13, 2027
**Budget:** $8,500

---

## Summary

| Tranche | Phase | Period | Deliverables | Integrations | Budget | % |
|---|---|---|---|---|---|---|
| T1 | Settlement core, both venues and threat model (MVP) | Dec 15 → Jan 29, 2027 | 3 | Aquarius, Soroswap | $31,200 | 33 % |
| T2 | Public testnet, open keeper network, yield mode | Jan 30 → Mar 1, 2027 | 3 | DeFindex, Stellar Wallets Kit | $32,400 | 35 % |
| T3 | Mainnet, verified checkpoint by checkpoint | Mar 2 → Apr 13, 2027 | 3 | All of the above on mainnet | $30,400 | 32 % |
| **Total** | | **4 months** | **9** | **4** | **$94,000** | 100 % |
