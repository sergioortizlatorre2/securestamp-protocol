# SecureStamp Protocol

**[English](#english) · [Español](#español)**

> **Do not trust the message. Verify the action.**
> **No agent action without Proof-of-Intent.**

<sub>Every endpoint, tool name, dist-tag and version on this page was checked against the deployed
services and the public npm registry on **2026-10-05** — including downloading and opening the
published tarballs. Anything that could not be verified is listed under
[Live / prerelease / pending](#live--prerelease--pending), together with the defects that check
turned up.</sub>

---

## English

### Start here

| If you want to… | Go to |
| --- | --- |
| Understand the idea in five minutes | this page |
| **Connect an agent, assistant or MCP host** | **[Getting Started](developers/GETTING-STARTED.en.md)** |
| **Diagnose and bound what an agent or harness can do** | **[Agents and harnesses](developers/AGENTS-AND-HARNESSES.en.md)** |
| **Check a domain from the terminal right now** | [The CLI](#the-cli) — no API key needed |
| Install the packages | [npm packages](#npm-packages) — *read the dist-tag note first* |
| Verify a receipt or a proof bundle offline | [Getting Started §5](developers/GETTING-STARTED.en.md#5-verify-without-an-account) |
| Read the normative protocol spec | [Protocol v0.2](protocol/SECURESTAMP-PROTOCOL-v0.2.en.md) |
| Understand the execution layer | [Action Proof and the Execution Guardian](#action-proof-and-the-execution-guardian) |
| Know what is solid vs. prerelease vs. broken | [Live / prerelease / pending](#live--prerelease--pending) |

### Proof-of-Intent for the AI era

**SecureStamp is a Proof-of-Intent protocol for the AI era.**
**Do not trust the message. Verify the action.**
**No agent action without Proof-of-Intent.**

It is an open protocol that verifies a **sensitive action** before it happens — whether a
person, an organization, or an AI agent is about to execute it. It checks five dimensions
of the action:

```
intention · counterparty · channel · policy · action
```

and returns a **verdict** (`allow` / `needs_confirmation` / `block`). When the verdict is
authoritative, SecureStamp can issue a **verifiable Action Receipt** that any third party
can check.

SecureStamp **authorizes; it does not execute.** It is the checkpoint, not the actor.

> SecureStamp began as an email and digital-communication trust layer and evolved into a
> general-purpose Proof-of-Intent protocol for sensitive human and agentic actions. The
> email seal is still valid (see v0.1 below) — SecureStamp is not email-only. Read
> [ADR-005](adr/ADR-005-proof-of-intent-pivot.en.md) for the why, and
> [Protocol v0.2](protocol/SECURESTAMP-PROTOCOL-v0.2.en.md) for the what.

### Why it matters now

A message that passes SPF, DKIM, and DMARC can still induce a harmful, irreversible
action — a payment, a payout-detail change, an approval. And AI agents remove the human
pause: they read a request and *act*. SecureStamp gives software a checkpoint to consult
**before** it acts.

### Three pillars

| Pillar | For | Answers |
|---|---|---|
| **VendorShield** | finance & operations | *Is this counterparty and this money movement safe to act on?* |
| **Guardian** | people & channels | *Is this inbound thing safe for a person to trust and act on?* |
| **MCP Guard** | AI agents | *As an autonomous agent, am I allowed to do this?* |

### MCP Guard — the Agent Trust Layer

A remote [Model Context Protocol](https://modelcontextprotocol.io) server agents consult
before acting.

- **Endpoint:** `https://mcp.securestamp.online/mcp`
- **MCP protocol version:** `2025-11-25`
- **Discovery:** [`/.well-known/securestamp-mcp.json`](https://mcp.securestamp.online/.well-known/securestamp-mcp.json)
  (tool catalog, scopes, limits) · [`/version`](https://mcp.securestamp.online/version)
  (service version, commit SHA, catalog hash) · `/healthz`
- **Auth (three modes):** OAuth 2.1 authorization-code + PKCE (an unauthenticated call
  returns `401` with `WWW-Authenticate` pointing at
  [`/.well-known/oauth-protected-resource`](https://mcp.securestamp.online/.well-known/oauth-protected-resource))
  · API key `ss_live_…` for machines · delegated session `ss_sess_…` for a human authorizing
  a client from their `.online` account
- **Stdio hosts:** `@securestamp/mcp-guard` on npm, or `@securestamp/execution-guardian-mcp`
  for the credential-free Guardian bridge

The remote catalog is **nine tools** — the six original guard tools plus the three
execution-layer tools:

| Tool | Purpose |
| --- | --- |
| `authorize_action` | Action Verdict before a sensitive action. Authorizes only; never executes. |
| `analyze_message_intent` | Map abstract signals to sensitive action intents. **Signals only — never raw message text.** |
| `verify_counterparty` | Check a counterparty against the tenant registry. Registry facts only. |
| `create_action_challenge` | Manual dual-control challenge for a known counterparty. |
| `get_safe_next_step` | Compute the Safe Next Step from already-known facts. |
| `issue_action_receipt` | Re-affirm a receipt **already computed and signed** by `authorize_action`, by its `receiptId`. |
| `request_execution_grant` | Single-use grant for one exact, device-sourced effect. SecureStamp never executes it. |
| `get_execution_status` | Tenant-scoped status and receipt reference for a grant. |
| `get_source_envelope` | Latest tenant-scoped signed source envelope from an enrolled device. **Contains no message body.** |

`read_message_request` exists only in the local wrapper's source and is deliberately never
exposed remotely. Note the published `@securestamp/mcp-guard@0.2.1` build ships the first six
remote tools above; its Doctor and Harness binaries are local CLIs, not remote MCP tools — see
[Getting Started §3](developers/GETTING-STARTED.en.md#3-run-it-over-stdio-instead).

```bash
curl -sS https://mcp.securestamp.online/mcp \
  -H "Authorization: Bearer ss_live_..." \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Full connection recipes for all three modes: **[Getting Started](developers/GETTING-STARTED.en.md)**.

### Agents and harnesses

MCP is one route. A coding agent also reaches the shell, files, scripts and direct API calls,
and its harness — mounts, sockets, credential helpers, proxies, network — decides its real
authority. SecureStamp applies the same rule to all of them: **agents can propose; authority
stays outside the agent**, and coverage is limited to declared and tested routes.

- **Doctor** — a static diagnosis of the configuration you already have: MCP server files,
  native client settings, agent workflows and complete harness profiles. It never executes
  what it reads and proposes corrections on a copy.
- **Portable scenario** — one versioned JSON document (`SSPI-execution-scenario`) drives the
  CLI, the library and any dashboard, with idempotent `hold` / `resume` / `stop` orders.
- **Exact Export** — the agent prepares a change; only the frozen, reviewed candidate leaves,
  through the Guardian.
- **Reports** — route result, coverage, integration level and completeness are kept apart;
  an unevaluated route is never reported as protected.

Commands, contracts and limits: **[Agents and harnesses](developers/AGENTS-AND-HARNESSES.en.md)**.
The Doctor and Harness binaries ship in `@securestamp/mcp-guard@0.2.1`, and the portable scenario
CLI in `@securestamp/execution-governance@0.1.1`. See the availability note there.

### Action Proof and the Execution Guardian

Proof-of-Intent answers *may this happen?* **Action Proof** answers the harder question
that follows: *did exactly the authorized thing happen, and can a third party prove it
offline?*

The chain is deliberately split so that no single party can both authorize and execute:

```
enrolled device signs a SourceEnvelope   (opaque refs + digest — never the message body)
      ↓
canonical ActionEffect                   (deterministic provider operation, JCS/SHA-256)
      ↓
SecureStamp Cloud issues an ExecutionGrant   (audience-bound, short-lived, single-use)
      ↓
YOUR Execution Guardian claims and executes  (it holds the provider credentials, not us)
      ↓
ActionReceipt + proof bundle                 (verifiable offline, no SecureStamp call)
```

Two claims are kept apart on purpose: the **grant** is cryptographic proof of *what was
authorized*; the **receipt** is signed evidence of the *outcome* the gateway could
establish. The second is weaker than the first, and the protocol says so.

**Access control limits what software can reach. Execution Authorization bounds the exact
effect it may cause.** They are not synonyms.

Design invariants a developer should know before integrating:

- **SecureStamp Cloud never holds your provider credentials and never executes a provider
  operation.** The Execution Guardian daemon runs in your environment and is the only
  process that reads those credentials.
- A cloud grant is **necessary but never sufficient**. The effective permission is the
  intersection of the grant, the local policy *you* sign, the adapter constraints and the
  kill switches. The cloud can narrow an authorization; it cannot broaden it.
- **`device_signed` is not hardware attestation.** It means a signature by an enrolled,
  non-exportable P-256 WebCrypto key.
- V1 schemas are **closed contracts**: unknown top-level fields are rejected, not ignored,
  and source envelopes recursively reject raw-body-shaped fields.
- All digests use **SHA-256 over RFC 8785 (JCS)**. Both the emitter and the independent
  verifier publish the same interoperability vectors, so an implementation can be tested
  against the corpus without trusting either package.
- Verification is **offline by construction**: `@securestamp/action-proof-verify` has no
  network access, no dependencies, and takes trust anchors from the caller.

### npm packages

Seven packages are public on npm under **Apache-2.0**.

> **Read this before `npm install`.** `npm install <pkg>` resolves the **`latest`** tag, and for
> several packages `latest` is deliberately **older** than the newest prerelease. The `0.3.0`
> prereleases sit under **`beta-unverified`**: installable and discoverable, but they have
> **not** cleared the external evidence gate. Ask for a tag explicitly when you want one.
>
> This matters most for the verifier: `latest` is still `0.3.0-beta.1`, whose `bin` does not
> execute. The repaired build is `0.3.0-beta.3`, and it is reachable only through
> `@beta-unverified`.

| Package | What it is | `latest` | `beta` | `beta-unverified` |
| --- | --- | --- | --- | --- |
| [`@securestamp/cli`](https://www.npmjs.com/package/@securestamp/cli) | Terminal trust checks — `ss check`, `ss registry`, `ss batch`, `ss status`. **Stable.** | `1.0.1` | — | — |
| [`@securestamp/mcp-guard`](https://www.npmjs.com/package/@securestamp/mcp-guard) | Local stdio wrapper plus the Doctor and Harness local CLIs. | `0.2.1` | — | — |
| [`@securestamp/execution-governance`](https://www.npmjs.com/package/@securestamp/execution-governance) | Portable scenario, control-plane and redacted-report contracts plus the `securestamp-execution-governance` CLI. Node `>=22.22.3`. | `0.1.1` | — | — |
| [`@securestamp/action-proof-verify`](https://www.npmjs.com/package/@securestamp/action-proof-verify) | Offline, dependency-free verifier for receipts and proof bundles. No network. **Start here.** | `0.3.0-beta.1` | — | `0.3.0-beta.3` |
| [`@securestamp/action-registry`](https://www.npmjs.com/package/@securestamp/action-registry) | Dependency-free declarative registry of operations. | `0.3.0-beta.1` | — | `0.3.0-beta.2` |
| [`@securestamp/action-proof`](https://www.npmjs.com/package/@securestamp/action-proof) | Protocol primitives: source envelopes, canonical effects, grants, bundles, signing. | `0.2.0-beta.1` | `0.2.0-beta.1` | `0.3.0-beta.2` |
| [`@securestamp/execution-guardian`](https://www.npmjs.com/package/@securestamp/execution-guardian) | The customer-controlled execution daemon. Holds *your* provider credentials. | `0.2.0-beta.2` | `0.2.0-beta.2` | `0.3.0-beta.2` |
| [`@securestamp/execution-guardian-mcp`](https://www.npmjs.com/package/@securestamp/execution-guardian-mcp) | Credential-free stdio MCP bridge to your Guardian, over a Unix socket only. | `0.2.0-beta.1` | `0.2.0-beta.1` | `0.3.0-beta.2` |

Only three packages carry a `beta` tag, and on those it currently points at the same version as
`latest`. `@securestamp/cli` and `@securestamp/mcp-guard` have `latest` only.

The verifier and the registry are published **ahead of** what they verify and what consumes
them, so a released verifier accepts a new bundle version before anything emits one. That is
why their `latest` runs ahead of the emitter's.

**No published version carries an npm provenance attestation.** Verify a tarball by its
integrity hash and its contents, not by assuming a signed build chain.

**MCP Guard and Execution Governance — on the registry.** `@securestamp/mcp-guard@0.2.1`
includes the `securestamp-mcp-doctor` and `securestamp-harness` binaries next to
`securestamp-mcp-guard` (`0.2.0` has them too, but there `npx -y @securestamp/mcp-guard` cannot
pick a command). The separate `@securestamp/execution-governance` package (Apache-2.0, Node
`>=22.22.3`) carries the portable scenario, control-plane and report contracts plus the
`securestamp-execution-governance` binary; use `0.1.1` or later, because `0.1.0` rejects Node
22.23. See [Agents and harnesses](developers/AGENTS-AND-HARNESSES.en.md).

### The CLI

```bash
npm install -g @securestamp/cli
ss check securestamp.org --json
```

`ss check` needs **no API key** and works against the public trust API. `ss status` and quota
reporting need a key (`ss login ss_live_…` or `ss_test_…`, stored `0600` under
`~/.securestamp/config.json`; `SS_API_KEY` also works).

`1.0.1` repaired three commands that were broken in `1.0.0`: `ss check` without `--json` and
`ss batch` both crashed on a trust field the API does not return, and `ss registry` targeted a
host where the route does not exist. It also lowered the `engines` floor to `>=20.0.0`, which had
been `>=22.22.2`. `latest` points at `1.0.1`, so a plain install now gets the repaired build.

### SSFML — on-device recognition

**SSFML** (SecureStamp ML) is the recognition layer that runs **on the device**, inside the
email plugins and the browser extension. It reads the message body **locally** and emits
abstract signals, intents and candidate instructions.

**The message body never leaves the device.** What reaches the API is signals, intents and
version metadata — never the text.

```
message body (local) → SSFML on-device → abstract signals / intents / verdict evidence → API · MCP · action layer
```

Two versions travel together and mean different things:

| | Where it lives | Live today | Changes when |
| --- | --- | --- | --- |
| **Engine version** (`ssfmlVersion`) | compiled into the plugin bundle | `2.5.0` | the plugin itself ships |
| **Knowledge-pack version** (`ssfmlRulesVersion`) | the active signed pack | `stable` channel, 100 % rollout | auto-updates in the background |

The live pack manifest declares `minPluginVersion: 0.7.0`, so a plugin older than that never
receives an update. Current plugin builds are Gmail `0.8.3`,
Outlook/Microsoft 365 `1.10.3` and Safari `1.1.2`. The Gmail extension is published on the Chrome
Web Store and the Outlook add-in on Microsoft AppSource.

The pack channel is **data-only and signed**: a pack manifest carries an ES256 signature over
a SHA-256 of the artifact, the public key is pinned in the plugin, and a stable build rejects
an unsigned or invalid pack and falls back to the bundled data. A pack may only add literal
phrases to signal IDs the bundled engine already knows, within bounded weight limits — it can
never introduce a rule, an operator, or a category, and **it can never lower a verdict below
what the bundled engine would have produced.**

The bounds are numbers in the code, not intentions: at most 512 entries per grammar table and
1 024 across all of them, 512 cues per pack and 2 048 cue states in total, 256 predicate terms,
and 200 characters per copy override. The signature is ES256 (ECDSA P-256 / SHA-256) over a
canonical manifest payload that contains the artifact's own SHA-256, checked against a public
key pinned in the plugin.

SSFML classification is an **input** to a proposed action. It is not authority: it cannot
elevate provenance, and Action Proof and the Guardian still enforce the registered operation,
effect, policy and approval requirements independently.

**SSFML v2 scope is Spanish.** The delivery mechanism is complete — schema, compilation,
regex-injection escaping, size and cost limits, signing, allowlist, versioning, atomic
activation, rollback and fail-closed fallback. What a second language needs is not mechanism
but a measured corpus, and writing grammar tables without one would be inventing the very
measurement the channel exists to carry. The second language is therefore **deferred, not
owed**.

### Verifiable receipts you can trust

SecureStamp **never signs a verdict declared by the caller.** A receipt can only
originate from a canonical verdict (`authorize_action` or an Action Challenge
resolution). Verify one without an account:

```bash
curl https://securestamp.online/api/action/receipts/<receiptId>
```

For full cryptographic verification with no network and no SecureStamp involvement, use
[`@securestamp/action-proof-verify`](https://www.npmjs.com/package/@securestamp/action-proof-verify).
See [ADR-006](adr/ADR-006-canonical-action-receipts.en.md).

### Channels

Email · Web · **Telegram** (live via configured channel integrations) · **WhatsApp**
(live via configured channel integrations) · MCP hosts.

Telegram and WhatsApp are supported through configured channel integrations for
Proof-of-Intent workflows. Availability depends on the deployed SecureStamp channel
connector and the policies of each messaging platform; SecureStamp claims no official or
native partner status with either platform.

### Repository contents

| Path | Description |
|---|---|
| [`developers/GETTING-STARTED.en.md`](developers/GETTING-STARTED.en.md) | **Developer quickstart** — connect, install, verify (English) |
| [`developers/GETTING-STARTED.es.md`](developers/GETTING-STARTED.es.md) | Developer quickstart (Spanish) |
| [`developers/AGENTS-AND-HARNESSES.en.md`](developers/AGENTS-AND-HARNESSES.en.md) | **Agents and harnesses** — Doctor, portable scenario, Exact Export, hold/resume/stop, reports (English) |
| [`developers/AGENTS-AND-HARNESSES.es.md`](developers/AGENTS-AND-HARNESSES.es.md) | Agents and harnesses (Spanish) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.2.en.md`](protocol/SECURESTAMP-PROTOCOL-v0.2.en.md) | **Current** protocol spec — Proof-of-Intent (English) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.2.es.md`](protocol/SECURESTAMP-PROTOCOL-v0.2.es.md) | Current protocol spec (Spanish) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md) | **Current** whitepaper + manifesto (English) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md) | Current whitepaper + manifesto (Spanish) |
| [`adr/ADR-005-proof-of-intent-pivot.en.md`](adr/ADR-005-proof-of-intent-pivot.en.md) | ADR — the pivot to Proof-of-Intent |
| [`adr/ADR-006-canonical-action-receipts.en.md`](adr/ADR-006-canonical-action-receipts.en.md) | ADR — canonical Action Receipts (no caller-declared verdicts) |
| [`adr/ADR-004-hyperledger-fabric-ledger.en.md`](adr/ADR-004-hyperledger-fabric-ledger.en.md) | ADR — Fabric ledger (**roadmap / non-normative**) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.1.en.md`](protocol/SECURESTAMP-PROTOCOL-v0.1.en.md) | v0.1 email trust layer (**historical**) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.1.en.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.1.en.md) | v0.1 whitepaper (**historical**) |

The Action Proof execution-layer contract is published with the packages themselves — the
READMEs of `@securestamp/action-proof` and `@securestamp/action-proof-verify` on npm, and
the docs at [securestamp.org/en/docs/action-proof](https://securestamp.org/en/docs/action-proof).

### Live / prerelease / pending

Each row below was checked against the deployed service or the public registry on
**2026-10-05**. Nothing here is asserted from a document.

**Live — stable, externally checkable**

| Thing | Evidence |
| --- | --- |
| MCP Guard endpoint | `/healthz` `200`, `/readyz` `200`, `/version` `200` |
| MCP protocol version | `2025-11-25`, reported by `/version` |
| Nine-tool remote catalog | `/.well-known/securestamp-mcp.json` |
| Auth required, fail-closed | `/mcp` returns `401` on GET and POST without a token |
| OAuth protected-resource metadata | `/.well-known/oauth-protected-resource` `200` |
| Public receipt lookup | `/api/action/receipts/<id>` — `404` on an unknown id |
| `@securestamp/cli` | `1.0.1` on `latest`; every command run against production |
| `@securestamp/mcp-guard` | `0.2.1` on `latest`; includes Doctor and Harness local CLIs |
| `@securestamp/execution-governance` | `0.1.1` on `latest`; `validate` and `run` work on Node 22.23 |
| SSFML signed pack channel | `stable`, engine `2.5.0`, rollout 100 %, signature present |
| SSFML engine + tests | `MODEL_VERSION = '2.5.0'`; 1 828 tests across 82 files passing |
| Credential-free Guardian bridge | published tarball has 2 deps, no provider SDK, no HTTP listener |

**Prerelease — published, explicitly unverified**

The `0.3.0` prereleases sit under the `beta-unverified` dist-tag and a `server.json` that says
so. Discoverable is not certified; the verified beta still requires the external evidence gate.
For `@securestamp/action-proof`, `execution-guardian` and `execution-guardian-mcp`, `latest`
and `beta` both still point at the 0.2 line. `@securestamp/action-proof-verify@0.3.0-beta.3` —
the build whose `bin` actually runs — is published under `beta-unverified` only; its `latest`
stays at `0.3.0-beta.1` deliberately.

**Pending — known gaps and defects**

- **A default install of the verifier still gets the broken binary.** `latest` is
  `0.3.0-beta.1`, whose `dist/cli.js` has no shebang, so the executable does not run and no
  `LICENSE` ships. The repaired `0.3.0-beta.3` carries both, but only under `beta-unverified`.
  Install `@securestamp/action-proof-verify@beta-unverified`, or use the library API, which works
  on every version.
- `@securestamp/action-proof` still declares a bin named `action-proof-verify`, and installing it
  alongside the verifier makes **the emitter win** — you can run the emitter's CLI believing you
  ran the independent verifier. The verifier keeps the name and the emitter takes `action-proof`,
  but that rename ships only when `@securestamp/action-proof` is next released.
- No published version of any package carries an npm provenance attestation, including the two
  most recent releases.
- The published `@securestamp/mcp-guard@0.2.1` build exposes the same six remote guard tools;
  Doctor and Harness are local CLIs, while the execution-layer tools and local reader remain
  outside the published remote catalog.
- Node-operator materials are **not in this repository**. There is no `node/` directory here,
  so any instruction to `cd securestamp-protocol/node` cannot work.

**Not claimed**

Verified conformance with any *named* third-party MCP host or client through its own GUI —
protocol compatibility is verified against the official `@modelcontextprotocol/sdk`, which is
the library those hosts use, but we do not claim a literal in-app smoke we have not run ·
submission to, or listing in, any MCP registry or marketplace · npm provenance attestations ·
certification of the full grant + Guardian + human-approval chain · a permissioned distributed
ledger (Fabric) as a multi-operator substrate.

*We do not assert compatibility we have not verified end-to-end.*

### Links

- 🌐 Foundation: [securestamp.org](https://securestamp.org)
- 🛠 Developer quickstart: [GETTING-STARTED.en.md](developers/GETTING-STARTED.en.md)
- 🤖 Agents and harnesses: [AGENTS-AND-HARNESSES.en.md](developers/AGENTS-AND-HARNESSES.en.md)
- 📖 Current protocol: [SECURESTAMP-PROTOCOL-v0.2.en.md](protocol/SECURESTAMP-PROTOCOL-v0.2.en.md)
- 📄 Current whitepaper: [SECURESTAMP-WHITEPAPER-v0.2.en.md](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md)
- 📦 npm: [`@securestamp`](https://www.npmjs.com/org/securestamp)

### Contributing

This repository is open. Protocol issues, ADR proposals, and pull requests are welcome.
See [CONTRIBUTING.md](CONTRIBUTING.md).

### Security

Report vulnerabilities to **security@securestamp.org**. Do not open a public issue for a
suspected vulnerability.

### License

Protocol specification and documentation: [CC BY 4.0](LICENSE).
The published npm packages are Apache-2.0.
Technical collaboration: Ivan.

---

## Español

### Empezá acá

| Si querés… | Andá a |
| --- | --- |
| Entender la idea en cinco minutos | esta página |
| **Conectar un agente, asistente o host MCP** | **[Guía de inicio](developers/GETTING-STARTED.es.md)** |
| **Diagnosticar y acotar lo que puede hacer un agente o un harness** | **[Agentes y harnesses](developers/AGENTS-AND-HARNESSES.es.md)** |
| **Chequear un dominio desde la terminal ya mismo** | [El CLI](#el-cli) — sin API key |
| Instalar los paquetes | [Paquetes npm](#paquetes-npm) — *leé primero la nota de dist-tags* |
| Verificar un receipt o un proof bundle offline | [Guía de inicio §5](developers/GETTING-STARTED.es.md#5-verificar-sin-cuenta) |
| Leer la spec normativa | [Protocolo v0.2](protocol/SECURESTAMP-PROTOCOL-v0.2.es.md) |
| Entender la capa de ejecución | [Action Proof y el Execution Guardian](#action-proof-y-el-execution-guardian) |
| Saber qué es sólido, qué prerelease y qué está roto | [Live / prerelease / pendiente](#live--prerelease--pendiente) |

### Proof-of-Intent para la era de la IA

**SecureStamp es un protocolo de Proof-of-Intent para la era de la IA.**
**No se confía en el mensaje. Se verifica la acción.**
**Ninguna acción de agente sin Proof-of-Intent.**

Es un protocolo abierto que verifica una **acción sensible** antes de que ocurra — ya sea
una persona, una organización o un agente de IA quien esté por ejecutarla. Comprueba cinco
dimensiones de la acción:

```
intención · contraparte · canal · política · acción
```

y devuelve un **veredicto** (`allow` / `needs_confirmation` / `block`). Cuando el
veredicto es autoritativo, SecureStamp puede emitir un **Action Receipt verificable** que
cualquier tercero puede comprobar.

SecureStamp **autoriza; no ejecuta.** Es el checkpoint, no el actor.

> SecureStamp comenzó como una capa de confianza para email y comunicaciones digitales, y
> evolucionó hacia un protocolo general-purpose de Proof-of-Intent para acciones sensibles
> humanas y agentic workflows. El sello de email sigue válido (ver v0.1 más abajo) —
> SecureStamp no es sólo email. Leé
> [ADR-005](adr/ADR-005-proof-of-intent-pivot.es.md) para el porqué, y
> [Protocolo v0.2](protocol/SECURESTAMP-PROTOCOL-v0.2.es.md) para el qué.

### Por qué importa ahora

Un mensaje que pasa SPF, DKIM y DMARC puede aun así inducir una acción dañina e
irreversible — un pago, un cambio de datos de payout, una aprobación. Y los agentes de IA
eliminan la pausa humana: leen un pedido y *actúan*. SecureStamp le da al software un
checkpoint para consultar **antes** de actuar.

### Tres pilares

| Pilar | Para | Responde |
|---|---|---|
| **VendorShield** | finanzas y operaciones | *¿Es seguro actuar sobre esta contraparte y este movimiento de dinero?* |
| **Guardian** | personas y canales | *¿Es seguro que una persona confíe y actúe sobre esto que llegó?* |
| **MCP Guard** | agentes de IA | *Como agente autónomo, ¿tengo permitido hacer esto?* |

### MCP Guard — el Agent Trust Layer

Un servidor [Model Context Protocol](https://modelcontextprotocol.io) remoto que los
agentes consultan antes de actuar.

- **Endpoint:** `https://mcp.securestamp.online/mcp`
- **Versión de protocolo MCP:** `2025-11-25`
- **Descubrimiento:** [`/.well-known/securestamp-mcp.json`](https://mcp.securestamp.online/.well-known/securestamp-mcp.json)
  (catálogo de tools, scopes, límites) · [`/version`](https://mcp.securestamp.online/version)
  (versión del servicio, commit SHA, hash de catálogo) · `/healthz`
- **Auth (tres modos):** OAuth 2.1 authorization-code + PKCE (una llamada sin autenticar
  devuelve `401` con `WWW-Authenticate` apuntando a
  [`/.well-known/oauth-protected-resource`](https://mcp.securestamp.online/.well-known/oauth-protected-resource))
  · API key `ss_live_…` para máquinas · sesión delegada `ss_sess_…` para un humano que
  autoriza un cliente desde su cuenta de `.online`
- **Hosts stdio:** `@securestamp/mcp-guard` en npm, o `@securestamp/execution-guardian-mcp`
  para el bridge credential-free del Guardian

El catálogo remoto son **nueve tools** — las seis originales del guard más las tres de la
capa de ejecución:

| Tool | Propósito |
| --- | --- |
| `authorize_action` | Action Verdict antes de una acción sensible. Sólo autoriza; nunca ejecuta. |
| `analyze_message_intent` | Mapea señales abstractas a intents de acción sensible. **Sólo señales — nunca el texto del mensaje.** |
| `verify_counterparty` | Verifica una contraparte contra el registro del tenant. Sólo hechos del registro. |
| `create_action_challenge` | Challenge manual de doble control para una contraparte conocida. |
| `get_safe_next_step` | Calcula el Safe Next Step a partir de hechos ya conocidos. |
| `issue_action_receipt` | Re-afirma un receipt **ya calculado y firmado** por `authorize_action`, por su `receiptId`. |
| `request_execution_grant` | Grant de un solo uso para un efecto exacto, device-sourced. SecureStamp nunca lo ejecuta. |
| `get_execution_status` | Estado y referencia de receipt de un grant, scopeado al tenant. |
| `get_source_envelope` | Último source envelope firmado del tenant desde un dispositivo enrolado. **No contiene el cuerpo del mensaje.** |

`read_message_request` existe sólo en el código del wrapper local y deliberadamente nunca se
expone de forma remota. Ojo: la build publicada de `@securestamp/mcp-guard@0.2.1` incluye
únicamente las primeras seis tools de arriba — ver
[Guía de inicio §3](developers/GETTING-STARTED.es.md#3-correrlo-por-stdio).

```bash
curl -sS https://mcp.securestamp.online/mcp \
  -H "Authorization: Bearer ss_live_..." \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Recetas completas de conexión para los tres modos: **[Guía de inicio](developers/GETTING-STARTED.es.md)**.

### Agentes y harnesses

MCP es una ruta. Un agente de código también llega a la shell, a los archivos, a scripts y a
llamadas directas a APIs, y su harness —mounts, sockets, helpers de credenciales, proxies, red—
decide su autoridad real. SecureStamp aplica la misma regla a todas: **los agentes pueden
proponer; la autoridad queda fuera del agente**, y la cobertura se limita a las rutas declaradas
y probadas.

- **Doctor** — diagnóstico estático de la configuración que ya tenés: archivos de servidores
  MCP, configuración nativa de clientes, workflows con agentes y perfiles de harness completos.
  Nunca ejecuta lo que lee y propone correcciones sobre una copia.
- **Escenario portable** — un único documento JSON versionado (`SSPI-execution-scenario`)
  gobierna la CLI, la biblioteca y cualquier dashboard, con órdenes idempotentes
  `hold` / `resume` / `stop`.
- **Exact Export** — el agente prepara un cambio; sólo sale el candidato congelado y revisado, a
  través del Guardian.
- **Reportes** — resultado de ruta, cobertura, nivel de integración y completitud se mantienen
  separados; una ruta sin evaluar nunca se informa como protegida.

Comandos, contratos y límites: **[Agentes y harnesses](developers/AGENTS-AND-HARNESSES.es.md)**.
Los binarios de Doctor y Harness vienen en `@securestamp/mcp-guard@0.2.1`, y la CLI del escenario
portable en `@securestamp/execution-governance@0.1.1`. Ver la nota de disponibilidad ahí.

### Action Proof y el Execution Guardian

Proof-of-Intent responde *¿esto puede pasar?* **Action Proof** responde la pregunta más
difícil que viene después: *¿pasó exactamente lo autorizado, y puede un tercero probarlo
offline?*

La cadena está partida a propósito, para que ninguna parte pueda autorizar y ejecutar a la vez:

```
un dispositivo enrolado firma un SourceEnvelope   (refs opacas + digest — nunca el cuerpo)
      ↓
ActionEffect canónico                             (operación determinista, JCS/SHA-256)
      ↓
SecureStamp Cloud emite un ExecutionGrant         (audience-bound, corto, un solo uso)
      ↓
TU Execution Guardian lo reclama y ejecuta        (tiene las credenciales del proveedor, no nosotros)
      ↓
ActionReceipt + proof bundle                      (verificable offline, sin llamar a SecureStamp)
```

Dos afirmaciones se mantienen separadas a propósito: el **grant** es prueba criptográfica de
*qué se autorizó*; el **receipt** es evidencia firmada del *resultado* que el gateway pudo
establecer. La segunda es más débil que la primera, y el protocolo lo dice.

**El control de acceso limita qué puede alcanzar el software. La Execution Authorization
acota el efecto exacto que puede causar.** No son sinónimos.

Invariantes de diseño que conviene conocer antes de integrar:

- **SecureStamp Cloud nunca tiene tus credenciales de proveedor ni ejecuta una operación del
  proveedor.** El daemon Execution Guardian corre en tu entorno y es el único proceso que lee
  esas credenciales.
- Un grant de la nube es **necesario pero nunca suficiente**. El permiso efectivo es la
  intersección del grant, la política local que *vos* firmás, las restricciones del adapter y
  los kill switches. La nube puede angostar una autorización; no puede ensancharla.
- **`device_signed` no es atestación de hardware.** Significa una firma con una clave P-256
  WebCrypto no exportable y enrolada.
- Los schemas V1 son **contratos cerrados**: los campos top-level desconocidos se rechazan, no
  se ignoran, y los source envelopes rechazan recursivamente campos con forma de cuerpo crudo.
- Todos los digests usan **SHA-256 sobre RFC 8785 (JCS)**. El emisor y el verificador
  independiente publican los mismos vectores de interoperabilidad, así que una implementación
  se puede testear contra el corpus sin confiar en ninguno de los dos paquetes.
- La verificación es **offline por construcción**: `@securestamp/action-proof-verify` no tiene
  acceso a red, no tiene dependencias, y toma los trust anchors del llamante.

### Paquetes npm

Siete paquetes públicos en npm bajo **Apache-2.0**.

> **Leé esto antes de `npm install`.** `npm install <pkg>` resuelve el tag **`latest`**, y en
> varios paquetes `latest` es deliberadamente **más viejo** que el prerelease más nuevo. Los
> prereleases `0.3.0` están bajo **`beta-unverified`**: instalables y descubribles, pero **no**
> pasaron el gate de evidencia externa. Pedí el tag explícitamente cuando quieras uno.
>
> Donde más importa es en el verificador: `latest` sigue en `0.3.0-beta.1`, cuyo `bin` no
> ejecuta. La build reparada es `0.3.0-beta.3`, y se alcanza únicamente por
> `@beta-unverified`.

| Paquete | Qué es | `latest` | `beta` | `beta-unverified` |
| --- | --- | --- | --- | --- |
| [`@securestamp/cli`](https://www.npmjs.com/package/@securestamp/cli) | Chequeos de confianza desde la terminal — `ss check`, `ss registry`, `ss batch`, `ss status`. **Estable.** | `1.0.1` | — | — |
| [`@securestamp/mcp-guard`](https://www.npmjs.com/package/@securestamp/mcp-guard) | Wrapper stdio local más las CLIs locales Doctor y Harness. | `0.2.1` | — | — |
| [`@securestamp/execution-governance`](https://www.npmjs.com/package/@securestamp/execution-governance) | Contratos del escenario portable, del plano de control y de los reportes redactados, más la CLI `securestamp-execution-governance`. Node `>=22.22.3`. | `0.1.1` | — | — |
| [`@securestamp/action-proof-verify`](https://www.npmjs.com/package/@securestamp/action-proof-verify) | Verificador offline y sin dependencias de receipts y proof bundles. Sin red. **Empezá acá.** | `0.3.0-beta.1` | — | `0.3.0-beta.3` |
| [`@securestamp/action-registry`](https://www.npmjs.com/package/@securestamp/action-registry) | Registro declarativo de operaciones, sin dependencias. | `0.3.0-beta.1` | — | `0.3.0-beta.2` |
| [`@securestamp/action-proof`](https://www.npmjs.com/package/@securestamp/action-proof) | Primitivas del protocolo: source envelopes, efectos canónicos, grants, bundles, firma. | `0.2.0-beta.1` | `0.2.0-beta.1` | `0.3.0-beta.2` |
| [`@securestamp/execution-guardian`](https://www.npmjs.com/package/@securestamp/execution-guardian) | El daemon de ejecución controlado por el cliente. Tiene *tus* credenciales de proveedor. | `0.2.0-beta.2` | `0.2.0-beta.2` | `0.3.0-beta.2` |
| [`@securestamp/execution-guardian-mcp`](https://www.npmjs.com/package/@securestamp/execution-guardian-mcp) | Bridge MCP stdio credential-free hacia tu Guardian, sólo por Unix socket. | `0.2.0-beta.1` | `0.2.0-beta.1` | `0.3.0-beta.2` |

Sólo tres paquetes tienen tag `beta`, y en ésos hoy apunta a la misma versión que `latest`.
`@securestamp/cli` y `@securestamp/mcp-guard` tienen únicamente `latest`.

El verificador y el registry se publican **antes** de lo que verifican y de lo que los consume,
así que un verificador ya liberado acepta una versión nueva de bundle antes de que alguien la
emita. Por eso su `latest` va más adelante que el del emisor.

**Ninguna versión publicada lleva attestation de provenance de npm.** Verificá un tarball por su
hash de integridad y su contenido, no asumiendo una cadena de build firmada.

**MCP Guard y Execution Governance — en el registro.** `@securestamp/mcp-guard@0.2.1` incluye los
binarios `securestamp-mcp-doctor` y `securestamp-harness` junto a `securestamp-mcp-guard` (la
`0.2.0` también, pero ahí `npx -y @securestamp/mcp-guard` no puede elegir un comando). El paquete
aparte `@securestamp/execution-governance` (Apache-2.0, Node `>=22.22.3`) trae los contratos del
escenario portable, del plano de control y de los reportes, más el binario
`securestamp-execution-governance`; usá `0.1.1` o posterior, porque la `0.1.0` rechaza Node 22.23.
Ver [Agentes y harnesses](developers/AGENTS-AND-HARNESSES.es.md).

### El CLI

```bash
npm install -g @securestamp/cli
ss check securestamp.org --json
```

`ss check` **no necesita API key** y funciona contra la API pública de trust. `ss status` y el
reporte de cuota sí necesitan clave (`ss login ss_live_…` o `ss_test_…`, guardada con permisos
`0600` en `~/.securestamp/config.json`; `SS_API_KEY` también sirve).

`1.0.1` reparó tres comandos que estaban rotos en `1.0.0`: `ss check` sin `--json` y `ss batch`
crasheaban por un campo de confianza que la API no devuelve, y `ss registry` apuntaba a un host
donde la ruta no existe. También bajó el piso de `engines` a `>=20.0.0`, que estaba en
`>=22.22.2`. `latest` apunta a `1.0.1`, así que un install pelado ya trae la build reparada.

### SSFML — reconocimiento on-device

**SSFML** (SecureStamp ML) es la capa de reconocimiento que corre **en el dispositivo**,
dentro de los plugins de email y la extensión de navegador. Lee el cuerpo del mensaje
**localmente** y emite señales abstractas, intents e instrucciones candidatas.

**El cuerpo del mensaje nunca sale del dispositivo.** Lo que llega a la API son señales,
intents y metadata de versión — nunca el texto.

```
cuerpo del mensaje (local) → SSFML on-device → señales / intents / evidencia de veredicto → API · MCP · capa de acción
```

Viajan dos versiones y significan cosas distintas:

| | Dónde vive | Hoy en vivo | Cambia cuando |
| --- | --- | --- | --- |
| **Versión del motor** (`ssfmlVersion`) | compilada en el bundle del plugin | `2.5.0` | se publica el plugin |
| **Versión del knowledge-pack** (`ssfmlRulesVersion`) | el pack activo | canal `stable`, rollout 100 % | auto-actualiza en background |

El manifiesto vivo del pack declara `minPluginVersion: 0.7.0`, así que un plugin más viejo que
eso nunca recibe una actualización. Las builds actuales son Gmail `0.8.3`,
Outlook/Microsoft 365 `1.10.3` y Safari `1.1.2`. La extensión de Gmail está publicada en Chrome
Web Store y el complemento de Outlook en Microsoft AppSource.

El canal de packs es **sólo datos y firmado**: el manifiesto lleva una firma ES256 sobre un
SHA-256 del artefacto, la clave pública está pinneada en el plugin, y un build estable rechaza
un pack sin firma o inválido y cae al dato empaquetado. Un pack sólo puede agregar frases
literales a signal IDs que el motor empaquetado ya conoce, dentro de límites acotados de peso
— nunca puede introducir una regla, un operador ni una categoría, y **nunca puede bajar un
veredicto por debajo de lo que habría producido el motor empaquetado.**

Los límites son números en el código, no intenciones: como máximo 512 entradas por tabla
gramatical y 1 024 entre todas, 512 cues por pack y 2 048 estados de cue en total, 256 términos
de predicado, y 200 caracteres por override de copy. La firma es ES256 (ECDSA P-256 / SHA-256)
sobre un payload canónico de manifiesto que contiene el propio SHA-256 del artefacto, contra una
clave pública pinneada en el plugin.

La clasificación de SSFML es una **entrada** a la acción propuesta. No es autoridad: no puede
elevar la procedencia, y Action Proof y el Guardian siguen exigiendo por su cuenta la
operación registrada, el efecto, la política y las aprobaciones.

**El alcance de SSFML v2 es el español.** El mecanismo de entrega está completo — schema,
compilación, escapado contra inyección de regex, límites de tamaño y costo, firma, allowlist,
versionado, activación atómica, rollback y fallback fail-closed. Lo que falta para un segundo
idioma no es mecanismo sino corpus medido, y escribir sus tablas gramaticales sin él sería
inventar la medición misma que el canal existe para transportar. Por eso el segundo idioma
queda **diferido, no adeudado**.

### Receipts verificables en los que confiar

SecureStamp **nunca firma un veredicto declarado por el llamante.** Un receipt sólo puede
originarse en un veredicto canónico (`authorize_action` o la resolución de un Action
Challenge). Verificá uno sin cuenta:

```bash
curl https://securestamp.online/api/action/receipts/<receiptId>
```

Para verificación criptográfica completa, sin red y sin intervención de SecureStamp, usá
[`@securestamp/action-proof-verify`](https://www.npmjs.com/package/@securestamp/action-proof-verify).
Ver [ADR-006](adr/ADR-006-canonical-action-receipts.es.md).

### Canales

Email · Web · **Telegram** (vivo mediante integraciones de canal configuradas) ·
**WhatsApp** (vivo mediante integraciones de canal configuradas) · hosts MCP.

Telegram y WhatsApp están soportados mediante integraciones de canal configuradas para
flujos de Proof-of-Intent. La disponibilidad depende del conector de canal desplegado por
SecureStamp y de las políticas de cada plataforma de mensajería; SecureStamp no reclama
estatus de partner oficial ni nativo con ninguna de las dos plataformas.

### Contenido del repositorio

| Ruta | Descripción |
|---|---|
| [`developers/GETTING-STARTED.es.md`](developers/GETTING-STARTED.es.md) | **Guía de inicio para developers** — conectar, instalar, verificar (español) |
| [`developers/AGENTS-AND-HARNESSES.es.md`](developers/AGENTS-AND-HARNESSES.es.md) | **Agentes y harnesses** — Doctor, escenario portable, Exact Export, hold/resume/stop, reportes (español) |
| [`developers/GETTING-STARTED.en.md`](developers/GETTING-STARTED.en.md) | Guía de inicio (inglés) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.2.es.md`](protocol/SECURESTAMP-PROTOCOL-v0.2.es.md) | Spec **actual** del protocolo — Proof-of-Intent (español) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.2.en.md`](protocol/SECURESTAMP-PROTOCOL-v0.2.en.md) | Spec actual del protocolo (inglés) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md) | Whitepaper + manifiesto **actual** (español) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md) | Whitepaper + manifiesto actual (inglés) |
| [`adr/ADR-005-proof-of-intent-pivot.es.md`](adr/ADR-005-proof-of-intent-pivot.es.md) | ADR — el pivot a Proof-of-Intent |
| [`adr/ADR-006-canonical-action-receipts.es.md`](adr/ADR-006-canonical-action-receipts.es.md) | ADR — Action Receipts canónicos (sin veredictos del llamante) |
| [`adr/ADR-004-hyperledger-fabric-ledger.es.md`](adr/ADR-004-hyperledger-fabric-ledger.es.md) | ADR — ledger Fabric (**roadmap / no-normativo**) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.1.es.md`](protocol/SECURESTAMP-PROTOCOL-v0.1.es.md) | v0.1 capa de confianza de email (**histórico**) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.1.es.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.1.es.md) | Whitepaper v0.1 (**histórico**) |

El contrato de la capa de ejecución de Action Proof se publica con los paquetes mismos — los
READMEs de `@securestamp/action-proof` y `@securestamp/action-proof-verify` en npm, y la
documentación en [securestamp.org/es/docs/action-proof](https://securestamp.org/es/docs/action-proof).

### Live / prerelease / pendiente

Cada fila de abajo se comprobó contra el servicio desplegado o el registro público el
**2026-10-05**. Nada de esto se afirma a partir de un documento.

**Live — estable, comprobable desde afuera**

| Qué | Evidencia |
| --- | --- |
| Endpoint de MCP Guard | `/healthz` `200`, `/readyz` `200`, `/version` `200` |
| Versión de protocolo MCP | `2025-11-25`, informada por `/version` |
| Catálogo remoto de nueve tools | `/.well-known/securestamp-mcp.json` |
| Auth obligatoria, fail-closed | `/mcp` devuelve `401` en GET y POST sin token |
| Metadata OAuth de protected-resource | `/.well-known/oauth-protected-resource` `200` |
| Lookup público de receipts | `/api/action/receipts/<id>` — `404` con un id inexistente |
| `@securestamp/cli` | `1.0.1` en `latest`; todos sus comandos corridos contra producción |
| `@securestamp/mcp-guard` | `0.2.1` en `latest`; incluye las CLIs locales Doctor y Harness |
| `@securestamp/execution-governance` | `0.1.1` en `latest`; `validate` y `run` funcionan en Node 22.23 |
| Canal firmado de packs SSFML | `stable`, motor `2.5.0`, rollout 100 %, firma presente |
| Motor SSFML + tests | `MODEL_VERSION = '2.5.0'`; 1 828 tests en 82 archivos pasando |
| Bridge Guardian credential-free | el tarball publicado tiene 2 deps, sin SDK de proveedor ni listener HTTP |

**Prerelease — publicado, explícitamente no verificado**

Los prereleases `0.3.0` están bajo el dist-tag `beta-unverified` y un `server.json` que lo
declara. Descubrible no es certificado; la beta verificada sigue exigiendo el gate de evidencia
externa. En `@securestamp/action-proof`, `execution-guardian` y `execution-guardian-mcp`,
`latest` y `beta` siguen apuntando a la línea 0.2. `@securestamp/action-proof-verify@0.3.0-beta.3`
—la build cuyo `bin` sí corre— está publicada sólo bajo `beta-unverified`; su `latest` queda
deliberadamente en `0.3.0-beta.1`.

**Pendiente — huecos y defectos conocidos**

- **Un install por defecto del verificador todavía trae el binario roto.** `latest` es
  `0.3.0-beta.1`, cuyo `dist/cli.js` no tiene shebang, así que el ejecutable no corre y no viaja
  ningún `LICENSE`. El `0.3.0-beta.3` reparado trae las dos cosas, pero sólo bajo
  `beta-unverified`. Instalá `@securestamp/action-proof-verify@beta-unverified`, o usá la API de
  librería, que funciona en todas las versiones.
- `@securestamp/action-proof` sigue declarando un bin llamado `action-proof-verify`, e instalarlo
  junto al verificador hace que **gane el emisor** — podés correr el CLI del emisor creyendo que
  corriste el verificador independiente. El verificador conserva el nombre y el emisor pasa a
  `action-proof`, pero ese rename sale recién cuando se publique `@securestamp/action-proof`.
- Ninguna versión publicada de ningún paquete lleva attestation de provenance de npm, incluidos
  los dos releases más recientes.
- La build publicada de `@securestamp/mcp-guard@0.2.1` expone las mismas seis tools remotas;
  Doctor y Harness son CLIs locales, mientras las tools de la capa de ejecución y el lector local
  quedan fuera del catálogo remoto publicado.
- El material para operadores de nodo **no está en este repositorio**. Acá no hay directorio
  `node/`, así que cualquier instrucción de `cd securestamp-protocol/node` no puede funcionar.

**No reclamado**

Conformidad verificada con cualquier host o cliente MCP de terceros *nombrado* a través de su
propia GUI — la compatibilidad de protocolo está verificada contra el `@modelcontextprotocol/sdk`
oficial, que es la librería que esos hosts usan, pero no afirmamos un smoke literal dentro de la
app que no corrimos · envío o listado en cualquier registro o marketplace MCP · attestations de
provenance en npm · certificación de la cadena completa grant + Guardian + aprobación humana ·
un ledger distribuido permisionado (Fabric) como sustrato multi-operador.

*No afirmamos compatibilidad que no hayamos verificado end-to-end.*

### Links

- 🌐 Fundación: [securestamp.org](https://securestamp.org)
- 🛠 Guía de inicio: [GETTING-STARTED.es.md](developers/GETTING-STARTED.es.md)
- 🤖 Agentes y harnesses: [AGENTS-AND-HARNESSES.es.md](developers/AGENTS-AND-HARNESSES.es.md)
- 📖 Protocolo actual: [SECURESTAMP-PROTOCOL-v0.2.es.md](protocol/SECURESTAMP-PROTOCOL-v0.2.es.md)
- 📄 Whitepaper actual: [SECURESTAMP-WHITEPAPER-v0.2.es.md](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md)
- 📦 npm: [`@securestamp`](https://www.npmjs.com/org/securestamp)

### Contribuir

Este repositorio es abierto. Issues sobre el protocolo, propuestas de ADR y pull requests
son bienvenidos. Ver [CONTRIBUTING.md](CONTRIBUTING.md).

### Seguridad

Reportá vulnerabilidades a **security@securestamp.org**. No abras un issue público por una
vulnerabilidad sospechada.

### Licencia

Especificación del protocolo y documentación: [CC BY 4.0](LICENSE).
Los paquetes npm publicados son Apache-2.0.
Colaboración técnica: Ivan.
