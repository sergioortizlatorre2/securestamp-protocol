# SecureStamp Protocol v0.2 — Proof-of-Intent

**Status:** Draft
**Date:** 2026-07-09
**Maintained by:** SecureStamp Foundation
**Supersedes:** [SecureStamp Protocol v0.1](SECURESTAMP-PROTOCOL-v0.1.en.md) *(email trust layer — retained as historical)*
**Repository:** https://github.com/sergioortizlatorre2/securestamp-protocol

---

## Abstract

SecureStamp v0.1 defined a **trust layer for email**: a verifiable seal published by
a sender and checked by a recipient. v0.2 generalizes that idea into a **Proof-of-Intent
protocol for the AI era**.

The doctrine is a single sentence:

> **You do not trust the message. You verify the action.**

Before a person, an organization, or an AI agent executes a sensitive digital operation
— a payment, a change of banking or payout instructions, an approval, an irreversible
account change — SecureStamp verifies five dimensions of that operation:

```
intention · counterparty · channel · policy · action
```

and returns a **verdict**. When the operation is backed by a canonical verdict,
SecureStamp can issue a **verifiable Action Receipt** that a third party can check
independently.

The central claim of the protocol is:

> **No agent action without Proof-of-Intent.**

This document specifies the verification model, the three product pillars that expose
it, the remote **MCP Guard** interface for AI agents, the **canonical Action Receipt**
rule, the channel adapters, and the transparency/records model. It also states plainly
what is production-live today and what is roadmap.

---

## 1. Relationship to v0.1

v0.2 does **not** discard v0.1. The email trust seal (`stamp`), the
`_securestamp.<domain>` DNS TXT record, the `X-SecureStamp` header, and the domain
trust score remain valid and are still implemented. Under v0.2 they are reframed:

- The email/domain trust signals of v0.1 are **inputs** to the `counterparty` and
  `channel` dimensions below — not the whole protocol.
- v0.2 is **additive**: a v0.1 verifier keeps working; a v0.2 verifier additionally
  understands actions, verdicts, receipts, and the MCP interface.

Read v0.1 when you care about *"is this sender who they claim to be?"*. Read v0.2 when
you care about *"should this action be allowed to happen?"*.

---

## 2. Terminology

| Term | Definition |
|---|---|
| **Action** | A sensitive digital operation about to be executed (payment, payout change, approval, instruction change, irreversible account change). |
| **Intent** | The inferred purpose of a message or request that would lead to an action. |
| **Counterparty** | The other party in the action (a vendor, a bank account, a contact, an organization). |
| **Channel** | The medium the request arrived on (email, web, Telegram, WhatsApp, an MCP host). |
| **Policy** | The tenant's rules that constrain what is allowed, when, and by whom. |
| **Verdict** | The result of verification: `allow`, `needs_confirmation`, or `block`. |
| **Action Challenge** | A single-use, token-gated dual-control step used to confirm or reject a risky action out of band. |
| **Action Receipt** | A signed, publicly verifiable record that refers to a **canonical** verdict or challenge resolution computed by SecureStamp. |
| **MCP Guard** | The remote Model Context Protocol server that exposes verification to AI agents and hosts. |
| **Verdict engine** | The server-side component that computes a verdict from tenant-verified data. It is the **only** producer of authoritative verdicts. |

---

## 3. Doctrine and central claim

Two invariants govern the entire protocol:

1. **Verification is about actions, not messages.** A message that "looks legitimate"
   proves nothing. SecureStamp verifies the *action the message would cause*.
2. **SecureStamp never signs a verdict declared by the caller.** An authoritative
   verdict can only be produced by the SecureStamp verdict engine over
   tenant-verified data, or by the resolution of a real Action Challenge. This is
   what makes an Action Receipt meaningful (see §7).

From these follows the claim printed on every surface: **No agent action without
Proof-of-Intent.**

---

## 4. The five verification dimensions

A verification request is evaluated across five dimensions. Each contributes to the
verdict; a failure in a high-risk dimension can force `needs_confirmation` or `block`
regardless of the others.

| Dimension | Question answered | Example signals |
|---|---|---|
| **Intention** | What is this request actually trying to do? | message intent classification, risk category, urgency/pressure cues |
| **Counterparty** | Is the other party known and consistent? | tenant counterparty graph, prior interactions, v0.1 domain trust, identity match |
| **Channel** | Did this arrive on an expected, trusted medium? | email/web/Telegram/WhatsApp/MCP origin, channel-trust rules, brand-claim boundary |
| **Policy** | Does the tenant's policy allow this now? | approval thresholds, dual-control rules, allowlists, plan limits |
| **Action** | Is the concrete operation itself safe? | instruction fingerprint, payout/account-change detection, irreversibility |

The verification is **advisory and non-executing**: SecureStamp authorizes; it does
not perform the action.

---

## 5. The three pillars

The verification model is exposed through three products. They share the same verdict
engine and receipt model.

### 5.1 VendorShield — counterparty & sensitive operations
Verification of counterparties and high-risk operations: payments, vendors, invoices,
bank-account and payout changes, changes of instructions, and approvals. Answers
*"is this counterparty and this money-movement safe to act on?"*.

### 5.2 Guardian — human & channel protection
Protection of people and channels: Brand Claim Requests, alerts, user education, and
channel-level trust across email, web, Telegram, and WhatsApp. Answers *"is this
inbound thing safe for a human to trust and act on?"*.

### 5.3 MCP Guard — Agent Trust Layer
A remote MCP interface that AI agents, assistants, and workflows consult **before**
executing a sensitive action. Answers *"as an autonomous agent, am I allowed to do
this?"*. Specified in §6.

---

## 6. MCP Guard (Agent Trust Layer)

MCP Guard is a remote, always-on [Model Context Protocol](https://modelcontextprotocol.io)
server. An agent about to act consults it first; the Guard returns a verdict and,
optionally, a receipt. The Guard **authorizes; it does not execute** — it never moves
money, deletes data, or performs destructive operations.

### 6.1 Endpoint

| Environment | Endpoint | Status |
|---|---|---|
| Production | `https://mcp.securestamp.online/mcp` | **Live** (HTTPS) |

The manifest is served at `/.well-known/securestamp-mcp.json`.

### 6.2 Tools

| Tool | Purpose |
|---|---|
| `authorize_action` | Compute a verdict (`allow` / `needs_confirmation` / `block`) for an action. |
| `analyze_message_intent` | Classify the intent and risk of a message. |
| `verify_counterparty` | Check a counterparty against the tenant's known graph. |
| `create_action_challenge` | Create a dual-control challenge for a risky action. |
| `get_safe_next_step` | Recommend the safe next step given the current context. |
| `issue_action_receipt` | Re-affirm an already-computed receipt by `receiptId` — it never signs a caller-declared verdict (see §7). |

### 6.3 Verdicts

`authorize_action` returns one of:

- `allow` — the action is consistent with counterparty, channel, and policy.
- `needs_confirmation` — the action requires an out-of-band Action Challenge before
  proceeding.
- `block` — the action must not proceed.

### 6.4 Authentication

A single `Authorization: Bearer <token>` header carries the credential; the server
routes by prefix.

1. **API key — `ss_live_…`** — server-to-server (agents, backends, machines). Stored
   only as a SHA-256 hash; one-time reveal; created from the `.online` dashboard.
2. **Delegated session — `ss_sess_…`** — a logged-in `.online` user authorizes an MCP
   client via a short-lived pairing code, which the client exchanges for a delegated
   session token. Hash-only, revocable, tenant-scoped.
3. **Stdio wrapper** — for hosts that only speak stdio, a local wrapper
   (`securestamp-mcp-guard`) exposes the same tools over the local process using an
   `ss_live_` key.

Both bearer modes resolve to the same internal identity (`userId` / `orgId` /
`allowedTools` / rate limit); the downstream pipeline is identical.

### 6.5 Non-goals (V1)

- Does **not** execute the action, move money, or perform destructive operations.
- Does **not** act on the message content — it verifies the action.

---

## 7. Canonical Action Receipts

An **Action Receipt** is a signed (ES256), publicly verifiable record of a verified
action. Public verification is available without authentication at:

```
GET https://securestamp.online/api/action/receipts/<receiptId>
```

The defining rule of the protocol:

> A verifiable receipt may originate **only** from a canonical verdict SecureStamp
> itself computed — never from a verdict declared by the caller.

### 7.1 The two — and only two — canonical sources

1. **`authorize_action`** computes a verdict over tenant-verified data (counterparty
   registry, policy, instruction fingerprint), signs it, persists it, and returns a
   `receiptId`.
2. **Action Challenge resolution** — when a challenge is confirmed or rejected via its
   single-use, token-gated endpoint, the resulting `challenge_confirmed` /
   `challenge_rejected` verdict is computed from a real state transition, signed, and
   persisted.

Both emit their receipt internally, from data they computed themselves.

### 7.2 `issue_action_receipt` is lookup, not issuance

`issue_action_receipt` accepts a `receiptId` (and, optionally, a `detectedIntent`
that must match) and **re-affirms** an existing canonical receipt. It:

- looks up the receipt by its server-generated `receiptId`;
- enforces tenant scoping (a receipt belonging to another tenant is indistinguishable
  from "not found");
- returns the **already-signed** token — it never re-signs anything;
- returns a single unified error for not-found / wrong-tenant / fingerprint-mismatch,
  so it cannot be used as an existence oracle.

It does **not** accept a caller-declared `verdict`, `reasons`, or `safeNextStep`.
This is what lets a third party rely on a receipt: it always traces back to a verdict
the SecureStamp engine produced.

---

## 8. Channel adapters

Verification and Guardian protection are delivered across multiple channels. A channel
adapter maps an inbound request on that medium into the five-dimension model, and maps
verdicts/alerts back out.

| Channel | Status |
|---|---|
| **Email / Web** | Live (v0.1 seal + v0.2 verification and public verify pages). |
| **Telegram** | Live via configured channel integrations — Channel-Trust connector. |
| **WhatsApp** | Live via configured channel integrations — Channel-Trust connector. |
| **MCP hosts** | **Live** — remote MCP Guard endpoint (§6). |

Telegram and WhatsApp are supported through configured channel integrations for
Proof-of-Intent workflows. Availability depends on the deployed SecureStamp channel
connector and the policies of each messaging platform. They answer channel-trust and
intent questions for humans in those apps, and SecureStamp claims no official or native
partner status with either platform. Conformance of a given *third-party* MCP host or
client is a separate matter — see §12.

---

## 9. Transparency and records (the "ledger")

Security-relevant history in SecureStamp is kept in **append-only Merkle transparency
logs** that support **inclusion proofs** (this entry is in the log) and **consistency
proofs** (the log was only appended to, never rewritten). This is the mechanism that
actually ships today. It backs, for example:

- **Key Transparency** for end-to-end-encrypted identity keys (publish/revoke are
  appended; clients can prove inclusion and monitor their own key history).
- **Abuse / consultation transparency** for channel-trust events.

> **Roadmap, non-normative:** a permissioned distributed ledger (Hyperledger Fabric)
> was proposed in [ADR-004](../adr/ADR-004-hyperledger-fabric-ledger.en.md) as a
> future multi-operator substrate. It is **not** the current implementation and no
> conformance is claimed for it. The normative substrate for v0.2 is the append-only
> Merkle transparency log described above. See ADR-004's roadmap banner.

---

## 10. Security considerations

- **Fail-closed everywhere.** Missing/expired/revoked credentials, missing scopes, or
  absent verdicts result in denial, never in a permissive default.
- **No caller-declared verdicts.** See §3 and §7 — the load-bearing invariant.
- **Hash-only credentials.** API keys and delegated session tokens are stored as
  SHA-256 hashes; one-time reveal; revocation is immediate.
- **Tenant scoping.** Every lookup is scoped to the calling tenant; cross-tenant
  existence is never disclosed.
- **Rate limiting** per tool and per tenant/session.
- **Audit without sensitive payload.** Audit events record that an action was verified,
  not the sensitive content of the action; IP/UA are hashed.
- **Least-privilege** operational identity and environment isolation
  (production/staging).

---

## 11. Versioning and compatibility

- The protocol uses semantic versioning with a `v` prefix. v0.2 is **additive** over
  v0.1.
- Integration points keep their version markers (DNS TXT `v=1`, `X-SecureStamp` `v=1`,
  REST `/v1/`); the MCP interface advertises its own protocol version in the manifest.
- A change is **major** if it breaks a receipt/verdict contract, a credential format,
  or an existing integration point.

---

## 12. Production-live vs roadmap (stated plainly)

**Live today:**

- MCP Guard production endpoint (`https://mcp.securestamp.online/mcp`), both auth modes.
- The six MCP tools and the `allow` / `needs_confirmation` / `block` verdict model.
- Canonical Action Receipts with public verification.
- Telegram and WhatsApp Channel-Trust, live via configured channel integrations.
- Append-only Merkle transparency logs (Key Transparency, channel-trust transparency).

**Roadmap / not yet claimed:**

- Conformance with any specific third-party MCP host or client (e.g. desktop agents,
  IDE assistants) — **not** independently smoke-verified; not asserted.
- A published stdio wrapper package on a public registry.
- Marketplace listings.
- A permissioned distributed ledger (Hyperledger Fabric) as a multi-operator substrate
  (ADR-004) — proposal only.

No compatibility is claimed that has not been verified end-to-end.

---

*End of document — SecureStamp Protocol v0.2 (Proof-of-Intent).*
*Technical collaboration: Ivan.*
