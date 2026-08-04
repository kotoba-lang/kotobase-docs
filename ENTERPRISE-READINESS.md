# Enterprise readiness gates

`enterprise` is a deployment qualification, not a product label. A dedicated
customer deployment may be called enterprise-ready only when every P0 gate
below has current evidence. A design document or a passing unit test is not
operational evidence.

## P0 admission gates

| Gate | Required evidence | Shared production state, 2026-08-03 |
|---|---|---|
| Tenant isolation | Cross-tenant negative tests for every read/write/query surface | Implemented release-wide evidence registry: `docs/enterprise/tenant-isolation-matrix.json`, enforced by `enterprise-isolation.yml`; private and public-by-design surfaces are explicit |
| Controlled change | Protected default branch, two-person review, required checks, protected production environment, CI-only deploy | Partial: the fail-closed weekly audit and 30-day evidence contract are implemented, and `production` accepts only `main`. The branch-protection, deployment-review and CI-credential requirements are not all satisfied on the current repository plan, so the two-person-review and CI-only-deploy gates are **not** met. Control-level detail is held in the operator's internal `CONTROLLED-CHANGE` record |
| Vulnerability management | High/Critical dependency findings block release; remediation owner and deadline | Implemented repository-wide: all seven npm lockfiles run a daily and change-triggered High/Critical release gate; Dependabot covers every workspace |
| Query contract | Machine-readable capability/version declaration and cross-protocol conformance | Partial: SPARQL/Cypher/Gremlin/GraphQL discovery is implemented here; GraphDB/RDF4J conformance remains an independent surface |
| Client ecosystem | Authenticated protocol conformance in every supported language and conflict-safe local-first integration | Implemented: TypeScript, Python, Rust and PHP each open a hermetic WebSocket, send the tenant bearer and GraphSON bytecode envelope, correlate the response and reject GraphSON errors in the required SDK matrix. The Obsidian plugin has CI-gated push, pull and two-way sync primitives; unresolved concurrent edits are preserved and block automatic overwrite until an explicit local edit |
| Availability | Continuous customer-journey SLI, paging, error budget and status communication | Partial: customer-journey and protocol/GraphDB probes run every five minutes and daily SLO/resilience gates remain. A protected live drill now fail-closes on a current two-person 24x7 rota, real P1 delivery, in-rota acknowledgement within 900 seconds, and investigating/resolved publication outside Kotobase/Cloudflare. No live receipt, staffed rota or independent status deployment is present yet |
| Recovery | Storage-level and multi-region restore meets contracted RPO/RTO | Partial: logical restore is drilled, and a fail-closed protected workflow now exercises isolated B2 primary-object loss, cross-region checksum verification, failover, restoration, timing limits and exact cleanup. No successful two-region live artifact has been produced yet, so contracted recovery remains unqualified |
| Security assurance | OWASP ASVS 5.0 control evidence and independent penetration test with no open Critical/High | Not met: the full multi-protocol scope, rules of engagement, cryptographic report intake, assessor-key pinning, retest/freshness contract and protected receipt workflow are ready. No independent assessor, signed final report or retest exists yet; see `docs/PENETRATION-TEST.md` |
| Data governance | DPA, subprocessors, retention schedule, DSAR, legal hold, residency and immutable-data deletion language | Partial: versioned DPA review template, machine-readable processor/retention registers, DSAR/deletion runbook and explicit residency/immutable-erasure boundary are CI-gated. Enterprise evidence now requires current counsel approval, exact document digests and passing DSAR/export/cleanup plus legal-hold apply/release drills within 180 days. Counsel approval and live drills are not present, and production legal hold is explicitly unqualified |
| Supply chain | Immutable CI artifact, signed provenance/SBOM, deployment of the verified digest | Partial: `protocols-worker`, `kotobase-api-gateway`, Authn and `kotobase-graph-database` have been deployed from SHA-256-checked, keyless-Sigstore-verified CI artifacts. The deployed backend was verified live: `/_health` returned 200 and a direct data-plane request remained correctly edge-restricted with 401. The CI-only production deployment path is **not** yet in use, so provenance verification and deployment are not performed by the same automated actor. Exact release identifiers are held in the operator's internal evidence bundle |
| Customer operations | SSO/SCIM, RBAC/service accounts, audit export, key rotation, 24x7 escalation | Partial: enterprise OIDC is CI-gated; strongly-consistent tenant membership, role/permission enforcement, digest-only service credentials with atomic rotation/revocation, SCIM 2.0 User lifecycle, tenant identity-change audit listing, and bounded atomic identity-plane cleanup have hermetic Worker tests. Every enterprise mutation now checks tenant liveness in its write transaction, so cleanup cannot race an orphan back into existence. A fail-before-create live runner covers SCIM, RBAC denial, rotation, exact-byte data-plane audit export and verified cleanup, but it has not yet run against a deployment containing the cleanup route. Authn health and all three SCIM discovery endpoints return HTTP 200 in production. The guarded live lifecycle, fail-before-read audit-plane evidence and staffed 24x7 rota remain open |

## Protocol registry

| Surface | Deployment owner | Public contract | Qualification rule |
|---|---|---|---|
| Datomic Client API | main kotobase Worker / D1 service binding | `/api/*` | semantic and tenant isolation suite |
| SPARQL | `protocols-worker` | documented read-only subset | capability discovery + live smoke + conformance fixtures |
| Cypher | `protocols-worker` | documented read-only subset; no Bolt | capability discovery + live smoke + conformance fixtures |
| Gremlin | `protocols-worker` | WebSocket GraphSON bytecode subset | capability discovery + live smoke + conformance fixtures |
| GraphDB/RDF4J | independent query deployment | read-only `default` repository | health contract + RDF4J conformance suite; deployment ownership must be recorded before promotion |
| GraphQL | `protocols-worker` | read-only `kotobase-document-v1` over `POST /graphql` | private-by-default auth + schema validation + audit + capability discovery + conformance tests |

The five-minute `Protocol surfaces public smoke` workflow verifies that SPARQL,
Cypher, Gremlin, GraphQL and GraphDB are reachable, preserve request correlation, and do
not silently broaden the declared query contract.

## Promotion rule

The shared endpoint remains P2P/Secure Managed while any P0 row is `Partial` or
`Not met`. Enterprise contracts provision a dedicated deployment and attach its
evidence bundle to a release ID using
`protocols-worker/scripts/verify-operational-evidence.mjs`.

Framework mapping should use NIST CSF 2.0 for governance, NIST SSDF 1.1 for the
development lifecycle, OWASP ASVS 5.0.0 for application verification and SLSA
1.2 for build provenance. Mapping does not constitute certification.

The complete machine-enforced operational bundle contract is documented in
`docs/enterprise/EVIDENCE-BUNDLE.md`. Missing evidence fails qualification; the
verifier never converts a template or a unit test into operational proof.
