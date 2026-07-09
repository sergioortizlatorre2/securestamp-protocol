# ADR-006: Canonical Action Receipts — SecureStamp never signs a caller-declared verdict

## Status
Accepted — 2026-07-09

## Context

A core deliverable of Proof-of-Intent (see
[ADR-005](ADR-005-proof-of-intent-pivot.en.md) and
[Protocol v0.2](../protocol/SECURESTAMP-PROTOCOL-v0.2.en.md)) is a **verifiable Action
Receipt**: a signed (ES256) record of a verified action that a third party can check
without an account at `GET /api/action/receipts/<receiptId>`.

A receipt is only as trustworthy as the rule governing what gets signed. If the
receipt-issuing surface accepted a verdict, intent, or "safe next step" *declared by the
caller* and signed it, then any authenticated caller could fabricate a cryptographically
valid "SecureStamp Action Receipt" asserting `verified_action` — including for a
`payment_request` — indistinguishable from a legitimate one. That would let a party
defraud a third party with a receipt "verified by SecureStamp" that never passed through
any verdict computation. It directly contradicts the doctrine *No agent action without
Proof-of-Intent*.

## Decision

**SecureStamp never signs a verdict declared by the caller.** A verifiable Action
Receipt may originate from exactly two canonical sources, and from nowhere else:

1. **`authorize_action`** — computes a verdict with the verdict engine over
   tenant-verified data (counterparty registry, policy, instruction fingerprint), signs
   it, persists it, and returns a server-generated `receiptId`.
2. **Action Challenge resolution** — a single-use, token-gated confirm/reject transition
   computes `challenge_confirmed` / `challenge_rejected` from a real state change, signs
   it, and persists it.

The `issue_action_receipt` MCP tool is therefore a **lookup-and-re-affirm** operation,
not an issuance operation:

- Its only real input is a `receiptId` (plus an optional `detectedIntent` that must
  match the recorded one, acting as an action fingerprint).
- It enforces tenant scoping: a receipt owned by another tenant returns the same
  response as "not found," so it is not an existence oracle.
- It returns the **already-signed** token; it never calls the signer again.
- It does **not** accept a caller-declared `verdict`, `reasons`, or `safeNextStep`; the
  input schema rejects them.
- It returns a single unified error for not-found / wrong-tenant / fingerprint-mismatch.
- Both successful re-affirmation and denial are audited, with no sensitive payload.

Provenance is therefore an invariant of the system: only the two canonical flows write
receipts, so every receipt traces back to a verdict SecureStamp actually computed.

## Consequences

**Positive:**
- A public receipt is meaningful: it cannot be forged by a caller asserting a verdict
  about itself.
- No new signing surface, table, or PKI is introduced — re-affirmation reuses existing
  storage and returns an already-signed token.
- `issue_action_receipt` stays visible and enabled in `tools/list`, but its input
  surface no longer accepts anything a caller can invent.

**Negative / costs:**
- Callers cannot "self-issue" a receipt for an action that never went through the
  verdict engine — by design; there is no legitimate use case for that.
- Any future flow that needs to *initiate* a verdict outside `authorize_action` must add
  that logic to the verdict engine behind a server-side computed endpoint — never by
  reopening `issue_action_receipt` to caller-declared data.

## Alternatives considered

- **Sign caller-declared verdicts (original behavior).** Rejected: it makes receipts
  forgeable and defeats the doctrine.
- **Disable `issue_action_receipt` entirely / return `RECEIPT_SOURCE_REQUIRED`.**
  Considered as an emergency fallback; unnecessary because existing storage supported the
  full canonical lookup model.
- **Add a third receipt-writing path for batch/other channels.** Deferred: the correct
  extension is to compute such verdicts in the engine behind a server-side endpoint, not
  to relax the receipt surface.
