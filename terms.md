# kotobase.net — Terms of Service

> **DRAFT — This is not legal advice. Review by qualified counsel is required before publication.**

Last updated: 2026-07-26

Governing law: Japan

Service operator: Gftd Japan 株式会社 (Gftd Japan K.K.), GranTokyo South Tower 11F, 1-9-2 Marunouchi, Chiyoda-ku, Tokyo 100-6611, Japan (Corporate Number 1011101086505) (the "Operator", "we", "us").
Service: **kotobase.net**, a content-addressed Knowledge Graph Backend-as-a-Service (the "Service").

---

## 1. Acceptance

By creating an account, authenticating, calling the API, or otherwise using the
Service, you agree to these Terms of Service (these "Terms") and to the
[Privacy Policy](./privacy.md). If you use the Service on behalf of an
organization, you represent that you are authorized to bind that organization,
and "you" includes that organization. If you do not agree, do not use the
Service.

## 2. Service description

The Service is a managed, tenant-scoped knowledge-graph platform built on the
`kotoba` content-addressed distributed Datalog engine (CID / Prolly Tree / AT
Protocol). It provides, subject to your tier:

- creation of tenant-scoped knowledge graphs;
- knowledge-graph ingest via API and MCP;
- read query surfaces including Datomic/Datalog (`q`, `pull`, `datoms`, `asOf`,
  `history`) and SPARQL/Cypher-style graph queries over the Datom head;
- Datom write (`transact`, `with`) bound to your own tenant namespace;
- durable pinning of graph commits as content-addressed objects, including
  optional off-site archival as IPLD CARs on object storage; and
- standard IPFS pinning and retrieval interfaces (`/ipfs/*`, `/ipns/*`).

The Service is offered under one of three plans — **P2P**, **Secure Managed**, or
**Enterprise** — which differ principally in **who bears operational
responsibility**. Section 6 sets out the plans, Section 7 the service levels and
warranties that follow from them, and Section 8 the limitation of liability.
`docs/PLANS.md` records the per-plan commitments in full and forms part of these
Terms by reference. `docs/SERVICE-LEVELS.md` records the Operator's internal
operational objectives for the shared endpoint and is not itself a customer SLA.
Features, routes, quotas, and interfaces may change.

## 3. Accounts and DID authentication

- **Identity model.** The Service is tenant-scoped and keyed on a tenant
  decentralized identifier ("tenant DID"). You authenticate either (a) with a
  bearer token (JWT) issued by the Operator's authentication service, which the
  Service resolves to your tenant DID, or (b) self-sovereignly with a signed
  capability object ("CACAO"). Write operations to the Datom plane require a
  CACAO whose issuer equals your tenant DID.
- **Namespace binding.** The Service binds each write to your own tenant
  namespace (`kotobase/db/<tenant_did>/<db_name>`). A client-supplied target
  cannot override this binding; you can only write to graphs you own, except for
  explicitly allowlisted operator identities.
- **Credential security.** You are responsible for safeguarding your private
  keys, CACAO signing material, and bearer tokens, and for all activity under
  your tenant DID. You must notify us promptly of any suspected compromise.
- **Eligibility.** You must be capable of forming a binding contract and must not
  be barred from receiving the Service under applicable law. [CONFIRM: minimum
  age requirement, e.g. 18 / age of majority under Japanese Civil Code.]

## 4. Acceptable use

You must not:

- upload, store, or transmit content that is unlawful, infringing, or that you
  lack the rights to process;
- attempt to bypass authentication, tenant isolation, quotas, or the tenant-lock
  mechanism, or access another tenant's data;
- interfere with, overload, or disrupt the Service, its edge, or the backend
  graph pods;
- submit secrets, third-party credentials, or sensitive personal data into
  fields, logs, or public metadata contrary to the documented data-handling
  boundary (`docs/DATA-HANDLING.md`); or
- use the Service to store content that must be rendered permanently
  irretrievable, given the content-addressed nature described in Section 6 and
  Section 7.

We may suspend or throttle usage that threatens the integrity, security, or
availability of the Service.

## 5. Customer data and intellectual property ownership

- **Your content.** As between you and us, you retain all rights, title, and
  interest in the graph data, facts, relationships, pins, and payloads you
  ingest or store ("Customer Data"). You grant us a limited license to host,
  process, transmit, replicate, content-address, pin, and archive Customer Data
  solely to provide and operate the Service.
- **Content-addressing notice.** Customer Data is stored as content-addressed
  objects (CIDs). Identical content produces identical CIDs, and content may be
  replicated to IPFS peers, gateways, caches, or off-site CAR archives. See
  Section 7 for the consequences for deletion.
- **Our IP.** We and our licensors retain all rights in the Service, the Worker
  control plane, the `kotoba` engine, documentation, and related software. The
  `kotoba` submodule remains under its own upstream Apache-2.0 license; nothing
  here grants you rights in third-party or upstream components beyond their own
  licenses.
- **Feedback.** You grant us a perpetual, royalty-free license to use feedback
  you provide to improve the Service.

## 6. Plans, fees, and billing (Stripe)

### 6.1 The three plans

| Plan | Who bears operational responsibility | Availability commitment | Support | How it is entered |
|---|---|---|---|---|
| **P2P** | **You.** The Operator bears none. | None | None | Self-serve, no fee |
| **Secure Managed** | **The Operator** (Gftd Japan K.K.), on these Terms | 99.9% per calendar month, with service credits | Email, business hours JST, 1 business day acknowledgement | Self-serve subscription |
| **Enterprise** | **The Operator**, on the terms of a signed enterprise agreement, which prevail over these Terms where they conflict | 99.999% per calendar month, with service credits, **on the dedicated deployment provisioned under that agreement** | Named contact, 24x7, 15-minute P1 acknowledgement | Signed enterprise agreement |

- **P2P — no Operator responsibility.** On the P2P plan the Service is provided
  strictly "as is" and "as available". The Operator gives **no availability
  commitment, no service credits, no incident-response obligation, no recovery
  objective, and no support obligation**, and may perform maintenance or suspend
  the shared endpoint without notice. You accept operational risk in full. This
  allocation is a basis on which the P2P plan is offered at no fee.
- **Secure Managed — Operator responsibility.** On the Secure Managed plan the
  Operator operates, monitors, patches, and recovers the Service and is
  accountable for it on the terms of Section 7.2, including the availability
  commitment and service credits.
- **Enterprise — responsibility by agreement.** The Enterprise plan is entered
  by signed agreement. That agreement governs the availability commitment,
  service credits, support response, dedicated infrastructure, data residency,
  key custody, audit rights, retention and deletion, and any data processing
  addendum or business associate agreement. **The Enterprise availability
  commitment and multi-region recovery objectives attach only to the dedicated
  deployment provisioned under the agreement, and not to the shared endpoint.**
  There is no self-serve purchase path for the Enterprise plan.
- **Compliance status (no misrepresentation).** The Operator's SOC 2 Type II,
  ISO/IEC 27001, and ISMAP programs are **in preparation**; as of the "Last
  updated" date no such report or certificate has been issued. The Operator does
  not represent that it holds any of them, and will make a report available under
  a non-disclosure agreement once issued. `docs/PLANS.md` records the current
  status of each framework.
- **Prices.** [CONFIRM: current published prices and plan limits — do not treat
  repository figures as final pricing.] Legacy tier identifiers (`free`,
  `starter`, `standard`, `pro`, `regulated`) continue to resolve to the plan that
  superseded them for existing subscriptions and API clients.
- **Payment processor.** Paid subscriptions are processed through **Stripe**.
  Checkout sessions are created via the Service; your card and payment details
  are collected and processed by Stripe under Stripe's terms and privacy policy.
  We do not store full payment card numbers.
- **Billing cycle.** Secure Managed is a recurring subscription billed [CONFIRM:
  monthly / annual]. Subscription entitlements are provisioned and renewed based
  on signature-verified Stripe webhook events keyed to your tenant DID.
- **Taxes.** Fees are exclusive of taxes unless stated otherwise; you are
  responsible for applicable taxes, including Japanese consumption tax where
  relevant. [CONFIRM tax handling.]
- **Cancellation and downgrade.** On cancellation, paid entitlements downgrade to
  the P2P plan either at period end or immediately depending on the cancellation
  state. **On downgrade to P2P the Operator's availability, support, and recovery
  obligations end**, and the P2P allocation of responsibility in Section 6.1
  applies from that point. [CONFIRM: refund policy.]

## 7. Service levels, warranties, and disclaimers

### 7.1 P2P plan — no warranty, no service level

On the P2P plan the Service is provided on an **"as is" and "as available"** basis
without warranties of any kind, whether express or implied, including
merchantability, fitness for a particular purpose, and non-infringement, to the
maximum extent permitted by law. We do not warrant uninterrupted or error-free
operation, and there is **no committed uptime, RTO, RPO, multi-region
availability, support response, or uptime credit** on this plan.

### 7.2 Secure Managed plan — availability commitment and service credits

- **Availability commitment.** We will make the Service available to your tenant
  at least **99.9% of each calendar month**, excluding scheduled maintenance
  announced in advance, force majeure, failures of your own systems or network,
  suspension permitted under Section 4 or 9, and unavailability caused by your
  content or configuration.
- **Service credits.** If we miss that commitment in a calendar month, on your
  written request within 30 days of the end of that month we will credit:

  | Monthly availability | Credit |
  |---|---|
  | below 99.9% | 10% of that month's fee |
  | below 99.5% | 25% of that month's fee |
  | below 99.0% | 50% of that month's fee |

- **Exclusive remedy.** Service credits are your sole and exclusive remedy for a
  missed availability commitment.
- **Support.** We will acknowledge incidents you report by email within one
  business day (JST business hours).
- **Recovery.** We maintain a recovery objective of no loss of committed
  transactions and restoration of logical service within 10 minutes, verified by
  the recovery drills recorded in `docs/RECOVERY.md`. Storage-level loss of the
  underlying object store is outside the scope of the current drill.
- Except as set out in this Section 7.2, the disclaimers in Section 7.1 apply to
  the Secure Managed plan as well.

### 7.3 Enterprise plan

The Enterprise availability commitment (99.999% per calendar month), service
credits of up to 100% of the monthly fee, 24x7 support with a 15-minute P1
acknowledgement, multi-region recovery objectives, data residency, key custody,
audit rights, and compliance deliverables are set out in the signed enterprise
agreement and **apply to the dedicated deployment provisioned under it**. They do
not apply to the shared endpoint, and no statement in this repository or on the
Service's public pages creates them absent that signed agreement.

### 7.4 Applies to every plan

- **No guaranteed erasure.** Because Customer Data is content-addressed and may
  be replicated across IPFS peers, gateways, caches, and off-site archives, we
  do not warrant cryptographic erasure or that deleted content becomes
  permanently unretrievable. Deletion marks metadata as deleted; see Section 8
  and the Privacy Policy.
- The Service is **not** offered as a regulated (financial, medical,
  governmental) compliance product on the P2P or Secure Managed plans. Regulated
  use requires an Enterprise agreement with explicit controls.
- We do not represent that we hold a SOC 2, ISO/IEC 27001, or ISMAP report or
  certificate; those programs are in preparation as stated in Section 6.1.

## 8. Limitation of liability

To the maximum extent permitted by applicable law:

- neither party is liable for indirect, incidental, special, consequential, or
  punitive damages, or for lost profits, revenue, data, or goodwill;
- **on the P2P plan**, which is provided at no fee and on the express allocation
  of operational responsibility in Section 6.1, the Operator's aggregate
  liability arising out of or relating to the Service is limited to [CONFIRM:
  e.g. JPY 10,000 / the minimum permitted by applicable law];
- **on the Secure Managed plan**, the Operator's aggregate liability arising out
  of or relating to the Service will not exceed [CONFIRM: e.g. the fees you paid
  to us in the 12 months preceding the claim]; and
- **on the Enterprise plan**, the limitation in the signed enterprise agreement
  applies and prevails over this Section.

Nothing in these Terms limits liability that cannot be limited under applicable
mandatory law (including Japanese Consumer Contract Act protections where they
apply).

## 9. Term and termination

- These Terms apply while you use the Service.
- You may stop using the Service and close your account at any time.
- We may suspend or terminate access for material breach, non-payment, legal
  requirement, or risk to the Service, with notice where practicable.
- On termination we may delete tenant metadata subject to the deletion and
  retention limits described in Section 7 and the Privacy Policy. Content-addressed
  bytes already replicated may persist.
- Sections that by their nature should survive (ownership, disclaimers,
  limitation of liability, governing law) survive termination.

## 10. Governing law and dispute resolution

- These Terms are governed by the laws of **Japan**, without regard to conflict
  of laws rules.
- The parties submit to the Tokyo District Court (東京地方裁判所) as the court of
  exclusive jurisdiction of first instance, unless mandatory
  consumer-protection law provides otherwise.
- Nothing in this Section deprives a consumer of protections that cannot be
  derogated from under the mandatory law of their place of residence.

## 11. Changes

We may update these Terms. Material changes will be indicated by updating the
"Last updated" date and, where appropriate, by additional notice. [CONFIRM:
notice mechanism and advance-notice period.] Continued use after changes take
effect constitutes acceptance.

## 12. Contact

Email: hello@gftd.co.jp. The Operator's registered address is GranTokyo South Tower 11F, 1-9-2 Marunouchi, Chiyoda-ku, Tokyo 100-6611, Japan (Corporate Number 1011101086505).
