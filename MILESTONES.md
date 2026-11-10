# Seltra Milestones

SCF Build delivery plan for programmable limit-order execution on Stellar/Soroban.

## Plan summary


| Tranche                                                | Period                       | Delivery budget | Exit outcome                                                                                              |
| ------------------------------------------------------ | ---------------------------- | --------------- | --------------------------------------------------------------------------------------------------------- |
| 1. Settlement core, both venues, and threat model      | 2026-12-15 to 2027-01-29     | USD 31,200      | Frozen core contracts, two venue adapters, tested invariants, and externally reviewed threat model        |
| 2. Public testnet, open keeper network, and yield mode | 2027-01-30 to 2027-03-01     | USD 32,400      | Public testnet application, independent keeper path, complete order pipeline, and frozen DeFindex adapter |
| 3. Mainnet and checkpoint-by-checkpoint verification   | 2027-03-02 to 2027-04-13     | USD 30,400      | Hardened mainnet deployment, production application, and at least two independent keeper operators        |
| **Total**                                              | **2026-12-15 to 2027-04-13** | **USD 94,000**  | **Mainnet launch and independently operable execution network**                                           |


The tranche budgets above are delivery-cost allocations and reconcile to the USD 94,000 total in the application. The SCF application separately describes an award-disbursement schedule of 10% on approval, then 20%, 30%, and 40% at tranche completion, with professional user testing at mainnet launch. Those payout percentages must not be confused with the delivery-cost allocation in this file.

> **Consistency correction to apply to the SCF cells before submission:** the current application wording combines unconditional allowance equality with one shared allowance expiry, independently expiring orders, and a cancellation that succeeds when `try_approve` rejects. These cannot all imply continuous equality. This file and `ARCHITECTURE.md` use the technically consistent rule: equality at successful maker-authorized synchronization points; permanent unfillability plus finite, expiring stale capacity after a rejected reduction or an earlier individual order expiry; reconciliation on the next successful maker-authorized computed `approve`, or expiry of the shared allowance. Update the corresponding wording in `Application Build!E14:E16` so the application and repository do not diverge.

SCF funding is not allocated to marketing or promotion. The distribution activities below publish deliverables and notify relevant technical or existing-user communities; they are not budget line items.

## Cross-document consistency gates

These gates apply before the relevant contract freeze:

- [ ] Keep all component names, contract methods, trust assumptions, and the 13 numbered invariants aligned with `ARCHITECTURE.md`.
- [ ] Resolve allowance equality, independent order expiries, and cancellation liveness exactly as specified in [Architecture section 2.2](./ARCHITECTURE.md#22-cancellation-liveness-and-allowance-reconciliation): separate asset/share aggregates; equality at synchronization points; permanently unfillable cancelled/expired orders; later successful maker-authorized computed `approve` or shared-allowance expiry.
- [ ] Prove the exact nested Soroban authorization tree for settlement, router, adapter, venue, vault, and share-token calls without permissive authorization mocks.
- [ ] Bind settlement to one immutable router address; publish and independently reproduce the direct-WASM hashes for every Seltra-owned contract.
- [ ] Verify `min_out` and every maker/keeper/yield payout from exact final-recipient balance deltas, not only the settlement's route-output delta.
- [ ] Separate the pause guardian from the registry governor and test every role boundary and 72-hour activation path.
- [ ] Query live Stellar network settings for resource tests. The October 2026 values in the design are a snapshot, not hard-coded protocol constants.
- [ ] Pin the exact Aquarius, Soroswap, DeFindex, and Blend deployments/interfaces used by the tests.
- [ ] Preserve the architecture section numbering referenced by the application.



## Tranche 1: Settlement core, both venues, and threat model

**Status:** Planned  
**Period:** 2026-12-15 to 2027-01-29  
**Budget:** USD 31,200

**Distribution:** Publish the repository and threat model in Soroban developer channels. Inform existing Seltra users on Avalanche through the application and the project's X account.

### 1.1 `SeltraSettlement`: on-chain orders and spending path

**Due:** 2027-01-03  
**Budget:** USD 13,200  
**Architecture:** [sections 2-4 and 8](./ARCHITECTURE.md#2-design-constraints)

#### Scope

Implement the immutable Rust/Soroban settlement contract with:

- batch placement authorized by an ordinary Stellar wallet transaction;
- `(maker, nonce)` order IDs and bounded on-chain order storage;
- per-order cancellation that cannot be paused;
- whole-order DEX fills and direct P2P crosses;
- explicit expiry and guardian pause behavior;
- a contract-managed SEP-41 allowance based on the maker's open orders;
- separate aggregation of default-mode `amount_in` and yield-mode `shares` by the actual token contract for which settlement is spender;
- a per-order surplus split, defaulting to 70% keeper and 30% maker;
- no upgrade entrypoint; and
- no protocol fee or fee parameter.



#### Completion evidence

- [ ] A public repository contains a reproducible contract build.
- [ ] CI inspection demonstrates that the compiled settlement WASM has no exposed or reachable upgrade path.
- [ ] At least 55 unit tests cover architecture invariants 1-11, with every invariant mapped to a named test.
- [ ] Settlement line coverage is greater than 90%.
- [ ] At least 100,000 randomized sequences of placements, cancellations, fills, and expiries complete with zero invariant failures.
- [ ] Every successful maker-authorized settlement-computed `approve` sets the relevant default-asset or yield-share allowance to its live-order aggregate; before the next individual expiry, fills reduce allowance and aggregate by the same amount.
- [ ] If a cancellation-time allowance reduction is rejected, cancellation still succeeds, the deleted order cannot fill, `allowance_sync_failed` exposes the retained finite allowance and expiry, and equality is restored only by a later successful maker-authorized update or by shared-allowance expiry; earlier individual expiry is tested as a separate finite stale-capacity case.
- [ ] Validation rejects zero/negative or out-of-range amounts, identical token pairs, `surplus_bps` outside `0..=10_000`, past/overlong expiries, unchecked conversion, and arithmetic overflow.
- [ ] DEX and P2P tests coalesce payouts by `(token, recipient)`, snapshot every unique final recipient, and prove exact aggregate deltas while each maker entitlement remains at least `min_out`; self-fill/address-alias cases pass and fee-on-transfer payouts revert atomically.
- [ ] All negative tests pass: replay of filled/cancelled/expired orders, cancellation by a non-maker, a fill one base unit below `min_out`, double fill, fill after balance or allowance removal, and one test for each P2P cross rule.
- [ ] Placements signed with a plain account key succeed on testnet.
- [ ] At least 10 different makers place orders in the same ledger, with transaction hashes published.
- [ ] Resource reports cover a single cross, a cross batch at `MAX_CROSSES`, the deepest DEX route, and a 20-order ladder.
- [ ] Every measured boundary operation consumes no more than 50% of every live per-transaction resource limit.
- [ ] Placement fee and rent are published for a seven-day order and a maximum-duration order using then-current network settings.



#### Exit gate

The settlement interface, storage model, authorization tree, and invariants are frozen for independent review. Any unresolved ambiguity in allowance reconciliation or route authorization blocks the freeze.

### 1.2 `SeltraRouter`, Adapter Registry, Aquarius adapter, and Soroswap adapter

**Due:** 2027-01-15  
**Budget:** USD 11,600  
**Architecture:** [section 3](./ARCHITECTURE.md#3-contracts-and-core-components)

#### Scope

Implement:

- a router with read-only `quote` and state-changing `route` operations;
- complete route validation before the first token movement;
- a registry with a separate authenticated governor, exact proposal payloads, 72-hour activation delay, permissionless activation, and per-integration guardian pause;
- an Aquarius adapter wrapping `swap_chained`; and
- a Soroswap adapter wrapping `swap_exact_tokens_for_tokens`.

The immutable settlement binds one immutable router address. The router never takes custody and has no standing approval, admin privilege, or durable token allowance. Settlement authorizes the precise deeper transfer directly to the first selected adapter; each selected adapter may receive only the exact hop input and only within one atomic invocation. All input/output amounts are verified by balance deltas, and all nested token pulls use exact invocation-scoped authorization.

#### Completion evidence

- [ ] At least 10 routed Stellar testnet fills execute through Aquarius.
- [ ] At least 10 routed Stellar testnet fills execute through Soroswap.
- [ ] At least 20 total fill hashes are published, and every maker receives at least `min_out`.
- [ ] A published, re-runnable script seeds one compatible test pair on both venues.
- [ ] A route using an unknown or paused adapter reverts before any token movement in all negative tests.
- [ ] Pausing one adapter leaves the other adapter operational.
- [ ] Adapter activation before the 72-hour ledger-timestamp delay is impossible in the ledger-advance test.
- [ ] Only the registry governor can propose an adapter/vault/cap record; any account can activate the unchanged payload after maturity; changing address, hash, version, asset, route constraint, or cap restarts the delay.
- [ ] Guardian pause blocks the selected adapter immediately.
- [ ] The malicious-adapter suite covers retained input, under-reported output, wrong-token output, and an invalid nested authorization request.
- [ ] All malicious-adapter cases revert with maker balances unchanged.
- [ ] In controlled normal fills, settlement/router/adapter route-token balances are zero before and after. In a separate dust-prefill test, every non-zero pre-existing balance is exactly unchanged and never consumed or swept.
- [ ] Deployment evidence proves settlement, router, registry, and Seltra-owned adapters use the published direct WASM hashes, do not use mutable executable references, and contain no reachable upgrade path.
- [ ] Router, registry, and adapter crates each maintain greater than 90% line coverage.



#### Exit gate

The exact adapter interfaces, deployed test contracts, route-depth bound, `MAX_CROSSES`, and nested authorization trees are documented and frozen with the core contracts.

### 1.3 Threat model and invariant map

**Due:** 2027-01-29  
**Budget:** USD 6,400  
**Architecture:** [sections 6-7](./ARCHITECTURE.md#6-keeper-network-deployment-and-trust-assumptions)

#### Scope

Publish a STRIDE threat model covering:

- contract-managed allowances and cancellation-time reconciliation;
- on-chain order storage, TTL, archival, and spam bounds;
- maker, settlement, router, adapter, venue, token, and vault authorization;
- compromised venues, adapters, tokens, keepers, RPC providers, APIs, frontends, and MCP clients;
- compromised pause guardian or registry governor, including role-confusion and timelock-bypass attempts;
- keeper griefing, racing, and surplus extraction;
- AMM price and liquidity manipulation;
- yield-vault loss, oracle exposure, redemption liquidity, and exposure caps;
- classic-asset freeze and clawback behavior; and
- frontend and build supply-chain risk.



#### Completion evidence

- [ ] The published threat model identifies at least 45 threats.
- [ ] Every threat has an identifier, component, impact class, mitigation, and either a named test or an explicit residual-risk statement.
- [ ] All 13 numbered invariants in architecture section 7 map to at least one test identifier.
- [ ] Testnet, mainnet-launch, and post-award trust assumptions are documented separately.
- [ ] Default mode and yield mode have separate trust and loss-surface statements.
- [ ] One engineer outside the contract authors reviews and signs off the document with name and date.
- [ ] The core contract commit reviewed is the exact frozen Tranche 1 commit.



#### Tranche 1 budget reconciliation


| Deliverable                              | Budget         |
| ---------------------------------------- | -------------- |
| 1.1 Settlement                           | USD 13,200     |
| 1.2 Router, registry, and venue adapters | USD 11,600     |
| 1.3 Threat model                         | USD 6,400      |
| **Total**                                | **USD 31,200** |




## Tranche 2: Public testnet, open keeper network, and yield mode

**Status:** Planned  
**Period:** 2027-01-30 to 2027-03-01  
**Budget:** USD 32,400

**Distribution:** Launch the public testnet application. Publish the open-source keeper image and operator documentation for prospective keeper operators. Publish the MCP server for AI-assistant users.

### 2.1 Testnet deployment, open-source reference keeper, and monitoring

**Due:** 2027-02-12  
**Budget:** USD 12,600  
**Architecture:** [sections 6-7 and 9-10](./ARCHITECTURE.md#6-keeper-network-deployment-and-trust-assumptions)

#### Scope

Deploy all Tranche 1 contracts to Stellar testnet with the pause guardian, registry governor, and timelock configured. Publish a one-command Docker reference keeper that:

- reads events and verifies orders directly through the contract;
- quotes Aquarius and Soroswap;
- finds valid P2P matches;
- simulates every fill through Stellar RPC;
- submits through channel accounts; and
- can operate with the Seltra Order API disabled.

Run a public event indexer and monitors for keeper liveness/concentration, exact-`min_out` fills, integration activations, TTL headroom, allowance consistency, allowance-sync failures, and pause-guardian/registry-governor actions.

#### Completion evidence

- [ ] Testnet contract and integration addresses are published.
- [ ] At least 100 orders fill across both execution paths and both venues.
- [ ] At least 10 direct crosses complete, including at least two batched crosses.
- [ ] The public indexer lists all required transaction hashes.
- [ ] The keeper repository is public under an OSI-approved license.
- [ ] A fresh operator following only the published documentation executes at least five testnet fills, documented as a walkthrough.
- [ ] With the Order API disabled, the keeper discovers orders from contract events through a third-party RPC provider and executes at least 10 fills; configuration and hashes are published.
- [ ] The team keeper runs unattended for at least seven consecutive days with greater than 99% uptime on at least three channel accounts.
- [ ] For a documented sample of at least 20 fills with both venue quotes, the keeper selects the better eligible route.
- [ ] Adversarial tests cover every default-mode threat classified as loss of funds or forgone surplus, with each test named by threat ID.
- [ ] Guardian pause blocks fills and placements while cancellation and expiry continue to work, evidenced by transaction hashes.
- [ ] At least eight monitors are live, each with documented trigger, baseline, severity, and response.
- [ ] Across at least 10 injected events, the appropriate monitor triggers in under 60 seconds every time.
- [ ] The indexer exposes distinct keeper addresses, largest keeper share, last-fill age, and count of fills exactly at `min_out`.



### 2.2 Order pipeline: wallet flow, Order API, Order Builder, MCP server, and testnet application

**Due:** 2027-02-20  
**Budget:** USD 10,900  
**Architecture:** [sections 4.2, 5, and 9](./ARCHITECTURE.md#42-batch-placement-split-and-ladder)

#### Scope

Deliver:

- placement and cancellation through Stellar Wallets Kit `signTransaction`;
- pre-signing display of network, contract addresses, trustline/issuer status, allowance, and storage rent;
- a read-only REST and WebSocket Order API built from contract state and events;
- an Order Builder that compiles split and ladder intent into one bounded placement batch;
- application enforcement of the USD 10 minimum per order/slice;
- an MCP server that quotes, reads, and prepares unsigned placement/cancellation XDR; and
- a public Next.js testnet application with self-fill.



#### Completion evidence

- [ ] Freighter, Lobstr, xBull, Albedo, Hana, and a Ledger device each complete at least three placements and one cancellation through Stellar Wallets Kit.
- [ ] Transaction hashes and a tested wallet-support matrix are published.
- [ ] A placed order appears on the WebSocket feed in less than two seconds after ledger close across at least 50 measured placements.
- [ ] An order becomes flagged within one ledger when maker balance or allowance falls below its input size.
- [ ] A ladder of at least 20 orders is placed in one transaction.
- [ ] A split of at least 10 slices is placed in one transaction.
- [ ] At least 50% of each test batch is filled by keepers.
- [ ] One slice is cancelled individually and the remainder is cancelled in one transaction.
- [ ] Allowance behavior after each cancellation matches architecture section 2.2, including the rejected-update path.
- [ ] At least 10 distinct testnet accounts use the public application to place, observe, and cancel an order.
- [ ] At least five self-fills are documented with transaction hashes.
- [ ] An AI assistant connected to the MCP server prepares at least three orders and one cancellation that users sign in their own wallets.
- [ ] The MCP process holds no user or keeper signing key, verified by configuration and code review.
- [ ] Every tested order or slice below USD 10 at the current quote is rejected before signing.
- [ ] An OpenAPI specification is published.
- [ ] The frontend builds reproducibly with pinned dependencies and checks network passphrase and contract addresses before signing.



### 2.3 DeFindex yield adapter, opt-in

**Due:** 2027-03-01  
**Budget:** USD 8,900  
**Architecture:** [section 3.6 and section 7](./ARCHITECTURE.md#36-yield-adapter)

#### Scope

Implement and pin the yield adapter against a concrete DeFindex vault deployment:

- deposit during placement with shares minted to the maker;
- bounded share allowance for settlement;
- redemption in the same invocation as the DEX fill;
- keeper-side simulation of the real redemption and Blend unwind path;
- per-vault exposure cap measured in underlying-asset base units of committed order principal, with atomic increment/decrement;
- registry acceptance only for active, single-asset, non-upgradable vault deployments;
- a registry-pinned rounding tolerance no greater than one underlying base unit;
- accrued yield returned entirely to the maker; and
- application disclosure of fees, roles, risk, and a `waiting on liquidity` state.



#### Completion evidence

- [ ] At least 15 yield-mode testnet orders deposit into the allowlisted vault at placement and settle through an atomic redemption/fill, with hashes published.
- [ ] At least three of those orders settle within 10 ledgers of placement.
- [ ] In every tested fill, `min_out` holds and value redeemed above the order's principal returns to the maker, not the keeper.
- [ ] No Seltra contract holds vault shares between invocations; share balances are checked before and after every tested fill.
- [ ] Cancelling a yield-mode order leaves shares in the maker's account and applies the share-allowance reconciliation rules.
- [ ] Tests use the exact allowlisted DeFindex and Blend contract versions and record their interfaces and hashes.
- [ ] Activation evidence proves the vault instance uses the proposed direct `Wasm(hash)`, not an external executable reference, and review of that exact WASM finds no reachable executable-update path.
- [ ] Constrained redemption liquidity makes the keeper skip submission and the application show `waiting on liquidity`.
- [ ] A redemption shortfall greater than the active vault's maximum one-base-unit rounding tolerance reverts atomically with shares and order unchanged.
- [ ] Returned yield increases the maker's underlying balance by the exact computed amount before the trading-surplus split.
- [ ] A deposit above the exposure cap reverts; successful placement increments and fill/cancellation decrements the checked underlying-base-unit exposure counter by the order's `amount_in`.
- [ ] Paused, multi-asset, or upgradable vault deployments are rejected.
- [ ] A simulated vault-value loss causes the fill to revert with the maker's shares unchanged.
- [ ] Architecture invariants 12 and 13 map to named tests.
- [ ] Yield-adapter line coverage is greater than 90%.



#### Exit gate

The yield adapter and its pinned integration contracts are frozen for independent review on the exact Tranche 2 commit.

#### Tranche 2 budget reconciliation


| Deliverable                               | Budget         |
| ----------------------------------------- | -------------- |
| 2.1 Testnet, keeper, and monitoring       | USD 12,600     |
| 2.2 Wallet/order pipeline and application | USD 10,900     |
| 2.3 Yield adapter                         | USD 8,900      |
| **Total**                                 | **USD 32,400** |




## Tranche 3: Mainnet and checkpoint-by-checkpoint verification

**Status:** Planned  
**Period:** 2027-03-02 to 2027-04-13  
**Budget:** USD 30,400

**Distribution:** Launch the production application. Submit listing/integration requests to the wallet applications and protocols integrated in Tranche 2, including Aquarius, Soroswap, and DeFindex. Name two outside keeper operators and publish their keeper addresses at launch.

### 3.1 Hardened contracts and mainnet deployment

**Due:** 2027-03-24  
**Budget:** USD 12,400  
**Architecture:** [sections 3, 6-7, and 10](./ARCHITECTURE.md#3-contracts-and-core-components)

#### Scope

Remediate findings from the independent security review obtained through Stellar's security-review program for SCF-funded protocols or an equivalent independent firm. The review itself is not funded by this award.

The review targets:

- the core contract commit frozen at the end of Tranche 1; and
- the yield-adapter commit frozen at the end of Tranche 2.

After remediation, deploy all contracts to Stellar mainnet with verified source and published addresses. Activate the two venue adapters and first DeFindex vault through the timelock, publish the exposure cap, and configure the separate pause-guardian and registry-governor multisigs.

#### Completion evidence

- [ ] A public hardening report maps every finding to severity, fixing commit, and covering test.
- [ ] All critical and high findings are closed before deployment.
- [ ] Every medium finding is closed or documented with a specific residual-risk rationale.
- [ ] The complete unit, property/fuzz, adversarial, authorization, resource, integration, and yield fallback suites pass on hardened code.
- [ ] Tranche 1 and Tranche 2 coverage thresholds are maintained.
- [ ] Mainnet addresses and deployment configuration are published.
- [ ] The production build is reproducible from the tagged source.
- [ ] An independent party rebuilds the settlement WASM and matches the on-chain hash.
- [ ] An independent party resolves and matches the on-chain direct-WASM hashes for the router, registry, and Seltra-owned adapters and confirms that none uses a mutable executable reference.
- [ ] Aquarius and Soroswap adapters and the DeFindex vault activate on mainnet only after their timelocks, with transaction hashes published.
- [ ] The per-vault exposure cap is readable on-chain.
- [ ] The pause guardian and registry governor are separate explorer-verifiable 2-of-3 classic-account multisigs; each has three hardware-wallet signers and at least one named signer outside the team.



### 3.2 Production application, wallet integration, and mainnet yield mode

**Due:** 2027-04-04  
**Budget:** USD 9,500  
**Architecture:** [sections 5-6 and 9](./ARCHITECTURE.md#5-keeper-economics)

#### Scope

Release the mainnet application with:

- Stellar Wallets Kit on mainnet;
- default and yield-mode placement;
- split and ladder creation;
- order API, WebSocket feed, and MCP server configured for mainnet;
- the Tranche 2 monitoring set running against mainnet; and
- exercised incident, settlement pause, adapter/vault pause, RPC failover, and migration runbooks.



#### Completion evidence

- [ ] At least 25 mainnet fills complete across both execution paths and both venues.
- [ ] The total includes at least three direct crosses and at least five yield-mode fills.
- [ ] Orders are placed through at least four wallet applications, including Lobstr and xBull.
- [ ] Required hashes are published by the public indexer.
- [ ] A ladder of at least 10 orders is placed in one mainnet transaction.
- [ ] One ladder slice is cancelled independently, and allowance behavior matches the remaining live orders and architecture section 2.2.
- [ ] A mainnet drill pauses and unpauses fills and placements, executes a cancellation during the pause, and pauses/unpauses the vault or its registry entry, with hashes published; it also proves the guardian cannot change an allowlist or cap.
- [ ] Operational runbooks are published after the drill.
- [ ] At least eight documented monitors are live against mainnet.
- [ ] The public indexer serves mainnet fills, distinct keeper addresses, largest keeper share, last-fill age, and fills exactly at `min_out`.



### 3.3 Production keeper network and independent operators

**Due:** 2027-04-13  
**Budget:** USD 8,500  
**Architecture:** [section 6](./ARCHITECTURE.md#6-keeper-network-deployment-and-trust-assumptions)

#### Scope

Run production keeper infrastructure with:

- a self-hosted Stellar RPC node and provider fallback;
- channel accounts and alerting;
- an unprivileged team backstop keeper; and
- two keepers operated by named parties outside the team from the public image.

Each outside operator publishes its keeper address and discloses any related-party relationship.

#### Completion evidence

- [ ] Two named outside operators each execute at least 20 testnet fills from their own keeper addresses.
- [ ] Each outside operator executes at least five mainnet fills from its own keeper address.
- [ ] Operator statements, addresses, and transaction hashes are public.
- [ ] During the first 10 days after mainnet deployment, the indexer reports at least three distinct keeper addresses with successful fills.
- [ ] The largest keeper's share over the same period is reported publicly.
- [ ] Team backstop keeper uptime is greater than 99.5% over the same period.
- [ ] RPC failover is exercised at least once and recorded in an incident log.
- [ ] Mainnet operator documentation is current.
- [ ] Each outside operator reproduces the path from zero configuration to its first mainnet fill in less than one day, with confirmation linked.



#### Tranche 3 budget reconciliation


| Deliverable                          | Budget         |
| ------------------------------------ | -------------- |
| 3.1 Hardening and mainnet deployment | USD 12,400     |
| 3.2 Production application and yield | USD 9,500      |
| 3.3 Independent keeper network       | USD 8,500      |
| **Total**                            | **USD 30,400** |




## Final completion definition

The SCF Build is complete only when all of the following are true:

- [ ] All three tranche exit gates and their evidence requirements are complete.
- [ ] The deployed settlement WASM matches the independently reproduced immutable build.
- [ ] The architecture, threat model, contract interfaces, addresses, test mappings, and runbooks describe the deployed system rather than a planned variant.
- [ ] Mainnet users can place, observe, cancel, self-fill, and receive third-party keeper fills without trusting Seltra with custody.
- [ ] Aquarius and Soroswap routes, direct P2P crosses, and opt-in DeFindex yield fills are evidenced on mainnet.
- [ ] Cancellation remains available during protocol pause.
- [ ] No Seltra service or MCP component holds a user key.
- [ ] At least two outside keeper operators have successfully filled mainnet orders from their own published addresses.
- [ ] The professional user testing required at mainnet launch is complete and its actionable findings are triaged.
- [ ] All delivery budgets reconcile to USD 94,000 and no marketing, promotion, or external audit cost is charged to the award.