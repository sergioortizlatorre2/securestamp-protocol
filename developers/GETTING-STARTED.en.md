# Getting Started — SecureStamp for developers

**[English](GETTING-STARTED.en.md) · [Español](GETTING-STARTED.es.md)**

> **No agent action without Proof-of-Intent.**

This is the practical path: connect something, install something, verify something.
For the concepts, read the [README](../README.md) and
[Protocol v0.2](../protocol/SECURESTAMP-PROTOCOL-v0.2.en.md).

<sub>Endpoints, tool names, dist-tags and versions here were checked live on **2026-09-06**.</sub>

---

## 0. Three things to know before you write code

1. **SecureStamp authorizes; it does not execute.** No SecureStamp tool moves money, deletes
   data, or performs a destructive action. If you are looking for something that *does* the
   thing, this is the wrong layer.
2. **Never send raw message text to a remote tool.** The remote tools take *abstract signals*.
   Body reading happens on the device (SSFML) or in the local stdio wrapper
   (`read_message_request`), never over the wire.
3. **`npm install` resolves `latest`, which is deliberately behind the newest prerelease** on
   the Action Proof line. See [§4](#4-install-the-packages) — and check the known defects in
   [§8](#8-errors-you-should-expect) before you build on a published binary.

---

## 1. Look at the service before you authenticate

Everything here is public — no key needed.

```bash
# service version, deployed commit, tool-catalog hash
curl -sS https://mcp.securestamp.online/version

# the full tool catalog, scopes, limits and doctrine
curl -sS https://mcp.securestamp.online/.well-known/securestamp-mcp.json

# health
curl -sS -o /dev/null -w '%{http_code}\n' https://mcp.securestamp.online/healthz
```

`/version` reports the MCP protocol version (`2025-11-25`), the deployed `commitSha` and a
`catalogHash`. Pin the `catalogHash` if you want to detect a tool-catalog change.

An unauthenticated call to `/mcp` is expected to fail, and it tells you how to authenticate:

```bash
curl -sS -D - -o /dev/null -X POST https://mcp.securestamp.online/mcp \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```
```
HTTP/2 401
www-authenticate: Bearer error="invalid_token",
  scope="openid offline_access profile email",
  resource_metadata="https://mcp.securestamp.online/.well-known/oauth-protected-resource"
```

---

## 2. Connect to the remote MCP Guard

Three auth modes. Pick by *who* is authorizing.

### 2a. API key — `ss_live_…` (a machine authorizes itself)

Create a client and key from `.online` → `/dashboard/mcp` (one-time reveal). Keep it secret;
never commit it.

```bash
BASE=https://mcp.securestamp.online
KEY=ss_live_...

curl -sS $BASE/mcp \
  -H "Authorization: Bearer $KEY" \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Ask for a verdict:

```bash
curl -sS $BASE/mcp \
  -H "Authorization: Bearer $KEY" \
  -H 'content-type: application/json' \
  -d '{
    "jsonrpc":"2.0","id":2,"method":"tools/call",
    "params":{
      "name":"authorize_action",
      "arguments":{
        "actionType":"bank_account_change",
        "counterparty":"vendor@example.com",
        "sourceChannel":"email",
        "amount":18400,
        "currency":"EUR"
      }
    }
  }'
```

`actionType` is a closed enum — `payment_request`, `bank_account_change`,
`monetary_operation_change`, `credential_request`, `mfa_code_request`, `risky_attachment`,
`support_contact`, `software_install`, `document_upload`, `crypto_transfer`,
`identity_verification_request`, `unknown_sensitive_action`. Unknown fields are rejected, not
ignored.

### 2b. Delegated login — `ss_sess_…` (a human authorizes a client)

For when a person should authorize a tool without pasting a long-lived key.

1. In `.online` → `/dashboard/mcp` → **Connect with login** → a one-time pairing code
   (10-minute TTL, single use).
2. The client exchanges the code for a session token:
   ```bash
   curl -sS https://securestamp.online/api/mcp/delegated/exchange \
     -H 'content-type: application/json' \
     -d '{"code":"XXXX-XXXX-XXXX-XXXX-XXXX"}'
   # → { "sessionToken": "ss_sess_...", "expiresAt": "...", "allowedTools": [...] }
   ```
3. Use it against `/mcp` exactly like an API key.
4. Revoke from the dashboard. Sessions last 30 days and there is no refresh — revocation
   fails closed with `401`.

### 2c. OAuth 2.1 — authorization-code + PKCE

For MCP clients that speak OAuth. The protected-resource metadata is served at
`/.well-known/oauth-protected-resource`, and the `401` above hands your client the discovery
URL and scopes. Bearer tokens are accepted **by header only**.

### Remote host configuration

The shape verified against the official `@modelcontextprotocol/sdk`
(`StreamableHTTPClientTransport`) — the library remote-MCP hosts use internally:

```json
{
  "mcpServers": {
    "securestamp": {
      "url": "https://mcp.securestamp.online/mcp",
      "headers": { "Authorization": "Bearer ss_live_..." }
    }
  }
}
```

> We verify against the official SDK, not against any named application's GUI. Confirm the
> exact config shape against your host's own documentation and version.

---

## 3. Run it over stdio instead

For hosts that only speak stdio. Requires **Node.js 20+**, zero runtime dependencies.

```bash
SECURESTAMP_API_KEY=ss_live_... npx -y @securestamp/mcp-guard
```

```json
{
  "mcpServers": {
    "securestamp-guard": {
      "command": "npx",
      "args": ["-y", "@securestamp/mcp-guard"],
      "env": { "SECURESTAMP_API_KEY": "ss_live_..." }
    }
  }
}
```

**What the published `0.1.0` actually exposes is six tools**, not nine: `authorize_action`,
`analyze_message_intent`, `verify_counterparty`, `create_action_challenge`, `get_safe_next_step`
and `issue_action_receipt`. Verified by opening the published tarball on 2026-09-06.

The three execution-layer tools (`request_execution_grant`, `get_execution_status`,
`get_source_envelope`) and the local reader `read_message_request` exist in the source tree but
are **not** in the published build. If you need those, use the remote endpoint — or, for
execution, the separate `@securestamp/execution-guardian-mcp` bridge.

The remote always-on service is the primary surface; the wrapper is the stdio fallback.

---

## 4. Install the packages

Seven public packages, **Apache-2.0**.

> **`npm install` resolves `latest`, and `latest` is often behind.** For four packages the
> newest prerelease sits under **`beta-unverified`**, which means installable and discoverable
> but **not** past the external evidence gate. Choose the tag on purpose.
>
> The sharpest case is the verifier: its `latest` is `0.3.0-beta.1`, whose `bin` does not run.
> The repaired build is `0.3.0-beta.3`, published under `beta-unverified` only.

```bash
# stable, no tag needed
npm install -g @securestamp/cli                          # → 1.0.1
npm install @securestamp/mcp-guard                        # → 0.1.0

# what `latest` gives you on the Action Proof line
npm install @securestamp/action-proof                     # → 0.2.0-beta.1  (not 0.3!)
npm install @securestamp/action-proof-verify              # → 0.3.0-beta.1  (bin does not run)

# same as latest today, but pinned to the beta channel
npm install @securestamp/action-proof@beta                # → 0.2.0-beta.1

# the 0.3 line, explicitly, knowing what the tag withholds
npm install @securestamp/action-proof@beta-unverified     # → 0.3.0-beta.2
npm install @securestamp/action-proof-verify@beta-unverified  # → 0.3.0-beta.3  (working bin)
```

| Package | Deps | Node | `latest` | `beta` | `beta-unverified` |
| --- | --- | --- | --- | --- | --- |
| `@securestamp/cli` | 3 | ≥20 | `1.0.1` | — | — |
| `@securestamp/mcp-guard` | 0 | ≥20 | `0.1.0` | — | — |
| `@securestamp/action-proof-verify` | 0 | ≥20 | `0.3.0-beta.1` | — | `0.3.0-beta.3` |
| `@securestamp/action-registry` | 0 | ≥20 | `0.3.0-beta.1` | — | `0.3.0-beta.2` |
| `@securestamp/action-proof` | 1 | ≥20 | `0.2.0-beta.1` | `0.2.0-beta.1` | `0.3.0-beta.2` |
| `@securestamp/execution-guardian` | 9 | ≥22.5 | `0.2.0-beta.2` | `0.2.0-beta.2` | `0.3.0-beta.2` |
| `@securestamp/execution-guardian-mcp` | 3 | ≥22.5 | `0.2.0-beta.1` | `0.2.0-beta.1` | `0.3.0-beta.2` |

A `—` means the tag does not exist on that package, not that it points somewhere else. No
published version of any package carries an npm provenance attestation. `LICENSE` now ships in
`@securestamp/cli@1.0.1` and `@securestamp/action-proof-verify@0.3.0-beta.3`; the verifier's
`latest` (`0.3.0-beta.1`) still has none.

### The CLI

```bash
npm install -g @securestamp/cli
ss check securestamp.org --json      # no API key required
ss check someone@example.com
ss status                            # needs a key
```

`ss check` runs against the public trust API and returns score, state, level, SPF/DKIM/DMARC
signals and reasons. Keys are stored `0600` at `~/.securestamp/config.json`; `SS_API_KEY` and
`SS_API_BASE` override the file.

`1.0.1` is on `latest`, so a plain install gets it. It repaired three commands that were broken
in `1.0.0` and verified by running them: `ss check` without `--json` and `ss batch` both crashed
with a `TypeError` on a trust field the API does not return, and `ss registry` answered
`HTTP 404` because its route is served by `.org`, not the default `.online`. `1.0.1` also stops
counting an unknown state as trusted, reads its version from `package.json` instead of a
hardcoded string, ships `LICENSE`, and lowers the `engines` floor from `>=22.22.2` — which
blocked Node 20 LTS — to `>=20.0.0`.

> **Pin `>=1.0.1`** if you script against the CLI. On `1.0.0` only `ss check --json`,
> `ss status`, `ss login` and `ss logout` behave.

**Start with the verifier.** It is the entry point of the ecosystem, not an accessory: no
network access, no dependencies, and it is published *ahead of* what it verifies, so a
released verifier accepts a new bundle version before anything emits one. That is why its
`latest` runs ahead of the emitter's.

---

## 5. Verify without an account

Fetch a receipt over HTTP — no key, no login:

```bash
curl https://securestamp.online/api/action/receipts/<receiptId>
# unknown id → HTTP 404 {"error":"Receipt not found"}
```

For real cryptographic verification, do it **offline**. `@securestamp/action-proof-verify`
never contacts SecureStamp, never resolves JWKS over the network, and never trusts a provider
SDK. You supply the trust anchors; it checks every signature, digest, audience, single-use
grant, local-policy binding and transparency inclusion locally.

Use the **library API**. The package declares an `action-proof-verify` binary, but the published
`dist/cli.js` has no shebang in either released version, so the executable does not run — that
needs a release to fix.

```js
import { verifyActionProofBundleOffline, jcs } from '@securestamp/action-proof-verify'

// RFC 8785 canonical JSON — keys sorted, deterministic bytes to hash
jcs({ b: 1, a: [2, { d: 4, c: 3 }] })  // {"a":[2,{"c":3,"d":4}],"b":1}

// It THROWS on a malformed bundle rather than returning { ok: false } — catch it.
try {
  const result = await verifyActionProofBundleOffline({ bundle, trustAnchors })
  console.log(result)
} catch (err) {
  console.error('rejected:', err.message)   // e.g. INVALID_ACTION_PROOF_BUNDLE
}
```

Fail-closed is the point: an unparseable bundle raises rather than resolving to a soft "no".

It accepts `ActionReceiptV2` and `ActionReceiptV3`, and `ActionProofBundleV1` and
`ActionProofBundleV2`. It reports what it could and could not establish — `policyEvidence` as
`verified` or `legacy_untrusted`, `adapterManifestEvidence` as `verified_v1` or
`legacy_opaque` — rather than collapsing both into a single boolean.

The package also ships the RFC 8785 (JCS) interoperability corpus at
`@securestamp/action-proof-verify/vectors/jcs-rfc8785.json`, byte-identical to the emitter's.
You can test a third-party implementation against the corpus without trusting either package.

**SecureStamp never signs a verdict declared by the caller.** `issue_action_receipt` only
re-affirms a receipt SecureStamp already computed and signed, by its `receiptId`, after
checking it belongs to your tenant. See [ADR-006](../adr/ADR-006-canonical-action-receipts.en.md).

---

## 6. The Execution Guardian — you run it, you hold the credentials

The Guardian is the part of the architecture that is *not* ours at runtime.

```
device-signed SourceEnvelope → canonical ActionEffect → cloud ExecutionGrant
      → YOUR Guardian claims it, executes it, signs the claim → ActionReceipt
```

- The daemon runs in **your** environment and is the only process that reads your provider
  credentials. **SecureStamp Cloud never receives them and never executes a provider
  operation.**
- The bridge (`@securestamp/execution-guardian-mcp`) is **credential-free**. It talks to the
  daemon over a Unix socket only — no HTTP listener, no arbitrary URL, no provider SDK, no
  signing key. Socket directory `0700`, socket `0600`. If the daemon is absent it fails closed
  with `GUARDIAN_DAEMON_UNAVAILABLE`.
- A cloud grant is **necessary but never sufficient**: the effective permission is the
  intersection of the grant, the local policy *you* sign, the adapter constraints and the kill
  switches. The cloud can narrow it; it cannot broaden it.
- Grants are audience-bound, short-lived and `maxUses=1`. The daemon claims one durably
  *before* mutation and reconciles ambiguous outcomes rather than blind-retrying.
- Policy time limits are execution gates, not warnings: past `reviewAfter` you get
  `LOCAL_POLICY_REVIEW_REQUIRED`, past `expiresAt` you get `LOCAL_POLICY_EXPIRED` — both
  without provider access.
- The policy signing key is **customer-owned and not recoverable by SecureStamp**. The daemon
  CLI provides a local M-of-N recovery ceremony (`policy init` / `approve` / `recover` /
  `rotate`).

Sandbox is the default. Production is an explicit opt-in that requires
`GUARDIAN_MODE=production_opt_in` plus a customer-signed `GuardianLocalPolicyV1`, and it
remains your responsibility. A signed policy whose declared environment does not equal the
effective mode is rejected; omitting the mode never upgrades the safe default.

**SecureStamp proves what executed through the enrolled Guardian. It cannot prevent
out-of-band provider actions.** Do not use credentials you do not control.

---

## 7. Privacy limits — what is sent, and what never leaves the device

**SSFML** (SecureStamp ML) is the on-device recognition layer inside the email plugins and the
browser extension. It reads the body **locally** — tokenization, entity extraction (money,
auth intents, URLs, attachments, QR), the hybrid signal/rule engine, and candidate-instruction
detection (IBAN/CBU/CLABE/crypto references).

**The body never leaves the device.** What reaches the API is abstract signals, intents and
version metadata.

Concretely, per surface:

| Surface | Sent off the device | Never sent |
| --- | --- | --- |
| SSFML in a plugin | abstract signals, intents, rule ids, attachment/QR flags, engine + pack versions | message body, subject text, attachment contents |
| `analyze_message_intent` | a `signals` object you construct | raw message text |
| `authorize_action` | action type, counterparty, channel, amount/currency, opaque evidence strings | the message that prompted it |
| `get_source_envelope` | opaque `ref_v1:` / `acct_v1:` hashes, content digest, device key id | mailbox identifiers, account numbers, body |
| `read_message_request` (stdio wrapper) | nothing — it runs locally | everything stays in your process |
| `ss check` | the domain or email you pass | nothing else |

Source and account locators are hashed to `ref_v1:<sha256>` and `acct_v1:<sha256>` **before**
signing; the verifier rejects raw mailbox identifiers and account numbers outright. Source
envelopes also recursively reject raw-body-shaped fields, so a future field cannot smuggle a
body into a signed object by accident.

Plaintext monetary instructions should not be sent to remote tools; monetary matching is
fingerprinted server-side instead.

Two versions travel together: the **engine** (`2.5.0`, compiled into the plugin, changes only
when the plugin ships) and the **knowledge pack** (auto-updating from the `stable` channel).

The pack channel is data-only and signed: an ES256 signature over a SHA-256 of the artifact,
public key pinned in the plugin, and a stable build **rejects an unsigned or invalid pack** and
falls back to the bundled data. A pack may only add literal phrases to signal IDs the bundled
engine already knows, within bounded weight limits. It cannot introduce a rule, an operator or
a category — and it **cannot lower a verdict below what the bundled engine would have
produced**.

SSFML output is an *input* to a proposed action, never authority. It cannot elevate
provenance; Action Proof and the Guardian still enforce the registered operation, effect,
policy and approvals independently.

**SSFML v2's language scope is Spanish**, and that is a decision rather than a gap. The
delivery mechanism is complete — schema, compilation, regex-injection escaping, size and cost
limits, signing, allowlist, versioning, atomic activation, rollback, fail-closed fallback. A
second language does not need mechanism; it needs a measured corpus. Writing grammar tables
without one would be inventing the very measurement the channel exists to carry.

---

## 8. Errors you should expect

| What you see | What it means |
| --- | --- |
| `401` + `WWW-Authenticate` on `/mcp` | No token, or an invalid one. Follow `resource_metadata`. |
| `404 {"error":"Receipt not found"}` | The `receiptId` does not exist, or is not yours. |
| `409` on a `requestFingerprint` | The fingerprint does not match the referenced receipt. It is a **correlation** id only — never a provenance claim, and it never changes a verdict. |
| `GUARDIAN_DAEMON_UNAVAILABLE` | The bridge could not reach your daemon over its socket. It fails closed by design. |
| `LOCAL_POLICY_REVIEW_REQUIRED` / `LOCAL_POLICY_EXPIRED` | Your signed local policy is overdue or expired. Execution stops before provider access. |
| `RESOURCE_BINDING_MISMATCH` | A resource, tenant, destination, account, region, role or tool manifest was substituted after the effect was authorized. |
| `BUNDLE_VERSION_MISMATCH` | An `ActionProofBundleV2` was paired with a receipt version that does not match. |
| `INVALID_ACTION_PROOF_BUNDLE` | The offline verifier **throws** this on a malformed bundle. Catch it; it does not return `{ ok: false }`. |
| `action-proof-verify: import: command not found` | The published verifier binary has no shebang. Use the library API instead ([§5](#5-verify-without-an-account)). |
| `ss registry` → `HTTP 404` | Published CLI defect: it targets `.online`, where that route does not exist. Use `SS_API_BASE=https://securestamp.org`. |
| `Invalid API key format` from `ss login` | Keys begin with `ss_live_` or `ss_test_`, not the `sk_live_` shown in the shipped CLI README. |

Every backing call is tenant-scoped, rate-limited and audited. Auth failures, invalid
requests, rate limits, `initialize`/`tools/list` and health checks are never billed.

---

## 9. Where to go next

- [README](../README.md) — the idea, the pillars, what is live and what is not
- [Protocol v0.2](../protocol/SECURESTAMP-PROTOCOL-v0.2.en.md) — the normative spec
- [Whitepaper v0.2](../whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md) — the manifesto and the model
- [ADR-005](../adr/ADR-005-proof-of-intent-pivot.en.md) — why the protocol pivoted
- [ADR-006](../adr/ADR-006-canonical-action-receipts.en.md) — why receipts cannot be caller-declared
- [securestamp.org/en/docs/action-proof](https://securestamp.org/en/docs/action-proof) — the execution-layer docs
- npm: [`@securestamp`](https://www.npmjs.com/org/securestamp)

Protocol issues, ADR proposals and pull requests are welcome — see
[CONTRIBUTING.md](../CONTRIBUTING.md). Report vulnerabilities privately to
**security@securestamp.org**; do not open a public issue.
