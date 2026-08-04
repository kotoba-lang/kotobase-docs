# kotobase.net Plans

kotobase.net is sold as three plans, and the line between them is **who carries
operational responsibility**:

| Plan | Accountable party | Availability | Compliance | How to start |
|---|---|---|---|---|
| **P2P** | You do. The operator carries none. | No commitment | None | Self-serve, free |
| **Secure Managed** | Gftd Japan K.K. | 99.9% monthly SLA with service credits | Documented data-handling, threat model and SBOM; DPA template pending counsel | Self-serve subscription |
| **Enterprise** | Gftd Japan K.K., under contract | 99.999% monthly SLA with a credit schedule up to 100% | SOC 2 / ISO 27001 / ISMAP-class response; DPA/BAA pending counsel; proposed residency, key custody and read audit | Contract — hello@gftd.co.jp |

The machine-readable source of truth is `kotobase-api-gateway-cljs/src/kotobase/plans.cljc`
(`kotobase.plans`). The landing page, `/signup`, `/admin`, the
`POST /billing/checkout` admission rule, the `accountCreate` tier enum, and the
Stripe webhook fulfillment all read it, so this document and the pages cannot
drift from what the service actually enforces. `legal/terms.md` §6/§7/§8 is the
contractual projection. Decision record: ADR-2607261700.

## Reading the status vocabulary

A plan states what the operator owes, and a service can owe something it cannot
yet prove. Every commitment in the catalog therefore carries a status, and the
pages render it verbatim rather than flattening it into a checkmark:

| Status | Meaning |
|---|---|
| `:in-force` | In force today for tenants on the plan, measured by the daily `slo-gate` receipts (`kotobase-graph-database/slo/`, ADR-2607250100). |
| `:contractual` | In force only under a signed contract, on the dedicated deployment that contract provisions — **not** on the shared production endpoint. |
| `:none` | Deliberately not committed (P2P). |
| `:in-preparation` | A program that has started but has produced no report or certificate. Must never be presented as "certified". |
| `:available` | Deliverable on request today (contract paperwork, not an audited control). |
| `:review-required` | A template or proposed commitment exists, but qualified counsel has not approved it for customer execution. |

## P2P — the operator carries no responsibility

The plan for developers, self-hosters, researchers, and anyone federating their
own kotoba peer.

- **Responsibility:** yours. The service is provided "as is" and "as available"
  with **no availability commitment, no service credits, no incident-response
  obligation, no recovery objective, and no support obligation.** Maintenance and
  outages can happen without notice.
- **Included:** 512 MiB and 50 pins per tenant on the shared endpoint; unmetered on
  your own peer. The shared-endpoint numbers here are the same constants the edge
  enforces (`kotobase.b2/free-pin-quota`, `free-byte-quota` derive from the
  catalog), so a 429 can never contradict this table.
- **Capabilities:** tenant-scoped knowledge graphs, Datomic/Datalog reads and
  Datom writes, SPARQL/Cypher-style query, IPFS pinning and `/ipfs/*` `/ipns/*`
  reads, MCP tools, content-addressed commit provenance, and federation of your
  own peer over the native IPLD plane.
- **Auth:** gftd-AUTHN JWT (Bearer) or self-signed CACAO.
- **Isolation:** shared multi-tenant edge with tenant-DID namespace binding only.
- **Price:** free. `p2p` is the plan every unprovisioned tenant is on.

If your application cannot tolerate an unannounced outage, this is the wrong
plan. That is not a caveat buried in a footnote — it is the plan's defining
property, and choosing it is choosing to own the risk.

## Secure Managed — Gftd Japan K.K. is responsible

The plan for teams and production applications that need someone accountable for
uptime and data handling.

- **Responsibility:** Gftd Japan K.K. operates, monitors, patches, and recovers
  the service, and is accountable for it on the published terms.
- **Availability SLA:** **99.9% monthly** — a 43m 49s error budget per 30-day
  month. Measured by the daily `slo-gate` (probe + recovery drill), whose
  receipts are committed under `kotobase-graph-database/slo/receipts/`.
- **Service credits** (exclusive remedy, requested within 30 days of the affected
  month):

  | Monthly availability | Credit |
  |---|---|
  | below 99.9% | 10% of the monthly fee |
  | below 99.5% | 25% of the monthly fee |
  | below 99.0% | 50% of the monthly fee |

- **Support:** email, business hours JST, **one business day** acknowledgement
  target for customer-reported incidents.
- **Recovery:** RPO 0 committed transactions (the commit log is
  content-addressed); RTO under 10 minutes for a logical restore, drilled by
  `slo/recovery_drill.cljs`. Storage-level (B2 bucket loss) restore is out of the
  current drill's scope — see the honest scope note in
  `docs/SERVICE-LEVELS.md`.
- **Included:** 500 GiB and 10,000 pins, with usage expansion beyond that.
- **Isolation:** managed multi-tenant with enforced tenant-DID isolation, the
  tenant-lock write quota, and encryption in transit and at rest.
- **Compliance posture:** documented data-handling boundary
  (`docs/DATA-HANDLING.md`), published threat model (`docs/THREAT-MODEL.md`),
  supply-chain review and SBOM (`docs/SUPPLY-CHAIN.md`). A DPA review template
  exists, but is not available for execution until qualified counsel approves
  it and the privacy evidence gate passes.
- **Billing:** self-serve Stripe subscription.
  `POST /billing/checkout {"tier":"secure-managed"}`. The recurring Price ID
  comes from `STRIPE_PRICE_SECURE_MANAGED`.

## Enterprise — responsibility under contract

The plan for financial, medical, government, and enterprise data teams under
audit or regulatory obligation. Everything in Secure Managed, plus:

- **Availability SLA: 99.999% monthly** — a **26-second** error budget per
  30-day month, with a credit schedule:

  | Monthly availability | Credit |
  |---|---|
  | below 99.999% | 10% of the monthly fee |
  | below 99.99% | 25% of the monthly fee |
  | below 99.9% | 50% of the monthly fee |
  | below 99.0% | 100% of the monthly fee, plus termination for cause without penalty |

- **Support:** named technical account manager and a dedicated escalation
  channel, 24x7, **15-minute P1** and 2-hour P2 acknowledgement.
- **Dedicated tenant and dedicated infrastructure**, CACAO-only auth,
  encrypt-by-default, customer-held key custody, permissioned swarm with public
  DHT advertisement off.
- **Recovery:** RPO 0, RTO under 5 minutes, multi-region — verified by a recovery
  drill on the customer's own deployment.
- **Compliance response:**

  | Framework | Status |
  |---|---|
  | SOC 2 Type II | **In preparation** — no report issued yet; shared under NDA once the audit completes. Gate G1 in `docs/adr/regulated-tier-maturity.md` (currently L0). |
  | ISO/IEC 27001 | In preparation — same program; no certificate issued yet. |
  | ISMAP | In preparation — organizational, contract-and-year-scale work (ADR-2606060003). |
  | DPA | **Legal review required** — an unapproved template exists; it is not offered for execution today. |
  | BAA | **Legal review required** — no approved BAA is offered today. |
  | Security questionnaire / audit response | Available today, answered from the published threat model, data-handling boundary, supply-chain doc, and release-evidence gate. |
  | Data residency (in-jurisdiction cold pins) | Contractual — region-pinned B2 archival plus region-tagged peer filtering (L1 in the maturity ADR). |
  | Read audit (universal read receipts) | Contractual — ADR-2606131600 P0/P1 implemented; enabled per dedicated tenant. |
  | Customer-held key custody (HYOK/KMS) | Contractual — ADR-2606060003 key architecture, provisioned per contract. |

- **Billing:** sales-led contract. There is deliberately **no checkout control
  for Enterprise anywhere in the product** — not on the pricing section, not on
  `/signup`, and `POST /billing/checkout {"tier":"enterprise"}` is refused with
  400. A plan whose terms are negotiated cannot be bought with a button.

### The 99.999% readiness gates

A 26-second monthly error budget is an architecture requirement, not a marketing
number. The operator signs it **per contract, on the dedicated deployment that
contract provisions**, and only once that deployment clears these gates. They are
carried in the catalog (`:plan/readiness`) so the honest current state is visible
in code, not just in a slide:

| Gate | State | Tracked by |
|---|---|---|
| Multi-region write plane with quorum commit | not met | ADR-2607142200 |
| Independent witness mesh with direct peering (no operator pod in the reconciliation path) | not met | `docs/adr/regulated-tier-maturity.md` (E-track) |
| 24x7 on-call rota with paging and an error-budget policy | not met | `docs/RUNBOOK.md` |
| Continuous availability measurement at the contracted granularity | partial — a daily probe exists; per-minute continuous measurement does not | `kotobase-graph-database/slo/` (ADR-2607250100) |
| Multi-region recovery drill receipts meeting the contracted RPO/RTO | partial — logical restore drilled (RTO 4.8s against a 600s limit); storage-level and multi-region restore not drilled | `slo/recovery_drill.cljs` |
| SOC 2 Type II report issued | not met | maturity gate G1 (L0) |

**Consequence, stated plainly:** the shared production endpoint does not carry a
99.999% SLA today, and Enterprise is not a self-serve upgrade of it. Enterprise
is a contract for a deployment built to hold those numbers, and the sales
conversation is where the gate state and the timeline are disclosed. Do not
represent the current alpha endpoint as five-nines infrastructure.

## Tier vocabulary and backward compatibility

The plan ids are `p2p`, `secure-managed`, `enterprise`. The previous four-tier
ladder still resolves, so existing clients, persisted tenant state, and Stripe
Prices whose `metadata[tier]` predates the restructure keep working:

| Legacy tier | Resolves to |
|---|---|
| `free` | `p2p` |
| `starter`, `standard`, `pro` | `secure-managed` |
| `regulated` | `enterprise` |

- `accountCreate` accepts both vocabularies.
- `POST /billing/checkout` accepts `secure-managed` and its legacy aliases; it
  refuses `p2p` (nothing to charge) and `enterprise` (a contract).
- Stripe webhook fulfillment normalizes whatever tier an event carries to a
  current plan id before persisting, and a paid event carrying **no** tier
  defaults to `secure-managed` — never to the free plan, which would strip
  entitlements from a tenant who paid.
- `accountStatus` / `usageGet` return `tier` (a current plan id) plus `plan`,
  `plan_name`, and `responsibility`, so a console can state who is accountable
  without shipping its own copy of the plan table.

## Pricing

Prices live in Stripe, not in repository constants. The Worker resolves the
Secure Managed recurring Price from `STRIPE_PRICE_SECURE_MANAGED`, falling back
to `STRIPE_PRICE_PRO` if the newer variable is unset, so a deployment that has
not been reconfigured keeps billing rather than returning 503.

Live recurring prices in the Stripe account as of 2026-07-26 (resolved by
`lookup_key`, JPY, monthly):

| Stripe lookup_key | Product | Amount | Wired to |
|---|---|---|---|
| `net_kotobase_pro_monthly_jpy` | net-kotobase Team Pro | ¥19,800 | `STRIPE_PRICE_SECURE_MANAGED` **and** `STRIPE_PRICE_PRO` |
| `net_kotobase_standard_monthly_jpy` | net-kotobase Developer Standard | ¥2,980 | `STRIPE_PRICE_STANDARD` (legacy `standard` clients only) |
| `net_kotobase_regulated_monthly_jpy` | net-kotobase Regulated | ¥100,000 | **not wired** — Enterprise is a signed contract, not a self-serve price |

Secure Managed points at Team Pro because that is the entitlement it publishes
(500 GiB / 10,000 pins). A client still posting the legacy `standard` tier keeps
being charged ¥2,980, so nothing re-prices silently.

**Two open pricing decisions for the owner**, deliberately not made in code:

1. Whether ¥2,980 (Developer Standard) should be reinstated as a published
   entry-level option of Secure Managed rather than surviving only as a legacy
   alias. Today the published self-serve price is ¥19,800.
2. Whether the ¥100,000 Regulated price should become a self-serve floor for
   Enterprise. It is currently unused, because Enterprise commitments (99.999%,
   compliance response) require a signed agreement.

Historical note: before 2026-07-26 the two price IDs configured in
`kotobase-api-gateway/wrangler.jsonc` did not exist in the live Stripe account, so every paid
checkout failed with 502. ADR-2607261700 replaced them with the real ones.

Do not treat any figure in this repository as final published pricing —
`legal/terms.md` §6 carries the same caveat.
