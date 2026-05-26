# SecureStamp Protocol v0.1

**Status:** Draft  
**Date:** 2026-05-24  
**Maintained by:** SecureStamp Foundation  
**Repository:** https://github.com/securestamp/protocol

---

## Abstract

The SecureStamp protocol defines a standard mechanism for email senders to publish a verifiable trust seal (`stamp`), and for recipients or intermediary systems to independently verify that seal. The protocol operates on top of existing DNS infrastructure, standard SMTP headers, and a public HTTP API. A distributed ledger based on Hyperledger Fabric ensures the immutability of the history of issued and revoked stamps.

---

## 1. Terminology

| Term | Definition |
|---|---|
| **Stamp** | A cryptographic seal issued by an approved node for an (orgId, domain) pair |
| **StampId** | UUID v4 that uniquely identifies a stamp in the ledger |
| **Token** | JWT signed with the issuing node's private key; references the stamp |
| **Node** | A server operated by a foundation-approved organization; writes to the ledger |
| **Score** | Integer 0–100 representing the domain's trust level |
| **Channel** | Hyperledger Fabric channel `securestamp-main` where all events reside |

---

## 2. Email integration

The protocol defines three independent integration points. All three can coexist; the presence of any one of them enables verification.

### 2.1 DNS TXT record

The domain owner publishes a TXT record in their DNS zone:

```
_securestamp.example.com.  IN  TXT  "securestamp=v=1; id=<stamp_id>; url=https://securestamp.org/verify/<token>"
```

**Fields:**

| Field | Description |
|---|---|
| `v=1` | Protocol version. Required. |
| `id=<stamp_id>` | UUID v4 of the stamp in the ledger. Required. |
| `url=<url>` | Canonical public verification URL. Required. |

The record is published under the `_securestamp.<domain>` subdomain to avoid interfering with existing TXT records (SPF, DKIM, etc.).

**Full example:**
```
_securestamp.acmecorp.com.  3600  IN  TXT  "securestamp=v=1; id=f47ac10b-58cc-4372-a567-0e02b2c3d479; url=https://securestamp.org/verify/eyJhbGciOiJFUzI1NiJ9..."
```

**DNS verification process:**
1. Resolve `_securestamp.<domain>` → obtain TXT
2. Parse fields; validate `v=1`
3. Call `GET https://securestamp.org/v1/trust/<stamp_id>` with the obtained `id`
4. Verify that the JWT token at `url` signs the `stamp_id` and is not expired or revoked

### 2.2 Email Header

The sender's mail server injects a custom header into each message:

```
X-SecureStamp: v=1; token=<signed_jwt>; verify=https://securestamp.org/verify/<token>
```

**Fields:**

| Field | Description |
|---|---|
| `v=1` | Protocol version. Required. |
| `token=<jwt>` | JWT signed by the issuing node. Contains `stampId`, `domain`, `score`, `iat`, `exp`. Required. |
| `verify=<url>` | Public stamp verification URL. Required. |

**JWT payload structure:**
```json
{
  "stampId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "domain": "acmecorp.com",
  "orgId": "org_01hwx4...",
  "score": 92,
  "iat": 1716480000,
  "exp": 1748016000,
  "iss": "https://securestamp.org"
}
```

The JWT is signed with `ES256` (ECDSA P-256). The issuing node's public key is published at `https://securestamp.org/v1/keys/<node_id>`.

**Header verification process:**
1. Extract the `X-SecureStamp` header from the message
2. Parse fields; validate `v=1`
3. Verify JWT signature against the issuing node's public key
4. Validate that `domain` in the JWT matches the domain in the `From:` header
5. Verify that the `stampId` is not revoked in the ledger (via API or local cache)

### 2.3 API query

Any system can query the trust status of a domain without relying on DNS or the message itself:

```
GET https://securestamp.org/v1/trust/<domain>
```

**Successful response (200):**
```json
{
  "domain": "acmecorp.com",
  "stamp": {
    "stampId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "score": 92,
    "status": "active",
    "issuedAt": "2026-01-15T10:00:00Z",
    "expiresAt": "2027-01-15T10:00:00Z",
    "issuedBy": "node_foundation_01",
    "signals": {
      "spf": "pass",
      "dkim": "pass",
      "dmarc": "pass"
    }
  },
  "ledgerRef": "https://securestamp.org/v1/ledger/tx/abc123...",
  "verifiedAt": "2026-05-24T12:00:00Z"
}
```

**No stamp registered (200, no stamp):**
```json
{
  "domain": "unknowndomain.com",
  "stamp": null,
  "verifiedAt": "2026-05-24T12:00:00Z"
}
```

**Revoked stamp (200, revoked):**
```json
{
  "domain": "compromised.com",
  "stamp": {
    "stampId": "...",
    "score": 0,
    "status": "revoked",
    "revokedAt": "2026-05-20T08:00:00Z",
    "revocationReason": "PHISHING_DETECTED"
  },
  "verifiedAt": "2026-05-24T12:00:00Z"
}
```

**Rate limits:** 1,000 req/hour for unauthenticated IPs. With `Authorization: Bearer <api_key>`, the limit increases according to plan.

---

## 3. Federated node network

### 3.1 Principles

- Only nodes approved by the SecureStamp Foundation may write to the ledger.
- Each node maintains its own local correlation database (PostgreSQL).
- Nodes synchronize through the Hyperledger Fabric ledger (channel `securestamp-main`).
- Ledger reads are public via a REST API exposed by any node.
- A compromised node can be revoked by the foundation via the Certificate Authority.

### 3.2 Node approval process

```
1. APPLICATION
   The operator submits: organization name, ASN, geographic region,
   technical capacity (uptime SLA), declared node use.
   Form: https://securestamp.org/node-application

2. REVIEW
   The foundation's technical committee evaluates:
   - Reputation of the requesting organization
   - Technical capacity (historical uptime, team)
   - Absence of conflicts of interest with existing nodes
   - Geographic coverage the node would add to the network
   Deadline: 30 business days.

3. CA CERTIFICATE
   If approved: the foundation's CA issues an X.509 certificate to the node.
   The certificate identifies the node in the Fabric channel.
   Validity: 1 year, renewable.

4. JOIN CHANNEL
   The operator runs:
     peer channel join -b securestamp-main.block
   with the issued certificate. The node joins the channel as a peer
   and receives the full ledger (synced from genesis block).

5. OPERATION
   The node can now:
   - Endorse transactions
   - Issue stamps for its registered clients
   - Access the full world state
   - Publish its public verification API
```

### 3.3 Obligations of an active node

- Maintain uptime ≥ 99% in 30-day windows.
- Run the chaincode version within the range `[current - 1, current]`.
- Report anomalies detected in its correlation database to the foundation.
- Not modify endorsement policies locally without channel approval.
- Publish a public endpoint: `https://<node_domain>/v1/health` with a response within 2s.

---

## 4. Block structure (chaincode)

The `securestamp-core` chaincode manages three types of assets in the channel's world state:

### 4.1 StampIssuance

```go
type StampIssuance struct {
    StampId   string `json:"stampId"`   // UUID v4
    OrgId     string `json:"orgId"`     // Organization ID in the registry
    Domain    string `json:"domain"`    // normalized domain
    Score     uint8  `json:"score"`     // 0–100
    Signals   string `json:"signals"`   // JSON: { spf, dkim, dmarc, mx, rdns, ... }
    Timestamp int64  `json:"timestamp"` // Unix epoch ms
    IssuedBy  string `json:"issuedBy"`  // SHA-256 fingerprint of the node's cert
}
```

**World state key:** `STAMP:<stampId>`

**Chaincode validations on issuance:**
- The `domain` must be normalized (lowercase, no trailing dot, no protocol).
- The issuing node (`issuedBy`) must have the attribute `role=node` in its MSP certificate.
- No previous active stamp may exist for the same `(orgId, domain)` pair. If one exists, it must be revoked first.

### 4.2 StampRevocation

```go
type StampRevocation struct {
    StampId   string `json:"stampId"`   // reference to StampIssuance.stampId
    Reason    string `json:"reason"`    // enum: PHISHING_DETECTED | ORG_REQUEST | POLICY_VIOLATION | NODE_COMPROMISE | EXPIRED
    RevokedBy string `json:"revokedBy"` // fingerprint of the node's or foundation's cert
    Timestamp int64  `json:"timestamp"`
}
```

**World state key:** `REVOCATION:<stampId>`

**Effect:** The chaincode updates the `STAMP:<stampId>` asset with `status=revoked`. Revocation is irreversible; a new stamp requires a new `StampIssuance`.

### 4.3 ScoreChange

```go
type ScoreChange struct {
    Domain    string `json:"domain"`
    OldScore  uint8  `json:"oldScore"`
    NewScore  uint8  `json:"newScore"`
    Reason    string `json:"reason"`   // enum: DMARC_ADDED | PHISHING_CAMPAIGN | TYPOSQUATTING_CLUSTER | MANUAL_REVIEW | SIGNAL_DRIFT
    Evidence  string `json:"evidence"` // JSON array of weighted signals
    Timestamp int64  `json:"timestamp"`
}
```

**World state key:** `SCORE_HISTORY:<domain>:<timestamp>`

Score changes are recorded as an append-only history. The current effective score is the `newScore` of the record with the highest `timestamp`.

---

## 5. Correlations in each node's local DB

Each node maintains a local PostgreSQL database independent from the ledger. This DB is not replicated between nodes — it is each operator's private analysis workspace. However, **confirmed** events (issued stamps, revocations, score changes) always originate from the shared ledger.

### 5.1 Temporal domain behavior history

```sql
domain_behavior (
  domain          TEXT NOT NULL,
  observed_at     TIMESTAMPTZ NOT NULL,
  signal_type     TEXT NOT NULL,  -- SPF_CHANGE | DKIM_ADDED | MX_CHANGE | NS_CHANGE | ...
  old_value       TEXT,
  new_value       TEXT,
  source_node_id  TEXT NOT NULL,
  PRIMARY KEY (domain, observed_at, signal_type)
)
```

Purpose: detect domains that abruptly modify their DNS infrastructure (indicator of takeover or campaign preparation).

### 5.2 Typosquatting detection

The correlation engine calculates the **Levenshtein distance** between all domains in the database (without TLD) and the domains registered with the foundation.

**Rule:** If `levenshtein(new_domain, registered_domain) ≤ 3`, the new domain is flagged as a typosquatting candidate and a `WARNING` level alert is generated.

```sql
typosquatting_candidates (
  suspect_domain      TEXT NOT NULL,
  target_domain       TEXT NOT NULL,  -- legitimate domain being targeted
  levenshtein_dist    INT NOT NULL,
  detected_at         TIMESTAMPTZ NOT NULL,
  status              TEXT NOT NULL DEFAULT 'PENDING',  -- PENDING | CONFIRMED | FALSE_POSITIVE
  PRIMARY KEY (suspect_domain, target_domain)
)
```

### 5.3 Individual sender reputation

```sql
sender_reputation (
  email_address   TEXT NOT NULL PRIMARY KEY,
  domain          TEXT NOT NULL,
  seen_count      INT NOT NULL DEFAULT 0,
  last_seen       TIMESTAMPTZ,
  complaint_count INT NOT NULL DEFAULT 0,  -- spam/phishing reports received
  score_override  INT,  -- explicit score assigned by analyst; NULL = use domain score
  notes           TEXT
)
```

A sender can have a different score than their domain. E.g.: a legitimate domain with an employee whose account was compromised.

### 5.4 IPs → domains → coordinated phishing campaigns

```sql
ip_domain_map (
  ip_address   INET NOT NULL,
  domain       TEXT NOT NULL,
  first_seen   TIMESTAMPTZ NOT NULL,
  last_seen    TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (ip_address, domain)
)

phishing_campaigns (
  campaign_id   TEXT NOT NULL PRIMARY KEY,  -- UUID v4
  detected_at   TIMESTAMPTZ NOT NULL,
  confidence    NUMERIC(4,3) NOT NULL,  -- 0.000–1.000
  domains       TEXT[] NOT NULL,        -- participating domains
  ips           INET[] NOT NULL,        -- shared IPs
  evidence      JSONB NOT NULL,
  status        TEXT NOT NULL DEFAULT 'ACTIVE'  -- ACTIVE | MITIGATED | FALSE_POSITIVE
)
```

**Campaign detection heuristic:** two or more domains sharing the same `/24` IP block, with Levenshtein ≤ 3 against the same target domain, that began appearing within a 72-hour window.

---

## 6. Real-time alert system

### 6.1 Triggers

The correlation engine generates an immediate alert when it detects any of the following patterns:

| Alert Type | Condition | Minimum confidence to emit |
|---|---|---|
| `TYPOSQUATTING_DETECTED` | Levenshtein ≤ 3 against a registered domain | 0.7 |
| `PHISHING_CAMPAIGN_DETECTED` | Cluster of domains with shared IPs + Levenshtein | 0.8 |
| `SCORE_DROP_CRITICAL` | ScoreChange where `newScore < 30` and `oldScore ≥ 60` | — (always) |
| `STAMP_REVOKED` | StampRevocation confirmed in the ledger | — (always) |
| `DNS_INFRASTRUCTURE_CHANGE` | NS or MX change in a domain with an active stamp | 0.6 |
| `SENDER_COMPROMISE_SUSPECTED` | Sender with clean history suddenly reported by ≥ 3 recipients | 0.75 |

### 6.2 Delivery channels

Alerts are delivered simultaneously through three channels:

**Email (SMTP)**
```
Subject: [SecureStamp Alert] TYPOSQUATTING_DETECTED — acmecorp.com
```

**Webhook (HTTP POST)**
```
POST https://<org_webhook_url>
Content-Type: application/json
X-SecureStamp-Signature: <hmac-sha256>
```

**WebSocket (real-time dashboard)**
```
ws://securestamp.org/v1/ws/alerts
```
Authenticated clients receive a JSON frame for each alert generated for their subscribed domains.

### 6.3 Alert payload

```json
{
  "alertId": "ale_01hwx4...",
  "alertType": "TYPOSQUATTING_DETECTED",
  "affectedDomain": "acmecorp.com",
  "attackerDomain": "acmec0rp.com",
  "detectionConfidence": 0.91,
  "evidence": [
    {
      "type": "LEVENSHTEIN_DISTANCE",
      "value": 1,
      "description": "Character 'o' replaced by '0' at position 6"
    },
    {
      "type": "IP_SHARED",
      "value": "185.220.101.0/24",
      "description": "IP in same /24 as a previously registered campaign"
    }
  ],
  "timestamp": "2026-05-24T12:00:00Z",
  "recommendedAction": "Verify WHOIS records and report to registrar",
  "ledgerRef": null
}
```

`ledgerRef` is `null` if the alert comes from the local correlation engine before an event is confirmed on the ledger. It is populated with the TX hash once confirmed.

### 6.4 Alert subscription

```
POST https://securestamp.org/v1/alerts/subscribe
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "domains": ["acmecorp.com", "acmecorp.io"],
  "channels": ["email", "webhook", "websocket"],
  "webhook_url": "https://hooks.acmecorp.com/securestamp",
  "webhook_secret": "<hmac_secret>",
  "min_confidence": 0.7
}
```

---

## 7. Protocol versioning

The protocol uses semantic versioning with a `v` prefix. The version is included in all integration points:
- DNS TXT: field `v=1`
- Email header: field `v=1`
- API: path `/v1/`

Changes that increment the major version:
- Backwards-incompatible changes to the JWT token format
- Changes to the chaincode asset format
- Changes to the DNS TXT record structure

The foundation will maintain compatibility with the previous version for a minimum of 12 months after publishing a new major version.

---

## 8. Security considerations

- JWT tokens must be verified against the list of active public keys. A rotated key invalidates all tokens signed with it.
- The `domain` in the JWT must exactly match the domain in the email's `From:` header after normalization (lowercase, IDN-decoded).
- Webhooks must verify `X-SecureStamp-Signature` before processing the payload.
- Nodes must not directly expose Fabric's world state. All reads go through the REST API, which filters sensitive fields (internal certificate fingerprints).
- High-confidence alerts (`>= 0.9`) should be treated as blocking in email filtering pipelines.

---

*End of document — SecureStamp Protocol v0.1*
