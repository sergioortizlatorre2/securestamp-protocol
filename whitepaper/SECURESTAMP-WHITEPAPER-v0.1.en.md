# SecureStamp: A Trust Layer for Digital Communications

**Version:** 0.1 — Draft  
**Date:** May 2026  
**Authors:** SecureStamp Foundation  
**License:** CC BY 4.0

---

## Abstract

Email was designed for openness, not trust. Forty years after its invention, anyone can impersonate anyone. SPF, DKIM, and DMARC reduced spam but did not solve identity: an email that passes all three checks can still be a phishing attack. SecureStamp proposes a complementary layer — a verifiable, cryptographic, human-readable trust seal — that operates on existing infrastructure without replacing it. This document describes the problem, the proposed solution, the open protocol, the governance model, and the long-term vision.

---

## 1. The Problem: Identity Doesn't Exist in Email

Every day, 3.4 billion phishing emails are sent. The financial damage exceeds $10 billion annually. The root cause is architectural: email has no native identity system.

### What exists today

- **SPF** verifies that the sending IP is authorized by the domain.
- **DKIM** verifies that the message was not altered in transit.
- **DMARC** defines what to do when SPF or DKIM fails.

These protocols solve *transport integrity*. They do not solve *sender identity*. A phishing attacker can register `acmec0rp.com`, configure SPF, DKIM, and DMARC correctly, and send emails that pass all three checks. The recipient has no technical mechanism to distinguish `acmecorp.com` from `acmec0rp.com`.

### What doesn't exist

There is no open, decentralized registry of "this domain is who it claims to be." There is no universal signal that says: "this sender has been verified, has a history, and has committed to a trust standard." There is no visible seal that a non-technical user can trust at a glance.

---

## 2. The Proposed Solution: The Stamp

SecureStamp introduces the concept of a **stamp** — a cryptographic seal, publicly verifiable, associated with a domain.

A stamp is:
- **Cryptographic**: signed with ECDSA P-256, tied to an immutable ledger.
- **Verifiable**: anyone can check `securestamp.org/verify/<token>` in seconds.
- **Revocable**: if a domain is compromised, the stamp is immediately revoked and the revocation is recorded on the ledger.
- **Visual**: a PostalStamp — a vintage stamp with perforation edges — that appears in email signatures, browser extensions, and the public verification page.

```
┌┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┐  ← perforated edges (verifiable)
┊  SECURESTAMP       92¢   ┊  ← denomination = trust score
┊  ┌──────────────────────┐ ┊
┊  │       VERIFIED       │ ┊  ← artwork (today: icon; future: artist design)
┊  └──────────────────────┘ ┊
┊  acmecorp.com             ┊  ← verified domain
┊  ✓ TRUSTED                ┊  ← status
└┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┘
```

The stamp has two natures:
- **Functional**: verifies that the sender is who they claim to be.
- **Visual**: a distinctive, memorable identity element.

---

## 3. How It Works

### 3.1 Trust scoring

The score (0–100) is calculated from multiple signals:

| Signal | Weight | Description |
|---|---|---|
| SPF | 20% | DNS record properly configured |
| DKIM | 25% | Valid signatures on outgoing messages |
| DMARC | 25% | Strict policy (reject/quarantine) |
| Domain age | 10% | Domains registered recently score lower |
| MX reputation | 10% | Mail server history |
| Behavioral history | 10% | Absence of complaints in historical records |

| Score | State | Meaning |
|---|---|---|
| 70–100 | ✓ TRUSTED | Verified domain, clean history |
| 30–69 | ⚠ SUSPICIOUS | Incomplete configuration or recent domain |
| 0–29 | ✗ BLOCKED | Phishing detected or stamp revoked |

### 3.2 Three integration points

A stamp can be verified through:

1. **DNS TXT record**: `_securestamp.example.com` — zero-latency verification from any system.
2. **X-SecureStamp email header**: injected by the sender's server into every outgoing message.
3. **Public API**: `GET https://securestamp.org/v1/trust/<domain>` — for any system without DNS access.

These three methods are independent and complementary. Any one of them is sufficient to verify.

### 3.3 The immutable ledger

Every stamp issuance and revocation is recorded on a permissioned Hyperledger Fabric ledger operated by the SecureStamp Foundation and the federated network of approved nodes. The ledger guarantees:

- **Immutability**: no one can alter the history of a stamp.
- **Public auditability**: anyone can query the complete transaction history.
- **Decentralization**: no single organization controls the ledger.
- **No cryptocurrency**: no tokens, no gas. The protocol is economically independent.

---

## 4. The Open Protocol

The SecureStamp protocol is open, documented, and free to implement.

### Design principles

- **Zero new infrastructure**: works on existing DNS and SMTP.
- **Interoperable**: any email client, mail server, or browser extension can implement verification.
- **Backwards-compatible**: does not interfere with SPF, DKIM, or DMARC.
- **Versioned**: semantic versioning with 12-month backward compatibility guarantee.
- **No single point of failure**: any node can serve the public verification API.

### What the protocol defines

- The format of the DNS TXT record `_securestamp.<domain>`
- The format of the `X-SecureStamp` email header
- The structure of the signed JWT (ES256)
- The REST API for verification and alerts
- The three chaincode assets (StampIssuance, StampRevocation, ScoreChange)
- The trust scoring algorithm and weights
- The node approval and revocation process

### What the protocol does NOT define

- The user interface (each client decides)
- The specific implementation (Go, Node, Python, Rust — all valid)
- The business model of operators (each node chooses how to monetize)

Full protocol: [protocol/SECURESTAMP-PROTOCOL-v0.1.en.md](../protocol/SECURESTAMP-PROTOCOL-v0.1.en.md)

---

## 5. The Federated Network

SecureStamp is not a centralized service. It is a federation of independent nodes, each operated by an approved organization, all sharing a common ledger.

### What a node is

A node is a server that:
- Holds an X.509 certificate issued by the foundation's CA.
- Participates in the Hyperledger Fabric channel `securestamp-main`.
- Issues stamps for its registered clients.
- Publishes a public verification API.
- Maintains a local correlation database for threat detection.

### How to join the network

1. Submit an application at `securestamp.org/node-application`.
2. The technical committee evaluates the organization within 30 business days.
3. If approved, the foundation issues an X.509 certificate.
4. The node joins the channel and receives the full ledger.

We actively seek organizations from diverse geographic regions and sectors: ISPs, security companies, email providers, academic institutions.

---

## 6. Threat Detection

Beyond stamp issuance, each node contributes to a real-time threat detection system:

### Typosquatting detection
The system calculates the Levenshtein distance between all observed domains and registered domains. A distance ≤ 3 generates an automatic alert.

Example: `acmec0rp.com` → distance 1 from `acmecorp.com` → `TYPOSQUATTING_DETECTED` alert.

### Phishing campaign detection
Domains that share the same `/24` IP block, have similar names, and appear in a 72-hour window are automatically grouped as a coordinated campaign.

### Real-time alerts
Via email, webhook, and WebSocket, organizations receive instant notifications when:
- A typosquatting domain targeting them is detected.
- A phishing campaign using their name is identified.
- Their trust score drops critically.
- DNS infrastructure changes on their domain.

---

## 7. Governance

### The SecureStamp Foundation

The SecureStamp Foundation is the governing body of the protocol. It is responsible for:

- Maintaining the open protocol specification.
- Operating the root Certificate Authority.
- Approving and revoking nodes in the network.
- Publishing reference implementations.
- Ensuring backward compatibility.
- Managing disputes between nodes.

### Governance principles

- **Transparency**: all protocol changes are documented as ADRs (Architecture Decision Records) and public.
- **Meritocracy**: decisions are made by the technical committee, not by commercial interests.
- **Independence**: the foundation does not favor any commercial implementation.
- **Openness**: anyone can propose changes through the public repository.

### Commercial relationship

The foundation defines the protocol and maintains the network. `securestamp.online` is the reference commercial implementation. Other organizations may build competing commercial products using the same open protocol. This is intentional: competition improves the ecosystem without fragmenting the standard.

---

## 8. Vision: The Stamp as Identity

We believe that trust in digital communications should be as clear and universal as a physical seal on a document.

The stamp — inspired by the postal stamps that certified the origin of physical letters for centuries — is the visual metaphor that makes cryptographic trust legible to any person.

### The collectible future

Stamps have dual nature. Beyond their functional role, they are visual identity. In the future, organizations will be able to choose artistic stamp designs created by designers — limited editions that make verified identity distinctive and memorable.

This transforms the stamp from a technical security artifact into a brand element: a company's stamp becomes part of its visual identity, like a logo or seal.

The `securestamp.store` marketplace is the home of these collections. Digital philatelists — people who collect stamps from verified brands — complete "albums" of trust. Organizations mint limited editions to build community around their identity.

### Why this matters

1. **For organizations**: a verified stamp differentiates their emails from phishing attacks that use their name.
2. **For end users**: a visible, understandable trust signal without needing to understand SPF/DKIM/DMARC.
3. **For the security community**: an open, auditable, decentralized standard with no commercial dependencies.
4. **For regulators**: an infrastructure for digital identity compliance that doesn't require new legislation.

---

## 9. The Path to v1.0

| Phase | Description | Status |
|---|---|---|
| **Protocol v0.1** | Spec, DNS/header/API integration, Fabric ledger | Draft |
| **Reference implementation** | securestamp.online, trust API, node SDK | In development |
| **First federated node** | External node joins the network | Q3 2026 |
| **Browser extensions** | Gmail / Outlook / Firefox | Q4 2026 |
| **Protocol v1.0** | First stable version, backward compatibility guaranteed | Q1 2027 |
| **Node network (5+)** | Nodes in 3+ geographic regions | Q2 2027 |

---

## 10. How to Participate

### As an implementer
The protocol is open. You can implement a verifier, a stamp emitter, a browser extension, or a mail server integration without any permission. The specification is at [protocol/SECURESTAMP-PROTOCOL-v0.1.en.md](../protocol/SECURESTAMP-PROTOCOL-v0.1.en.md).

### As an organization (stamp issuer)
Register at [securestamp.online](https://securestamp.online) to obtain a stamp for your domain. Plans from free to enterprise.

### As a node operator
If your organization can contribute an approved node to the network, apply at `securestamp.org/node-application`.

### As a contributor
This repository is open. Protocol issues, ADR proposals, and pull requests are welcome.

---

## Appendix: Technical References

- [SecureStamp Protocol v0.1](../protocol/SECURESTAMP-PROTOCOL-v0.1.en.md)
- [ADR-004: Hyperledger Fabric as immutable ledger](../adr/ADR-004-hyperledger-fabric-ledger.en.md)
- Public API: `https://securestamp.org/v1/trust/<domain>`
- Node application: `https://securestamp.org/node-application`

---

*SecureStamp Foundation — securestamp.org*  
*This document is licensed under Creative Commons Attribution 4.0 International (CC BY 4.0).*
