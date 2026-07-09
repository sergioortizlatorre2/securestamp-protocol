# SecureStamp: Proof-of-Intent for the AI Era

**Version:** 0.2 — Draft
**Date:** July 2026
**Authors:** SecureStamp Foundation
**Supersedes:** [Whitepaper v0.1](SECURESTAMP-WHITEPAPER-v0.1.en.md) *(email trust layer — retained as historical)*
**License:** CC BY 4.0

---

## Manifesto

For forty years we tried to make the **message** trustworthy. We authenticated
senders, signed headers, scored domains. It helped — and it was never enough, because
the thing that hurts you is not the message. It is the **action** the message talks you
into: the wire transfer, the changed bank details, the approval you clicked, the
instruction your AI agent carried out.

The arrival of autonomous agents makes this unavoidable. An agent reads a message and
*acts*. There is no human pause between "this looks legitimate" and "the money is gone."

So SecureStamp changed its question.

> **You do not trust the message. You verify the action.**
> **No agent action without Proof-of-Intent.**

---

## 1. The problem has moved

SecureStamp began (v0.1) as a trust layer for email: a verifiable seal that answered
*"is this sender who they claim to be?"*. That problem is real and the seal still ships.
But the center of gravity of digital harm has moved from **impersonation** to
**induced action** — and, increasingly, **induced action executed by software with no
human in the loop.**

Consider the modern failure:

1. A message (email, chat, a tool call to an AI agent) requests a sensitive operation:
   *"update the payout account,"* *"approve this invoice,"* *"send the payment."*
2. Everything about the message looks fine. It may even pass SPF, DKIM, and DMARC.
3. A human — or an agent acting on the human's behalf — executes.
4. The counterparty was wrong, the instruction was tampered, the channel was hostile,
   or the policy was violated. The action is often irreversible.

No amount of message authentication catches this, because the message was, technically,
authentic. What was never verified was **the action**.

---

## 2. The idea: Proof-of-Intent

SecureStamp verifies an action across five dimensions before it happens:

```
intention · counterparty · channel · policy · action
```

- **Intention** — what is this request actually trying to do?
- **Counterparty** — is the other party known and consistent?
- **Channel** — did this arrive on an expected, trusted medium?
- **Policy** — does the tenant's own policy allow this, now, by this actor?
- **Action** — is the concrete operation itself safe (and reversible)?

The result is a **verdict**: `allow`, `needs_confirmation`, or `block`. When the verdict
is authoritative, SecureStamp can issue a **verifiable Action Receipt** — evidence,
checkable by a third party, that the action was verified.

SecureStamp **authorizes; it does not execute.** It is the checkpoint, not the actor. It
never moves money or performs the operation.

---

## 3. Three pillars

The same verification engine is delivered through three products, for three audiences.

### VendorShield — for finance and operations
Verification of counterparties and sensitive money-movement: vendors, invoices,
payments, bank-account and payout changes, approvals, changes of instruction. *"Is this
counterparty and this transfer safe to act on?"*

### Guardian — for people and channels
Protection of humans across channels — email, web, Telegram, WhatsApp — with Brand
Claim Requests, alerts, and education. *"Is this inbound thing safe for a person to
trust and act on?"*

### MCP Guard — for AI agents
A remote [Model Context Protocol](https://modelcontextprotocol.io) server that agents,
copilots, and workflows consult **before** they act. *"As an autonomous agent, am I
allowed to do this?"* This is the pillar the AI era demands, and it is production-live.

---

## 4. MCP Guard: the Agent Trust Layer

An AI agent about to take a sensitive action calls MCP Guard first. The Guard verifies
the five dimensions and returns a verdict — and, where appropriate, a receipt.

- **Endpoint (production, live):** `https://mcp.securestamp.online/mcp`
- **Six tools:** `authorize_action`, `analyze_message_intent`, `verify_counterparty`,
  `create_action_challenge`, `get_safe_next_step`, `issue_action_receipt`.
- **Two auth modes:** an API key (`ss_live_…`) for machines and backends, and a
  delegated session (`ss_sess_…`) for a human who authorizes a client from their
  `.online` account. A local stdio wrapper covers hosts that only speak stdio.
- **It authorizes; it does not execute.** No money moves through the Guard.

The protocol spec details the interface:
[SECURESTAMP-PROTOCOL-v0.2.en.md](../protocol/SECURESTAMP-PROTOCOL-v0.2.en.md).

---

## 5. Receipts you can actually trust

A receipt is only worth as much as the rule behind it. SecureStamp's rule:

> **SecureStamp never signs a verdict declared by the caller.**

A verifiable Action Receipt can originate from exactly two places: a verdict computed by
`authorize_action` over tenant-verified data, or the resolution of a real, single-use
Action Challenge. The `issue_action_receipt` tool does not *mint* receipts from
caller-supplied claims — it **looks up and re-affirms** a receipt SecureStamp already
computed and persisted, and returns the token that was already signed.

Anyone can verify a receipt without an account:

```
GET https://securestamp.online/api/action/receipts/<receiptId>
```

Because the only writers of receipts are the canonical verdict flows, a "SecureStamp
Action Receipt" always traces back to a decision SecureStamp actually made — not to a
verdict someone declared about themselves.

---

## 6. Channels

Guardian and verification reach people where they are:

- **Email / Web** — the original seal plus public verification pages.
- **Telegram** and **WhatsApp** — Channel-Trust, live via configured channel
  integrations, answering intent- and channel-trust questions for humans inside those
  apps. Availability depends on the deployed SecureStamp channel connector and each
  platform's policies; SecureStamp claims no official or native partner status.
- **MCP hosts** — the remote Agent Trust Layer for software.

---

## 7. Transparency and records

Security-relevant history is kept in **append-only Merkle transparency logs** with
inclusion and consistency proofs — the same family of construction as certificate
transparency. This is what ships: it backs **Key Transparency** for end-to-end-encrypted
identity keys and transparency for channel-trust events, so a client can prove an entry
exists and that the log was never rewritten.

A permissioned distributed ledger (Hyperledger Fabric) remains a **roadmap** idea for a
future multi-operator federation; it is not the current substrate and we do not claim it
as shipped. See [ADR-004](../adr/ADR-004-hyperledger-fabric-ledger.en.md).

---

## 8. Governance

The SecureStamp Foundation maintains the open protocol specification, publishes the
ADRs that record its decisions, and stewards the receipt and verdict contracts that make
receipts trustworthy. `securestamp.online` is the reference commercial implementation;
the protocol itself is open and free to implement. Competition on top of one shared
standard is intentional.

---

## 9. What is live, and what is not

We hold ourselves to one rule about claims: **we do not assert compatibility we have not
verified end-to-end.**

**Live today:**
- MCP Guard production endpoint and both auth modes.
- The six tools and the `allow` / `needs_confirmation` / `block` verdict model.
- Canonical Action Receipts with public verification.
- Telegram and WhatsApp Channel-Trust, live via configured channel integrations.
- Append-only Merkle transparency logs (Key Transparency and channel-trust).

**Roadmap / not yet claimed:**
- Verified conformance with any specific third-party MCP host or client (desktop
  agents, IDE copilots) — not independently smoke-tested; not asserted.
- A published stdio wrapper on a public registry, and marketplace listings.
- A permissioned distributed ledger as a multi-operator substrate (ADR-004).

---

## 10. How to participate

- **As an implementer** — the protocol is open; build a verifier, an agent integration,
  or a channel adapter. Start with
  [SECURESTAMP-PROTOCOL-v0.2.en.md](../protocol/SECURESTAMP-PROTOCOL-v0.2.en.md).
- **As an organization** — register at [securestamp.online](https://securestamp.online)
  to protect your counterparties, people, and agents.
- **As a contributor** — protocol issues, ADR proposals, and pull requests are welcome.
  See [CONTRIBUTING.md](../CONTRIBUTING.md).

---

*SecureStamp Foundation — securestamp.org*
*Licensed under Creative Commons Attribution 4.0 International (CC BY 4.0).*
*Technical collaboration: Ivan.*
