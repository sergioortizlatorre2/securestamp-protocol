# ADR-005: Pivot from email trust layer to a Proof-of-Intent protocol

## Status
Accepted — 2026-07-09

## Context

SecureStamp v0.1 (see [Protocol v0.1](../protocol/SECURESTAMP-PROTOCOL-v0.1.en.md))
framed the problem as **sender identity for email**: a verifiable seal that answers
*"is this domain who it claims to be?"*. That problem is real and the seal still ships.

Two forces changed the center of gravity of the problem:

1. **The harm is the action, not the message.** A message that passes SPF, DKIM, and
   DMARC — or that is genuinely from a known sender whose account was compromised — can
   still induce a harmful, often irreversible action (a payment, a payout-detail change,
   an approval). Message authentication does not verify the action.
2. **AI agents remove the human pause.** An autonomous agent reads a request and *acts*.
   There is no moment where a person inspects "this looks legitimate" before money
   moves. The industry now needs a checkpoint that software can consult **before** it
   acts.

The product had, in fact, already grown these capabilities: counterparty verification
(VendorShield), human/channel protection (Guardian), and a remote MCP interface for
agents (MCP Guard), plus canonical Action Receipts. The public protocol documentation
lagged behind, still describing only the v0.1 email trust layer.

## Decision

Adopt **Proof-of-Intent** as the protocol's organizing doctrine and publish it as
**Protocol v0.2** ([spec](../protocol/SECURESTAMP-PROTOCOL-v0.2.en.md)), alongside — not
replacing — v0.1.

- **Doctrine:** *You do not trust the message. You verify the action.*
- **Central claim:** *No agent action without Proof-of-Intent.*
- **Verification model:** every sensitive action is evaluated across five dimensions —
  **intention, counterparty, channel, policy, action** — producing a verdict of
  `allow`, `needs_confirmation`, or `block`.
- **Three pillars** expose the same engine: **VendorShield** (counterparties/money
  movement), **Guardian** (people/channels), **MCP Guard** (AI agents).
- **v0.1 is retained** as a historical spec and reframed as an *input* to the
  `counterparty` and `channel` dimensions, not the whole protocol. v0.2 is additive: a
  v0.1 verifier keeps working.
- **SecureStamp authorizes; it does not execute.** The protocol is a checkpoint, never
  an actor.

## Consequences

**Positive:**
- Public documentation now matches what actually ships (MCP Guard is production-live).
- The protocol addresses the agent era directly, with a checkpoint software can call.
- The email seal is preserved and given a clear role instead of being discarded.

**Negative / costs:**
- Two protocol versions coexist; readers must be pointed to the right one (handled via
  supersedes banners and README).
- Broader scope means a larger conformance surface to document honestly (see ADR-006 for
  the receipts invariant, and the "live vs roadmap" sections of v0.2).

## Alternatives considered

- **Rewrite v0.1 in place.** Rejected: it would erase the email-trust spec as a distinct
  artifact and its still-valid integration points.
- **Publish as v1.0.** Rejected for now: v1.0 implies a stability/backward-compatibility
  guarantee we are not ready to make while the agent-facing surface is still evolving.
- **Leave the docs on v0.1 and only update marketing.** Rejected: it would leave the
  open protocol materially misdescribing the product.
