# Canton Network Fiat Connector - Development Fund Proposal

**Author:** Andre Straube (PagFinance)

**Status:** Draft

**Created:** 2026-06-15

**Label:** financial-workflows-composability

**Champion:** need Champion

---

## Abstract

This proposal funds an open-source reference implementation of a fiat connector for the Canton Network: a reusable pattern that moves value in both directions between on-ledger token settlement and an off-ledger bank rail. It is proven through the first BRL (Brazilian Real) corridor on the network, settled via PIX, Brazil's central-bank instant payment system, running off-ramp (token to fiat) and on-ramp (fiat to token).

The deliverable is a rail-agnostic Daml package built on the Canton Token Standard (CIP-56), plus an off-ledger connector and an integration blueprint that let any Canton participant operating an enterprise validator node connect on-ledger settlement to an off-ledger instant-payment, wire, ACH, or SWIFT rail with atomic delivery-versus-payment guarantees. Nothing in the on-ledger contracts is specific to PIX or to Brazil: the templates describe a generic fiat settlement (an amount in any currency, a hashed payout destination, a settlement reference the rail returns on confirmation), and all rail-specific knowledge lives in the off-ledger connector. PagFinance is the first implementer and operates the live BRL corridor via PIX, but the contract templates, the settlement pattern, the connector, and the documentation are released as a common good under Apache-2.0 for the whole ecosystem.

The value to Canton is a repeatable, bidirectional pattern for fiat connectivity and the network's first connection to the Brazilian market, where PIX serves over 180 million users and moves more than US$5T annually.

---

## Specification

### 1. Objective

Canton has no documented, reusable bridge between on-ledger token settlement and a genuinely off-ledger fiat rail: a wire, ACH file, or SWIFT message that clears in a bank's own system rather than as a token on Canton. What Canton documents instead is the opposite move: tokenize the cash leg through a stablecoin, tokenized deposit, or regulated issuer (like ClearToken's CT Register), then settle both legs atomically on-ledger via CIP-56's transfer/DvP standard. That path is mature and audited, but it sidesteps the problem rather than solving it. Every team that wants a fiat corridor backed by an actual bank rail, not a tokenized proxy for one, starts from zero. There's no reference for how to structure the escrow across that boundary, how to guarantee the token leg and the off-ledger payment leg cannot both fail open, or how to reconcile an off-ledger payment confirmation against an on-ledger state transition.

The single objective of this proposal is to deliver, in the open, a reference fiat connector that closes this gap in both directions, off-ramp and on-ramp, with the first BRL/PIX corridor as its proving ground. Because the on-ledger contracts are rail-agnostic, other builders reuse the same templates and blueprint to build corridors for other currencies and rails (MXN over SPEI, USD over ACH or wire, EUR over SEPA) by writing only the off-ledger rail adapter.

### 2. Implementation Mechanics

The core is a propose-accept escrow contract written natively in Daml against the CIP-56 interfaces (Holding, TransferInstruction, Allocation). The on-ledger templates carry only generic fiat fields: a fiat amount, an ISO currency code, a hashed payout destination (PIX key, CLABE for SPEI, IBAN, ACH account), and a settlement reference the rail returns on confirmation. No rail-specific logic ever touches the ledger.

**Off-ramp (token to fiat).** The party holding tokens exercises a request that locks the tokens into a contract-controlled escrow via a CIP-56 Allocation and creates an off-ramp request signed by both the requesting party and the operator. Neither side can unilaterally alter or cancel it. Settlement uses a strict ordering that removes open-fail risk: the fiat payment executes off-ledger first; only after the rail confirms does the operator exercise the on-ledger settle, which releases the escrowed tokens to treasury and records an immutable receipt carrying the rail's settlement reference. If the payment fails, the operator exercises refund and the tokens return intact to the requester. An expiry choice guarantees the escrow can never be stranded. At no point can a counterparty hold both the tokens and the fiat.

**On-ramp (fiat to token).** The inverse corridor: the connector raises an off-ledger collection on the rail (a PIX charge, a pull, an inbound wire reference); only after the rail confirms the funds have cleared does the operator release tokens from treasury to the participant's party on-ledger. The same escrow, ordering, and reconciliation discipline applies in reverse, so the participant cannot receive tokens before the fiat has cleared, and the operator cannot strand a paid-in participant without tokens.

A backend connector consumes the ledger transaction stream, detects requests, validates them, drives the off-ledger payment or collection in either direction, and exercises the resulting choice. Idempotency is keyed on the contract id; the last processed ledger offset is persisted so the stream resumes without loss after a restart. Reconciliation is triple: ledger state, internal records, and the payment provider's statement.

The escrow logic, the state machine, and a five-scenario test suite (settle, refund, expiry, access control, input validation) are already validated on a local Daml sandbox, with an end-to-end script demonstrating tokens moving from requester through escrow to treasury on settle. The funded work migrates this from the sandbox mock token to CIP-56 on DevNet, then TestNet, then a production corridor on MainNet; generalizes the validated off-ramp into the bidirectional on-ramp/off-ramp connector; and adds the blueprint.

### 3. Architectural Alignment

The implementation programs against the Canton Token Standard (CIP-56), so it composes with USDCx and any standard-compliant token rather than forking a bespoke asset model. The escrow uses the standard Allocation mechanism for delivery-versus-payment, consistent with how the network expresses atomic settlement.

The design targets enterprise validator nodes and is single-participant for development but inherently multi-participant for production: because requests are contracts whose parties can be hosted on different participants, a counterparty running its own enterprise node signs with its own key on its own node, with no key custody by the operator. This aligns with Canton's privacy and sovereignty model and matches how institutions actually deploy: each participant runs its own node, holds its own keys, and connects to the shared synchronizer rather than to a custodial intermediary. The work falls under the financial-workflows-composability priority and contributes a reference implementation, an eligible category of common good.

### 4. Regulatory Boundary and Reusability

The off-ledger leg of any fiat corridor touches regulated payment infrastructure, and that boundary shapes how reusable the connector can be for a third party.

PagFinance operates the BRL corridor under a PSAV authorization (Brazilian Central Bank Digital Asset Service Provider, intermediation-only modality). If that authorization is in place, PagFinance can offer the operational infrastructure of the BRL corridor as a plug-and-play service to other participants on the network who want BRL connectivity without standing up their own licensed rail relationship: they integrate against the reference templates and consume the corridor as a service.

If a given participant cannot use PagFinance's licensed rail (a different jurisdiction, a different regulatory posture, or no agreement in place), the open-source artifact still delivers full value: the participant forks or replicates the connector and the blueprint and connects it to its own licensed rail relationship. The reusable common good is the on-ledger escrow, the settlement pattern, and the connector architecture, which are rail- and license-independent by design. The licensed operational service is an additional convenience PagFinance can offer where regulation permits, not a dependency the reference implementation imposes. This is a limitation of payment regulation, not of the reference design: licensing of fiat rails is jurisdiction-specific and cannot be open-sourced away.

### 5. Backward Compatibility

No backward compatibility impact. This is additive: a new connector pattern and new contract templates. It introduces no changes to existing protocol components, tokens, or workflows, and depends only on the published CIP-56 interfaces.

---

## Milestones and Deliverables

### Milestone 1: CIP-56 rail-agnostic core (off-ramp and on-ramp) on DevNet

- **Estimated Delivery:** ~7 weeks from start
- **Focus:** Migrate the validated sandbox logic to the Canton Token Standard and deploy on DevNet. Remove the mock token; the escrow becomes a CIP-56 Allocation. Generalize the validated off-ramp into both directions so the same core supports off-ramp (token to fiat) and on-ramp (fiat to token).
- **Deliverables / Value Metrics:** Public Apache-2.0 Daml package with rail-agnostic off-ramp and on-ramp templates and choices; full test suite passing against CIP-56 for both directions; a completed off-ramp and a completed on-ramp against a test token on DevNet that any reviewer or ecosystem team can run and verify from the open repository.

### Milestone 2: Open connector and integration blueprint

- **Estimated Delivery:** ~6 weeks after Milestone 1
- **Focus:** Publish the open backend connector that consumes the ledger stream, drives the off-ledger payment or collection in either direction, and exercises settle/refund/expire, with idempotency and offset-based recovery, plus the written blueprint other teams follow to build their own fiat corridors on other rails.
- **Deliverables / Value Metrics:** Public connector reference code with a documented rail-adapter interface; the integration blueprint document; a worked triple-reconciliation example; demonstrated by at least one external party or reviewer running the full off-ramp and on-ramp cycle on DevNet from the blueprint, including the forced-failure refund path.

### Milestone 3: External security audit

- **Estimated Delivery:** commissioning ~1 week after Milestone 2; accepted report ~4 weeks after commissioning
- **Focus:** Commission an independent, reputable security firm to audit the Daml package and the connector, covering the escrow state machine, the settlement ordering that prevents both-legs-fail-open, the access-control boundary, and the reconciliation logic. The allocation is split so the audit is funded ahead of the spend rather than reimbursed after it (see Funding).
- **Deliverables / Value Metrics:** A signed engagement naming the audit firm (Milestone 3a), followed by a published audit report covering the on-ledger templates and the connector (Milestone 3b). All findings classified as Critical or High have a verifiable patch merged into the public main branch before Milestone 3b is accepted.

### Milestone 4: Live BRL corridor on MainNet and public enablement

- **Estimated Delivery:** ~8 weeks after Milestone 3
- **Focus:** Operate the first BRL/PIX corridor on MainNet, off-ramp and on-ramp, and publish a public technical write-up so other teams can replicate the pattern for other currencies and rails.
- **Deliverables / Value Metrics:** A documented off-ramp and on-ramp settling on MainNet against the live PIX rail; a public post-implementation report; the reference package positioned as the starting point for additional fiat corridors, with the BRL corridor as evidence the bidirectional pattern works against a real central-bank rail.

### Milestone 5: Institutional Adoption Bounty

- **Estimated Delivery:** up to ~12 months after Milestone 4
- **Focus:** Drive verifiable institutional adoption of the connector, whether for institutions deploying Canton fiat connectivity for the first time or for participants replicating the pattern on their own rail. 
- **Deliverables / Value Metrics:** Verification that the connector was successfully integrated, in a production or staging environment, by financial institutions (defined as finance organizations of 100+ employees) running their own enterprise nodes. Structured as a performance-based adoption bounty so the Foundation takes on zero upfront risk for this allocation.

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- Deliverables completed as specified for each milestone, all open-source under Apache-2.0 in a public repository
- Demonstrated functionality: a reviewer or external team can independently run each milestone's artifacts and complete both an off-ramp and an on-ramp cycle
- Documentation and knowledge transfer: the blueprint enables a third party to build a fiat corridor on its own rail without dependency on PagFinance's private infrastructure
- Ecosystem value: the reference package is reusable by any participant running an enterprise node, the bidirectional pattern is proven, and the live BRL corridor demonstrates the first fiat connection of its kind on Canton, opening the Brazilian market to the network
- Adoption (Milestone 5 only, performance-based): verified institutional integrations as defined in the milestone, with the Foundation bearing zero upfront risk

---

## Funding

**Total Funding Request:** 3.600.000 CC

The request is sized against ecosystem value, not engineering effort alone. The connector unlocks an entire capability category for Canton, fiat moving both into and out of the network against real bank rails, and proves it on the first corridor into a market of more than 180 million users moving US$5T annually. CC amounts assume the Foundation's reference valuation and are subject to the volatility stipulation below.

### Payment Breakdown by Milestone

| Milestone | Focus | Funding |
| :--- | :--- | :--- |
| Milestone 1 | CIP-56 rail-agnostic core, off-ramp and on-ramp, on DevNet | 1.350.000 CC |
| Milestone 2 | Open connector and integration blueprint | 1.350.000 CC |
| Milestone 3a | External security audit, commissioning tranche | 300.000 CC |
| Milestone 3b | External security audit, accepted report | 400.000 CC |
| Milestone 4 | Live BRL corridor on MainNet and public enablement | 1.800.000 CC |
| Milestone 5 | Institutional Adoption Bounty (performance-based) | 1.000.000 CC |
| **Total** | | **6.200.000 CC** |

Engineering and corridor delivery (Milestones 1, 2, 4) total 2.250.000 CC. The external audit (Milestone 3) totals 350.000 CC. The adoption bounty (Milestone 5) is 1.000.000 CC and is strictly performance-based: it is paid only on verified institutional integrations, so the Foundation takes on zero upfront risk for that allocation.

**Audit funding sequence.** Independent security firms bill on engagement, not on delivery, so the audit allocation is structured to fund the spend rather than reimburse it after the fact. Milestone 3a (150.000 CC) is released when the engagement is signed and the audit firm is named, which is the deliverable that unblocks the audit and provides the cash to commission it. Milestone 3b (200.000 CC) is released on the published report, with every finding classified as Critical or High verifiably patched and merged into the public main branch. This sequencing means the project does not have to front the audit cost out of pocket and wait for reimbursement.

### Volatility Stipulation

The engineering and corridor work (Milestones 1 to 4) is expected to run under 6 months. Should that timeline extend beyond 6 months due to Committee-requested scope changes, any remaining milestones will be renegotiated to account for USD/CC price volatility. The Milestone 5 adoption bounty runs on a longer adoption horizon by design and is paid per verified integration as it occurs.

Separately from any timeline extension, either party may request a good-faith review of the unpaid milestone amounts if the CC reference valuation moves materially between approval and payout. As a guideline, a move of more than 20% from the valuation used to set this request, in either direction, may trigger such a review for milestones not yet paid. Any adjustment is by mutual agreement between PagFinance and the Tech & Ops Committee and applies only to milestones that have not yet been accepted and paid; amounts already disbursed are not clawed back or topped up. This protects the project against a sharp CC decline eroding the real value of the remaining work, and protects the Foundation symmetrically against a sharp appreciation.

---

## Co-Marketing

Upon release, PagFinance will collaborate with the Foundation on:

- Announcement coordination for the first BRL corridor on Canton, covering both on-ramp and off-ramp
- A technical blog post and case study on the rail-agnostic fiat connector pattern and the bidirectional DvP guarantee
- Developer enablement so other teams can build corridors for additional currencies and rails

---

## Motivation

Brazil's PIX serves over 180 million users and is among the highest-volume instant-payment systems in the world (more than US$5T annually). No fiat connector to BRL exists on Canton today, and more broadly there is no reusable reference for connecting any off-ledger bank rail to Canton settlement in either direction.

This work benefits the entire portion of the ecosystem that needs fiat on-ramps and off-ramps: stablecoin issuers, payment apps, treasury operators, and any institutional participant settling tokens against fiat. The reference package and blueprint are directly reusable by every team building a fiat corridor, and the BRL corridor itself connects the network to a major new market. The strategic importance is that bidirectional fiat connectivity is a precondition for real-world payment adoption: value has to get into the network and back out against the rails people actually use, and Canton currently has no open pattern for it.

---

## Rationale

This is a reference implementation, not a one-off integration, because the gap is shared across the ecosystem and the value compounds when the pattern is reusable. Making the on-ledger contracts rail-agnostic is what turns a single BRL integration into a template for every fiat corridor: a new rail needs only a new off-ledger adapter, not a new ledger model. Building on CIP-56 rather than a bespoke asset model ensures composability with USDCx and any compliant token, and extends existing ecosystem standards rather than replacing them. Targeting enterprise nodes and sovereign key custody aligns the pattern with how institutions actually join Canton, and structuring the adoption allocation as a performance-based bounty puts the network's risk on outcomes, not promises.
