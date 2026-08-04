# Using kotobase as a pin service (external clients)

kotobase (`kotobase.net`) is a content-addressed **pin & hosting** service on top
of [kotoba](https://github.com/etzhayyim/kotoba) — a Datomic-over-IPFS knowledge
graph. A *pin* is a named commit CID in kotoba's quad store; once pinned, the CID
is durable (off-site via CAR-on-B2, see ADR-2606042100) and addressable over IPFS.

This is the guide for **external callers**. The API is XRPC (HTTP POST) over the
`ai.gftd.apps.kotobase.*` namespace; the edge Worker translates to the kotoba
backend and enforces auth + quota.

## Endpoints

| Host | Use |
|---|---|
| `https://kotobase.net` | canonical XRPC API (this Worker) |
| `https://kotobase.gftd.ai` | legacy alias (301 for browsers; XRPC still works) |
| `https://ipfs.kotobase.net/ipfs/<cid>` | public IPFS/IPNS retrieval gateway (live 2026-07-29) |
| ~~`https://mcp.gftd.ai/mcp`~~ | retired — the hostname does not resolve (measured 2026-07-29) |
| ~~`https://ipfs.gftd.ai/ipfs/<cid>`~~ | retired — 530, origin gone; superseded by `ipfs.kotobase.net` |

## 1. Authenticate

There are **two** ways to authenticate. Pick either — every tenant-scoped call
accepts both.

### Option A — Self-sovereign CACAO (no gftd account)

You hold your own key; **no gftd webauth is required**. You sign a
[CACAO](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-74.md)
(CAIP-74 / SIWE) proving control of your DID, and the service
**cryptographically verifies** the signature.

**Supported signer (today): Ed25519 / `did:key`** — CLI keys, and the
server-custodied key `authn.kotobase.net` mints behind a passkey.

> **Not currently supported**, despite what older revisions of this page and
> ADR-2606060002 said: **SIWE / Ethereum (EIP-191 `personal_sign`, secp256k1)**,
> **EIP-1271 / ERC-4337 smart-contract wallets**, and **BIP-322 (Bitcoin)**.
> Those were verified by the Rust kotoba pod, which was retired 2026-06-24; the
> from-scratch cljs backend (`backend.kotobase.net`) and the edge
> (`kotobase.edge-cacao`) both verify Ed25519 only. **There is no live path that
> accepts a secp256k1 CACAO.** Bringing one back is blocked on Keccak-256 /
> secp256k1 at the edge — `kotoba-lang/eth-crypto` is JVM-only, so it would
> have to come from `@noble/hashes` + `@noble/curves`. See ADR-2608010930.

The `kotoba` CLI mints a conforming CACAO **standalone** (nothing is sent to a
server to sign):

```bash
API=https://kotobase.net
SEED=$(openssl rand -hex 32)                 # your secret key (keep it!)
DID=$(kotoba did-derive --seed "$SEED")      # → did:key:z6Mk…  (this is your tenant DID)

# Mint a CACAO authorizing pin operations on your own DID. Use a fresh nonce
# each request (replay-protected server-side).
CACAO=$(kotoba cacao-sign --seed "$SEED" --graph "$DID" \
          --capability kotobase:pin --nonce "$(uuidgen)")

# Send it as `Authorization: CACAO …` + assert your DID via `x-kotoba-did`.
AUTH=(-H "authorization: CACAO $CACAO" -H "x-kotoba-did: $DID")
```

The pod checks the `kotobase:pin` capability **and** that the CACAO's verified
issuer equals your `tenant_did`, so a forged `x-kotoba-did` cannot pass. A CACAO
minted for `graph:query`/`datom:transact` cannot be replayed as pin auth.

In the examples below, replace `-H "authorization: Bearer $TOKEN"` with
`"${AUTH[@]}"` and add `"tenant_did":"$DID"` to the JSON body.

### Option B — gftd JWT (managed)

Obtain a JWT from the gftd auth service (`authn.gftd.ai`); its `sub` is your
tenant DID. The edge Worker verifies the session and forwards the tenant DID to
the backend — here the Worker is the trust boundary (the pod does not re-verify
the JWT signature).

```bash
TOKEN="<your gftd JWT>"
API=https://kotobase.net
```

## 2. Create an account (once)

```bash
curl -s -X POST "$API/xrpc/ai.gftd.apps.kotobase.accountCreate" \
  -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' -d '{}'

# Check tier + quota usage anytime:
curl -s -X POST "$API/xrpc/ai.gftd.apps.kotobase.accountStatus" \
  -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' -d '{}'
# → {"ok":true,"tenantDid":"did:...","tier":"p2p","quotaPins":50,"quotaBytes":536870912,"usedPins":0,...}
```

The P2P plan allows 50 pins / 512 MiB on the shared endpoint and carries no
operator responsibility for availability; Secure Managed raises the quota to
500 GiB / 10,000 pins and puts Gftd Japan K.K. under a 99.9% SLA. See
[`PLANS.md`](PLANS.md).
Worker-B2 responses also keep snake_case compatibility fields such as
`quota_pins`, `quota_bytes`, `used_pins`, and `used_bytes` for older clients.

## 3. Pin content

`pinCreate` takes a human-readable `name` plus **either** an existing `cid`
(Pinata-style) **or** `quads` to write into your graph and pin the resulting
commit CID. Returns the canonical CID.

```bash
# (a) pin an existing CID
curl -s -X POST "$API/xrpc/ai.gftd.apps.kotobase.pinCreate" \
  -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"name":"my-doc","cid":"bafyrei...","size_hint_bytes":4096}'

# (b) write quads and pin the commit
curl -s -X POST "$API/xrpc/ai.gftd.apps.kotobase.pinCreate" \
  -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"name":"my-graph","quads":{"graph":"g1","triples":[
        {"subject":"doc1","predicate":"title","object":"Hello"}]}}'
# → {"ok":true,"pin_id":"pin_...","cid":"bafyrei...","status":"pinning"}
```

Manage pins:

```bash
curl -s -X POST "$API/xrpc/ai.gftd.apps.kotobase.pinList" \
  -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' -d '{}'
curl -s -X POST "$API/xrpc/ai.gftd.apps.kotobase.pinDelete" \
  -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"pin_id":"pin_..."}'
curl -s -X POST "$API/xrpc/ai.gftd.apps.kotobase.usageGet" \
  -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' -d '{}'
# → {"ok":true,"tenantDid":"did:...","pinCount":1,"totalBytes":4096,"quotaPins":50,"quotaBytes":536870912,...}
```

## 4. Retrieve

Pinned content is content-addressed, so in principle any IPFS gateway/node
can resolve it by CID once archived.

**Current status (2026-07-29): `https://ipfs.kotobase.net/ipfs/<cid>` is
live.** It is served by the dedicated retrieval Worker
[`net-kotobase-ipfs`](https://github.com/gftdcojp/net-kotobase-ipfs)
(ADR-2607072000 in the superproject, `MIGRATION.md` Phase 3 here), deployed
as a Cloudflare custom domain. `/ipns/<name>` works the same way.

```bash
curl -s https://ipfs.kotobase.net/ipfs/<cid>      # 200, the content
curl -s https://ipfs.kotobase.net/ipns/<name>     # 200
curl -s https://ipfs.kotobase.net/health          # {"ok":true,...}
```

`ipfs.gftd.ai` is retired and answers 530 — it proxied a self-hosted Kubo pod
that was permanently decommissioned. Do not use it.

**What that gateway does and does not promise.** Nothing has been archived to
B2 yet (`KOTOBASE_B2_ARCHIVE_CIDS=0` on this Worker), so the retrieval Worker
runs with B2 unconfigured and resolves every request through its configured
public-gateway fallback. That is not a degraded mode for you as a caller — a
CID pinned here is on the IPFS swarm, which is exactly what makes it fetchable
by any gateway — but it does mean **retrieval latency and availability
currently depend on the public IPFS network, not on kotobase's own storage.**
When B2 archiving is turned on, the same hostname starts serving from B2 first
and nothing about your calls changes.

If you would rather not depend on that: the CID is portable. Fetch it from any
IPFS node or gateway you run or trust, and use `pinList`/`usageGet` above to
confirm durability independently of any hostname.

## 5. Query the pinned graph (optional)

Because pins live in kotoba's Datom store, you can query them — read-only,
tenant-JWT-authed:

```bash
# Datalog / Datom API (read subset: q, pull, datoms, asOf, since, history, ...)
curl -s -X POST "$API/xrpc/ai.gftd.apps.kotobase.datomic.q" \
  -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"graph":"<graph-cid>","query_edn":"{:find [?s] :where [[?s :title ?t]]}"}'

# SPARQL / Cypher (auxiliary surface over the Datom head)
curl -s -X POST "$API/xrpc/ai.gftd.apps.kotobase.graph.sparql" \
  -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"query":"SELECT ?s WHERE { ?s <title> \"Hello\" }"}'
```

Datom **writes** (`datomic.transact` / `datomic.with`) are available to tenants as
the BaaS write plane (ADR-2606201700): authorize with a self-issued CACAO
(`datom:transact`, audience == node operator DID) and identify the target database
by `db_name`. The edge binds the write to your own namespace
(`kotobase/db/<your-did>/<db_name>`), so you can only write to your own databases.
Lighter-weight tenant writes remain `pinCreate` + `kg.ingest`/`kg.ingest_batch`.

## Durability

Every pinned commit's blocks are packed into one CAR and archived off-site to
Backblaze B2 (CAR-on-B2, ADR-2606042100): an off-site, content-addressed copy
that scales (object count ∝ commits, not blocks). Cold reads are served from B2
via a single ranged GET. The local durable tier is the kubo (IPFS) blockstore on
the pod's PVC.

An operator-only `KOTOBASE_STORAGE_MODE=b2` archive mode now exists for running
the pin control plane without Kubernetes: the Worker stores pin metadata directly
in Backblaze B2 and can optionally archive gateway-produced IPFS/IPNS IPLD CARs
from a configured IPFS gateway. Production remains in backend/Kubo mode. In
Worker-B2 mode the Worker verifies did:key EdDSA CACAO itself, rejects
non-round-tripping or future-issued CACAO timestamps, and uses a Durable Object
tenant lock for quota-safe metadata writes. EIP-191 / smart-account CACAO was
once described as "backend-only"; that backend no longer exists, so it is
unsupported everywhere (see above). See ADR-2606110003, ADR-2608010930.

## Standard IPFS Pinning Service API (`/pins`)

kotobase also implements the
[IPFS Pinning Service API](https://ipfs.github.io/pinning-services-api-spec/),
so **native IPFS tooling works directly** — register the service once (the
access token is your gftd JWT, the endpoint is the base URL), then pin/list/rm:

```bash
ipfs pin remote service add kotobase https://kotobase.net "$TOKEN"
ipfs pin remote add  --service=kotobase --name=my-doc <cid>
ipfs pin remote ls   --service=kotobase
ipfs pin remote rm   --service=kotobase --cid=<cid>
```

Raw HTTP (PSA):

```bash
curl -s -X POST "https://kotobase.net/pins" \
  -H "authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"cid":"bafyrei...","name":"my-doc"}'
# → 202 {"requestid":"pin_...","status":"pinning","pin":{"cid":...,"name":...},...}

curl -s "https://kotobase.net/pins?cid=bafyrei...&limit=10" -H "authorization: Bearer $TOKEN"
curl -s -X DELETE "https://kotobase.net/pins/<requestid>" -H "authorization: Bearer $TOKEN"
```

`requestid` is stable across `POST` / `GET` / `DELETE` (so `rm --cid`, which
resolves cid → requestid → delete, works). Auth is the same Bearer JWT or CACAO
used by the XRPC pin endpoints.
`cid`, `name`, `requestid`, and filter values must be printable single-line
strings without secret-like values. `requestid` is a printable token up to 128
characters; `limit` must be a positive decimal integer at most `1000`;
comma-list filters such as `cid` and `status` must not contain empty values;
`status` must contain only `queued`, `pinning`, `pinned`, `failed`, or `deleted`.

## Roadmap

- **API-key auth** (`sk_live_*`) for non-interactive clients — today auth is a
  gftd-AUTHN JWT or CACAO.
- **`delegates`** in PinStatus (swarm multiaddrs) so the client can push blocks
  for a not-yet-public CID over bitswap — currently empty (relies on the CID
  being reachable via the public DHT / gateway).
