# Issuer Metrics API

## Development Fund Proposal

| Field | Value |
| :---- | :---- |
| Organization | Kaiko |
| Author / Primary Contact | Charles Desmonty, Kaiko ([@Desmonty-Kaiko](https://github.com/Desmonty-Kaiko)) |
| Status | Draft |
| Created | 2026-10-22 |
| Proposal Type | RFP-aligned |
| RFP / Roadmap Area | Financial Markets, Standards & Verification: [RFP 11, Public verifiability](../../2026-2028-strategic-roadmap.md#financial-markets-standards--verification) |
| Champion | Charles Desmonty, Kaiko ([@Desmonty-Kaiko](https://github.com/Desmonty-Kaiko)), Financial Workflows & Composability SIG |
| Total Funding Request | Up to 8,791,667 CC: 7,491,667 CC for Milestones 1 to 4 and up to 1,300,000 CC in adoption payments ($1,055,000 at a $0.12 baseline, each payment adjusted to a 7-day TWAP) |
| Project Duration | 20 months: production at DA Utility by month 8, then 12 months of maintenance and adoption |
| Label | financial-workflows-composability (SIG: Financial Workflows & Composability) |

---

## Abstract

Canton keeps holdings and transfers private, so most Canton assets have no public supply, holder or activity figures, the figures on which tokenised assets are evaluated. An issuer that wants to publish them has no standard tooling: each data platform integrates each registry separately, and issuers' compliance teams need a record of what is published, and to whom, before they approve.

Kaiko would build the Issuer Metrics API, open-source tooling (Apache-2.0) that issuers deploy to publish a fixed catalogue of metrics for their Canton instruments: total supply, holders, mints, burns, transfers and an instrument descriptor. The issuer's system pushes daily figures. An operator UI records consent per metric and audience, revocation and an audit log, and issues API keys scoped per consumer. Consumers read one versioned API, the same on every deployment.

DA Utility is the design partner and the first production deployment. The grant funds the on-premise release, its rollout and a year of maintenance: 7,491,667 CC ($899,000 at $0.12 per CC), plus up to 1,300,000 CC paid per issuer publishing and per consumer integrating in production.

---

## RFP Alignment and Ecosystem Need

**RFP.** RFP 11 seeks proposals that progress Canton toward public verifiability of key metrics, and its first approach is standardised tooling for asset issuers, decentralised issuers included, to publish public aggregates of activity and value. This proposal is that tooling: one metric catalogue, one API specification and one conformance suite, shipped as software any issuer runs. The roadmap lists aggregate metrics with selective disclosure among the options for public verification of private transactions; consent per metric and per audience is that selective disclosure.

**Trust.** RFP 11 notes that a market may not trust an aggregate published by an issuer. The tooling narrows that gap without disclosing transactions: every deployment applies the same definitions; the party that computes and pushes the figures answers for them; total supply, name, symbol and decimals can be checked against the registry's Token Metadata API; restatements stay visible in the history; and the read-only part of the conformance suite runs against any public endpoint. The other approaches RFP 11 lists, TEE disclosure and proofs of aggregate data, can later attest figures computed under the same definitions.

**Secondary fit.** RFP 20 covers network activity reporting and deployable on-premise or hosted visibility tools. The four data questions it asks are answered under [Data and privacy](#data-and-privacy).

**Who needs it and why.**

- **Issuers and registries, DA Utility first.** They publish once, approve each metric and audience, revoke at any time and keep a record for compliance review. On premise, the figures stay inside the issuer's perimeter and DA keeps its client relationships.
- **Data consumers: explorers, dashboards, index and data providers.** One API and one set of definitions across issuers, in place of one integration per registry.
- **The network.** Canton assets appear in the cross-network figures on which tokenised assets are evaluated.

**Adoption.** DA Utility's deployment goes to production in Milestone 3, with its first clients publishing. Adoption payments are made only per issuer publishing in production and per consumer integration.

---

## Specification

### Objective

**Problem.** A Canton token issuer has no standard way to publish metrics for its instruments under its own control, and data platforms have no common definitions or API to read them.

**Intended outcome.** A release that any issuer deploys, on premise or hosted, to publish a fixed catalogue of metrics through one versioned API with consent per metric and audience; DA Utility in production with its clients publishing; consumers reading the same API on every deployment.

### Implementation Mechanics

Each deployment has five parts.

```mermaid
flowchart LR
    A["Issuer system"] -->|"daily figures"| B["Push API"]
    B --> C[("Metrics store")]
    D["Operator UI: consent, revocation, audit log, API keys"] --> C
    C --> E["Versioned API"]
    E -->|"public metrics"| F["Consumers"]
    E -->|"restricted metrics, with API key"| F
```

- **Push API.** The only way data enters. The issuer's own system computes the figures and posts them against the metric catalogue.
- **Metrics store.** Every accepted value and its history, with retention settings, backups and export.
- **UI.** Instruments, consent per metric and audience, revocation and an audit log. Two roles: the operator (such as DA) and one approver per instrument.
- **API keys.** Created per consumer, scoped to instruments and metrics, rotated or revoked at will, with usage logs and rate limits.
- **Versioned API.** One OpenAPI specification for every deployment. Each response states whether a figure is published, opted out or has no data.

#### Metric catalogue v1

Five metrics and an instrument descriptor are defined, so every deployment publishes the same data. Definitions are written against the token standard V2 account model (CIP-0112), where mints and burns are transfer legs from and to the instrument admin's special accounts, and cover V1 (CIP-0056) assets through the V1 rules in the table. They are stated in terms of the holdings and the event log of the instrument admin. The party that computes and pushes the figures answers for their accuracy: DA for DA Utility clients, the issuer otherwise. Total supply, name, symbol and decimals match the Token Metadata API, so a consumer can cross-check them against the registry's endpoint.

| Metric | Definition | Unit | Window |
| :- | :- | :- | :- |
| `total_supply` | Units outstanding at the window close | Instrument units | Daily cut at 00:00 UTC, hourly optional |
| `holders` | Accounts with an owner holding a positive balance at the window close. The instrument admin's special accounts are excluded, and a V1 holding counts as its owner's basic account. | Count | Daily cut |
| `mints` | Issuance legs from the instrument admin's special accounts into owner accounts, and units issued. In V1, the issuer's issuance records. | Count and units | 24 hours to the cut |
| `burns` | Redemption or destruction legs from owner accounts into the instrument admin's special accounts, and units removed. In V1, the issuer's redemption records. | Count and units | 24 hours to the cut |
| `transfers` | Transfer legs between owner accounts, netted legs and moves between one owner's accounts included; merges, splits, mints and burns excluded. A V1 transfer counts as one leg. | Count | 24 hours to the cut |
| `instrument` | Instrument ID (admin party and ID), name, symbol, decimals, issuer, registrar, asset class, token standard versions supported | Descriptor | On change |

Definitions are versioned: a published definition keeps its meaning, and a change ships as a new catalogue version.

#### Push API and consumer API

Two small surfaces: one the issuer's system writes to, one consumers read from.

| Push endpoint | Purpose |
| :- | :- |
| `POST /v1/ingest/records` | Metric records for one or more instruments |
| `POST /v1/ingest/instruments` | Create or update instrument descriptors |
| `POST /v1/ingest/backfill` | History in bulk, validated like live records |
| `GET /v1/ingest/status` | Last accepted window per instrument and metric |

- **Idempotence.** The same instrument, metric and window sent twice is stored once; a different value is a restatement, kept in the history.
- **Validation.** Known metric and unit, window aligned to the UTC cut, non-negative values.
- **Conformance.** A conformance suite checks that a deployment implements the specification: response shapes, publication states, key scoping, idempotent writes, restatements and validation errors. Operators run it against their own deployment, and its read-only part runs against any public endpoint.

| Consumer endpoint | Returns |
| :- | :- |
| `GET /v1/instruments` | Published instruments, with descriptors |
| `GET /v1/instruments/{id}/metrics` | Latest value and state of each metric |
| `GET /v1/instruments/{id}/metrics/{metric}/history` | Series between two dates |
| `GET /v1/status` | Specification version, health |

Public metrics need no key. Everything else takes an API key in a header, with a rate limit per key. A latest-values response:

```json
{
  "instrument": "example-usd",
  "spec_version": "1.0",
  "metrics": [
    {"metric": "total_supply", "window_end": "2026-09-30T00:00:00Z", "value": "12500000.00", "state": "published"},
    {"metric": "holders", "window_end": "2026-09-30T00:00:00Z", "value": null, "state": "opted_out"}
  ]
}
```

#### Deployment

The grant funds the on-premise release. Managed hosting runs the same release and is offered by Kaiko under contract, outside the grant. Both look the same to issuer operators and to consumers and differ only in data governance.

**On premise, funded by this grant.**

- Push API, metrics store, UI and versioned API ship as container images and a Helm chart, for the issuer's servers or cloud account. For DA Utility, the deployment sits inside DA's perimeter, next to its data service.
- The issuer alone holds the figures, the consent records and the keys. Kaiko has no access, and the only terms between them are the Apache-2.0 licence.
- The issuer applies releases, runs backups and answers for availability. The status endpoint reports the running version, and the project supports the two latest minor versions.
- History stays in the issuer's database for as long as the issuer keeps it. Scheduled backups and a full export ship with the release.
- The issuer's security team reviews the open code and its own deployment, with the external audit report and the security documentation pack funded by this grant.

**Managed, outside this grant.** Kaiko hosts one isolated tenant per issuer and holds its figures as a processor, under a service and data agreement: processing only for the issuer's API, retention and deletion periods, security obligations and audit rights, an availability SLA, and export and deletion on exit. For DA Utility, the agreement is with DA and covers its clients. The issuer stays the golden source: only its system pushes and only its approvers publish. Any other provider can offer the same hosting.

#### Data and privacy

- **Data required.** Daily aggregates per instrument from catalogue v1, and instrument descriptors. No party identifiers, positions or transactions.
- **Data class.** Application-provided. The issuer's system computes the figures from its own records and pushes them; the release reads nothing node-local or publicly observable.
- **Protocol metadata.** None is used, so changes to what the Mediator or other protocol messages expose do not affect the release.
- **Privacy, access controls and selective disclosure.** Nothing is published without the approver's consent, per metric and per audience. Public metrics are readable without a key; restricted metrics only with a consumer key scoped to instruments and metrics. Consent and keys can be revoked at any time, and the audit log records every consent change and key event. On premise, the figures leave the issuer's infrastructure only through its own API.

### Architectural Alignment

- **Token standard.** The definitions follow the V2 account model of CIP-0112 and map V1 (CIP-0056) holdings onto it, so one catalogue covers both versions. Instruments are identified by their instrument ID (admin party and ID), which both versions use. Total supply, name, symbol and decimals follow the Token Metadata API, which CIP-0112 keeps at v1.
- **Privacy model.** Holdings and transfers stay private. Only the aggregates an issuer approves leave its perimeter, through its own API.
- **Footprint.** The release is an off-ledger service next to the issuer's existing systems. It needs no Daml package, no ledger writes and no changes to Splice or Canton.
- **Roadmap.** It provides the aggregate-metrics path the roadmap lists for public verification of private transactions.

**Related work.**

| Existing work | What it does | Relationship |
| :- | :- | :- |
| Canton Asset Data Feed, Fairmint ([#890](https://github.com/canton-foundation/canton-dev-fund/pull/890), open, RFP 11) | Open, pull-based feed format for asset-level metrics, with daily declarations on Canton | Complementary layer. CADF defines how public metrics are published and anchored; the Issuer Metrics API is the issuer-side system that ingests figures, applies consent and serves public and keyed endpoints. Catalogue v1 is offered to CADF as its tokenised-asset profile, and CADF's reference publisher can read a deployment's versioned API as its data source. |
| Token Metadata API (CIP-0056, kept by CIP-0112) | Current instrument facts per registry, including total supply where the registry reports it | Same identifiers and definitions. The Issuer Metrics API adds history, holders, flows, consent and keys. |
| DA Utility | Registry service for assets that DA's clients issue on Canton | Design partner and first production deployment. DA's data service pushes its clients' figures to DA's own deployment. |

### Backward Compatibility

*No backward compatibility impact.* The release is a new off-ledger service. It writes nothing to the ledger and changes no Splice, Canton or token standard package.

---

## Milestones and Deliverables

### Milestone 1: Core release and metric catalogue v1

- **Estimated Delivery:** month 4 after approval
- **Focus:** the five components, the catalogue and its specification, and a public sandbox.
- **Deliverables / Value Metrics:**
  - Push API, metrics store, UI, API keys and versioned API, released under Apache-2.0 in a public repository.
  - Metric catalogue v1 with its methodology, and the OpenAPI specification.
  - API conformance suite.
  - Public sandbox with test data.
- **Acceptance:** token standard maintainers have reviewed the catalogue and specification against the stable V2 API, with the review recorded publicly; the sandbox is live at a public URL and passes the conformance suite.

### Milestone 2: Production readiness and external audit

- **Estimated Delivery:** month 6 after approval
- **Focus:** what issuers' operations and security teams need before production.
- **Deliverables / Value Metrics:**
  - Operator features: SSO, roles, alerting, audit export.
  - Deployment tooling: Helm chart, Terraform, upgrades, backups.
  - Consumer SDKs for TypeScript and Python.
  - Reference backend that posts figures from a database or a spreadsheet.
  - Hardening, an external code audit with retest, and a security documentation pack for issuers' compliance reviews.
- **Acceptance:** the audit report is published, with every critical and high finding fixed and retested; the release is tagged with its Helm chart, and the SDKs are published.

### Milestone 3: DA Utility in production

- **Estimated Delivery:** month 8 after approval
- **Focus:** the first production deployment.
- **Deliverables / Value Metrics:**
  - DA Utility's deployment live in production, with its first clients publishing.
  - Documentation, operator runbooks and two workshops for issuer operators and consumers.
- **Acceptance:** DA Utility's production deployment serves catalogue metrics for at least one client instrument through the versioned API, confirmed in writing by DA; documentation and runbooks are published and both workshops are held.

### Milestone 4: Maintenance and consumer onboarding

- **Estimated Delivery:** quarterly, months 8 to 20
- **Focus:** keep the release current and bring consumers onto it.
- **Deliverables / Value Metrics:**
  - Patches and release support on the two latest minor versions, including updates for token standard changes.
  - Onboarding of explorers and dashboards as consumers.
  - A public quarterly report: releases, fixes, known deployments and consumers.
- **Acceptance:** each quarter, maintained releases are published and the quarterly report is posted.

### Adoption payments

- **Estimated Delivery:** as they occur, months 8 to 20
- **Issuer:** an issuer publishes at least one catalogue metric for at least one instrument on a production deployment, DA Utility's, its own or a hosted one, for 30 consecutive days, confirmed in writing by the issuer or its operator. Each issuer counts once; DA Utility's clients count individually.
- **Consumer:** a named third party (explorer, dashboard, index or data provider) reads a production deployment through the versioned API and shows or uses the figures in its production product, confirmed in writing. Kaiko and its affiliates do not count.

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- Deliverables completed as specified for each milestone
- Demonstrated functionality or operational readiness
- Documentation and knowledge transfer provided
- Alignment with stated value metrics

Project-specific conditions:

- Each milestone's **Acceptance** line is its payment gate, checked from published evidence or written confirmation.
- A deployment counted for Milestone 3 or for an adoption payment passes the read-only conformance checks, and a reviewer designated by the Committee can read its figures through a public metric or a reviewer key.
- A counted deployment serves catalogue metrics only: no party identifiers, positions or transactions.

---

## Sustainability

- Issuers operate their own deployments; managed hosting serves those that prefer a third party.
- Kaiko maintains the open-source release for 12 months after Milestone 3, under Milestone 4.
- Afterwards, Kaiko keeps maintaining the release it runs, unmodified, for its managed customers. The specification and catalogue stay in the public repository, versioned so that published definitions keep their meaning.

---

## Funding

**Total Funding Request:** up to 8,791,667 CC ($1,055,000 at the $0.12 baseline): 7,491,667 CC ($899,000) for Milestones 1 to 4, and up to 1,300,000 CC ($156,000) in adoption payments.

### Payment Breakdown by Milestone

Each payment is due on Committee acceptance.

| Milestone | Due | USD baseline | CC at $0.12 |
| :- | :- | -: | -: |
| Milestone 1: Core release and metric catalogue v1 | Month 4 | $462,000 | 3,850,000 |
| Milestone 2: Production readiness and external audit | Month 6 | $311,000 | 2,591,667 |
| Milestone 3: DA Utility in production | Month 8 | $52,500 | 437,500 |
| Milestone 4: Maintenance and consumer onboarding | Quarterly, months 8 to 20 | $73,500, or $18,375 per quarter | 612,500, or 153,125 per quarter |
| Adoption: issuer publishing in production | As they occur, up to 10 | $12,000 each | 100,000 each, up to 1,000,000 |
| Adoption: consumer integration | As they occur, up to 4 | $9,000 each | 75,000 each, up to 300,000 |
| **Total** | | **up to $1,055,000** | **up to 8,791,667** |

### Cost Basis

| Work package | FTE-months | Cost | Milestone |
| :- | -: | -: | :- |
| Push API: validation, idempotent writes, backfill | 3.5 | $73,500 | 1 |
| Metrics store: history, restatements, retention, export | 4 | $84,000 | 1 |
| UI: instruments, consent per metric and audience, audit log | 4.5 | $94,500 | 1 |
| API keys: scoping, rotation, usage logs, rate limits | 2.5 | $52,500 | 1 |
| Versioned API: OpenAPI specification, publication states | 3.5 | $73,500 | 1 |
| Metric catalogue, methodology and API conformance suite | 3 | $63,000 | 1 |
| Public sandbox with test data | 1 | $21,000 | 1 |
| Operator features: SSO, roles, alerting, audit export | 4 | $84,000 | 2 |
| Deployment tooling: Helm, Terraform, upgrades, backups | 1.5 | $31,500 | 2 |
| Consumer SDKs, TypeScript and Python | 1 | $21,000 | 2 |
| Reference backend: posting from a database or a spreadsheet | 1.5 | $31,500 | 2 |
| Hardening and audit remediation | 2.5 | $52,500 | 2 |
| External code audit with retest | | $80,000 | 2 |
| Security documentation pack for issuers' compliance reviews | 0.5 | $10,500 | 2 |
| Documentation, operator runbooks, two workshops | 1 | $21,000 | 3 |
| Project management and specification governance | 1.5 | $31,500 | 3 |
| Consumer onboarding: explorers and dashboards | 1 | $21,000 | 4 |
| Open-source release maintenance, 12 months | 2.5 | $52,500 | 4 |
| **Total** | **39** | **$899,000** | |

39 FTE-months at $21,000 come to $819,000; the external code audit with retest adds $80,000, for $899,000. $21,000 per FTE-month covers loaded engineering cost with overhead for Kaiko France staff, in line with the approved CAPS grant ([#497](https://github.com/canton-foundation/canton-dev-fund/pull/497)): 9 FTE for 6 months at 9,850,000 CC, about $21,200 per FTE-month at its $0.116 baseline. Maintenance is 2.5 FTE-months spread over twelve months, for patches and release support on the two latest minor versions; it covers the code, and operating a deployment is priced separately as the managed service.

### Volatility Stipulation

CC amounts are computed at a baseline of $0.12 per CC. Each disbursement is adjusted using the 7-day time-weighted average CC/USD price (TWAP) immediately preceding its acceptance date, so that each payment keeps its baseline USD value, as in the approved CAPS grant (#497). The Committee designates the price source; Kaiko's own CC/USD feed is used only if the Committee chooses it. If the Committee prefers fixed CC, the baseline amounts apply, with the re-evaluation at the 6-month mark that the proposal template sets for projects longer than six months.

---

## Co-Marketing

Upon release, Kaiko will collaborate with the Foundation on:

- Announcement coordination for the open-source release (Milestone 1) and for the first production deployment (Milestone 3), with DA Utility's agreement
- A technical blog on publishing issuer metrics from a private network, and a case study of the DA Utility deployment
- Developer and ecosystem promotion through the public sandbox, the SDKs and the two workshops

---

## Motivation

Tokenised assets are compared on supply, holders and activity. On public chains these figures come from the ledger; on Canton, only the issuer can supply them. Without a standard way to do so, Canton assets are absent from the explorers, dashboards and indices where markets form views, or reach them through one-off integrations that each issuer and each platform must build and review.

With one release and one catalogue, an issuer approves each metric and audience once and keeps the record its compliance team asks for, and a data platform integrates once and reads every deployment the same way. The first deployment covers DA Utility's issuers; the release is open to every token standard registry, V1 (CIP-0056) and V2 (CIP-0112), including decentralised issuers.

---

## Rationale

- **Push only.** The issuer's system already holds the records and computes the figures. Pushing keeps the release free of ledger access and of each issuer's data model, so it runs the same at every issuer, and the issuer stays the golden source.
- **On premise first, managed as an option.** On premise, the issuer alone holds its figures, consent records and keys, and Kaiko is not a party to its data. Managed hosting serves issuers without an operations team, with the same release and API.
- **One catalogue, one API.** Consumers integrate once, and the conformance suite keeps deployments consistent across versions.
- **Aggregation stays with consumers.** Each issuer serves its own figures; combining issuers is a consumer's job.
- **Alternatives considered.** A Kaiko-hosted aggregator holding every issuer's data would make Kaiko a party to each issuer's data agreements and take publication control away from issuers. An attestation service with on-ledger verification adds weight that the first approach of RFP 11 does not require; CADF anchoring and the RFP's TEE and proof approaches cover that need. A ledger connector reading holdings directly would need access to the issuer's participant and its data model, and would differ per issuer.
- **Extending what exists.** The release reuses the token standard's instrument IDs and the Token Metadata API's definitions and can feed CADF. It adds the issuer-side layer: history, flows, consent and keys.

---

## Team

Kaiko provides digital-asset market data and benchmarks; it is an EU BMR benchmark administrator and holds a SOC 2 Type 2 report. On Canton, Kaiko operates a Super Validator, publishes the CC/USD feed that Super Validators can follow for their Amulet conversion rate votes, and authored the Kaiko Data Standard ([#113](https://github.com/canton-foundation/canton-dev-fund/pull/113), approved May 2026).

Kaiko France employees deliver the project: an engineering manager, backend, frontend, DevOps and integration engineers, and a product manager. Charles Desmonty, product manager and solution architect for Kaiko's Canton products, leads it.

---

## Disclosure

- The Canton Foundation took part in Kaiko's Series B extension in September 2026.
- Kaiko operates a Super Validator and publishes a CC/USD feed; the Volatility Stipulation leaves the choice of price source to the Committee.
- The author is a member of the Token Standards / Asset Standards and Financial Workflows & Composability SIGs and is also this proposal's champion.
- Outside this grant, Kaiko offers managed hosting of the release and may read deployments for its own data products, with issuers' keys, like any consumer.
