# Data Handling

`kotobase.net` stores and routes content-addressed graph pin data. This document
defines the current alpha data-handling boundary for operators and contributors.
It is not a regulated-tier privacy program.

## Data Classes

- Public service metadata: `/health` exact keys (`ok`, `app`, `version`, `ts`),
  `/_app/meta`, DID document, `llms.txt`, version IDs, storage mode, and request
  IDs.
- Tenant identity metadata: tenant DID, CACAO issuer DID, pin owner DID, and
  account status.
- Authn enterprise-control metadata: tenant name, membership roles and reduced
  permission sets, SCIM directory bindings, digest-only service credentials,
  and identity-change audit events.
- Pin metadata: `pin_id`, `name`, `cid`, target type, target, status,
  timestamps, size, content type, archived object keys, and error strings.
- Archive payloads: optional gateway-produced IPFS/IPNS IPLD CARs or raw bytes
  stored under the configured B2 prefix.
- Replay markers: hashed CACAO nonce keys plus issuer, nonce hash, issue time,
  expiry, and seen time. Raw CACAO nonce values are not stored in replay markers.
- Tenant audit receipts: event ID, timestamp, actor DID, optional account DID,
  action, HTTP outcome, request ID, country and transport. Request/response
  bodies, headers, credentials and client IP addresses are not stored.
- Operator diagnostics: `x-kotobase-request-id`, Cloudflare `cf-ray`, smoke
  output, deploy version IDs, and sanitized error summaries.
- B2 recovery-drill data: random synthetic payload bytes plus synthetic pin and
  head metadata below a unique `kotobase/dr-drill/<run-id>` prefix. No tenant
  data is used.

## Storage Locations

Backend mode:

- Tenant pin state, graph commits, Kubo pinning, and CAR-on-B2 export are handled
  by the kotoba backend path.
- The Worker is the public write boundary and forwards verified requests to the
  backend.
- Tenant edge-request receipts are stored as metadata under the structural KV
  prefix `audit:<percent-encoded-tenant-did>:` in `AUDIT_LOG`, or in
  `TENANT_STATE` when the dedicated binding is absent.

Worker-B2 mode:

- Pin metadata is stored under
  `KOTOBASE_B2_PREFIX/pins/{tenant_did}/{pin_id}.json`.
- Replay markers are stored under `KOTOBASE_B2_PREFIX/nonces/*.json`.
- Optional raw objects are stored under `KOTOBASE_B2_PREFIX/objects/{cid}`.
- Optional CAR archives are stored under
  `KOTOBASE_B2_PREFIX/cars/{ipfs|ipns}/{target}.car`.
- Quota-changing writes require `KOTOBASE_TENANT_LOCKS -> KotobaseTenantLock`.

Authn identity plane:

- Accounts, tenant memberships, SCIM bindings, service-account token digests,
  and identity-change events are strongly consistent records in `AUTHN_STORE`.
- Raw service-account tokens are returned once and are never persisted.

## Public Boundaries

- `/_app/meta` exposes only safe runtime facts: storage mode, archive flag, and
  tenant-lock availability. It must not expose B2 endpoint, bucket, key ID, app
  key, gateway URL, internal secret, bearer/CACAO/JWT-like values,
  private-CID-like values, nested secret-like keys, or tenant-specific data.
- Public smoke must not create authenticated writes or require secrets.
- Public issues must not contain bearer tokens, CACAOs, private keys, B2
  credentials, tenant data, private CIDs, sensitive request bodies, or raw logs
  containing sensitive data.
- Use GitHub Security Advisories for exploitable data exposure or auth bypasses.

## Deletion And Retention

- `DELETE https://authn.kotobase.net/v1/tenants/:tenant` deletes only the
  tenant's Authn identity/control-plane records and reverse indexes. It does
  **not** delete graph, pin, archive, billing, or edge-request-audit data. A
  recent human owner must confirm the exact tenant id and name and explicitly
  acknowledge `confirm_data_plane_retained: true`. Up to 100 keys are removed
  atomically; a larger tenant fails unchanged with 409 and requires an
  operator-assisted decommission. Account identities and browser sessions are
  not tenant-owned and remain. The live qualification drill uses a fresh
  tenant, verifies token revocation after cleanup, and emits only boolean,
  secret-free evidence.
- Worker-B2 `DELETE /pins/:requestid` marks pin metadata as `deleted`; it does
  not promise cryptographic erasure of previously archived content-addressed
  bytes.
- B2 staging cleanup uses `pnpm b2:pins` and requires explicit filters. Deletion
  outside a prefix containing `staging` requires `--force-non-staging-delete`.
  Worker-B2 runtime and cleanup both validate `KOTOBASE_B2_PREFIX` / `--prefix`
  as slash-separated key prefixes up to 120 characters without empty, dot,
  dot-dot, backslash, whitespace, unsupported punctuation, or secret-like
  segments.
  `KOTOBASE_B2_ENDPOINT` must be an origin-only HTTPS Backblaze B2 S3 endpoint
  without embedded credentials, path, query, or fragment; `KOTOBASE_B2_BUCKET`
  must be a 6-63 character S3-compatible bucket name using lowercase letters,
  digits, dots, or hyphens, must begin and end with a letter or digit, and must
  not use adjacent dots, IPv4-address form, or reserved bucket-name prefixes or
  suffixes. `KOTOBASE_B2_KEY_ID` must be a single printable token without
  whitespace, commas, or slashes, and `KOTOBASE_B2_APP_KEY` must be a
  single-line printable secret.
  Optional `KOTOBASE_B2_REGION` must be 1 to 32 lowercase letters, digits, or
  hyphens, starting and ending with a letter or digit.
  `--older-than-hours` filters must be positive decimal integers at most `8760`.
  Delete-mode `--name-prefix` filters must be 4 to 80 printable single-line
  characters, must not have leading or trailing whitespace, and must not contain
  secret-like values.
  `--did` filters must be tenant DID values on one printable line without
  slashes, backslashes, or secret-like values.
  Human-readable `pnpm b2:pins` output sanitizes record fields to single-line
  cells before writing operator logs; use `--json` for raw machine-readable
  inspection. `--json` output must not include B2 key IDs, app keys,
  authorization headers, signatures, or raw or unrecognized pin metadata fields.
  `--json` output is a projection limited to object key, tenant DID, pin ID,
  name, CID, status, created timestamp, and size.
  Cleanup fails closed if `pnpm b2:pins` cannot read matching pin metadata
  before deletion. Matching pin metadata must include valid tenant DID, pin ID,
  CID, status, and ISO UTC creation timestamp fields before audit or delete
  output is emitted. Cleanup metadata timestamps must round-trip as real UTC
  instants and must not rely on calendar-date normalization. Required cleanup
  metadata string fields must be single-line
  printable text without secret-like values, and tenant DID, pin ID, and CID
  fields must not contain slash or backslash characters. Cleanup metadata
  tenant DID values must also use tenant DID syntax without whitespace.
  Cleanup metadata pin ID values must be printable pin tokens up to 128
  characters.
  Cleanup metadata status values are limited to `queued`, `pinning`, `pinned`,
  `failed`, and `deleted`.
  Cleanup metadata CID values must parse as IPFS CIDs.
  Optional cleanup metadata `name` must be printable single-line text up to 200
  characters and must not contain secret-like values.
  Optional cleanup metadata `size_bytes` must be a non-negative safe integer.
  Cleanup metadata tenant DID and pin ID must match the
  `pins/{tenant_did}/{pin_id}.json` object key.
- CACAO nonce replay markers are retained to prevent replay. They store a
  nonce hash, not the raw CACAO nonce. They should not be manually deleted
  unless the operator accepts replay risk for the affected namespace.
- The B2 multi-region drill deletes only its three exact unique synthetic keys
  from both targets and then confirms absence. Its evidence contains region
  labels, counts, timings, policy limits and booleans; it excludes B2 endpoints,
  bucket names, object keys, key IDs, app keys, authorization material and
  payload bytes. A cleanup failure is an operational incident, not permission
  to list or broadly delete either bucket.
- Content-addressed CIDs may remain retrievable from IPFS peers, gateways,
  caches, B2 archives, or other holders even after local metadata is deleted.
- Edge-request audit receipts use a bounded KV TTL. `AUDIT_RETENTION_DAYS`
  controls it and defaults to 365 days. `/v1/audit/export` emits canonical,
  paginated NDJSON with an exact-byte SHA-256 response header; this provides
  export integrity, not proof that the best-effort KV writer could not omit or
  alter a receipt.

## Contributor Rules

- Do not add secrets, tenant data, private CIDs, raw CACAOs, bearer tokens, or
  sensitive request bodies to code, tests, fixtures, docs, screenshots, issues,
  or logs.
- Prefer `x-kotobase-request-id`, Cloudflare `cf-ray`, Worker version ID,
  endpoint, status code, and sanitized error class for debugging. Public API
  error responses must not include B2 object keys, upstream response bodies,
  credentials, CACAOs, or bearer tokens.
- Any change that adds a new stored field, object prefix, log field, or public
  metadata field must update this document, `docs/ENVIRONMENT.md` when relevant,
  and the metadata checks.

## Non-Goals

- This document does not claim GDPR/medical/financial regulated-tier compliance.
- It does not provide a right-to-erasure guarantee for immutable content-addressed
  payloads.
- Regulated retention, crypto-shred, audit PII encryption, and legal-hold policy
  remain regulated-tier design work tracked in ADR-2606060003 and
  `docs/adr/regulated-tier-maturity.md`.
