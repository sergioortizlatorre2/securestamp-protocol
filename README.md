# SecureStamp Protocol

**[English](#english) · [Español](#español)**

> **Do not trust the message. Verify the action.**
> **No agent action without Proof-of-Intent.**

<sub>Endpoints, tool catalog, npm dist-tags and on-device engine versions on this page were
checked against the live services and the public npm registry on **2026-09-06**. Where a claim
is not verified, it is listed under [What is live, and what is not](#what-is-live-and-what-is-not).</sub>

---

## English

### Start here

| If you want to… | Go to |
| --- | --- |
| Understand the idea in five minutes | this page |
| **Connect an agent, copilot or MCP host** | **[Getting Started](developers/GETTING-STARTED.en.md)** |
| Install the packages | [npm packages](#npm-packages) — *read the dist-tag note first* |
| Verify a receipt or a proof bundle offline | [Getting Started §5](developers/GETTING-STARTED.en.md#5-verify-without-an-account) |
| Read the normative protocol spec | [Protocol v0.2](protocol/SECURESTAMP-PROTOCOL-v0.2.en.md) |
| Understand the execution layer | [Action Proof and the Execution Guardian](#action-proof-and-the-execution-guardian) |

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

`read_message_request` exists only in the local stdio wrapper and is deliberately never
exposed remotely.

```bash
curl -sS https://mcp.securestamp.online/mcp \
  -H "Authorization: Bearer ss_live_..." \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Full connection recipes for all three modes: **[Getting Started](developers/GETTING-STARTED.en.md)**.

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

Six packages are public on npm under **Apache-2.0**.

> **Read this before `npm install`.** The `0.3.0-beta.2` line is published under the
> **`beta-unverified`** dist-tag, not `latest`. That tag is deliberate: it means the code
> is installable and discoverable, but has **not** cleared the external evidence gate. A
> plain `npm install` therefore gives you an **older** version for most packages. Ask for
> the tag explicitly if you want the 0.3 line.

| Package | What it is | `latest` | `beta-unverified` |
| --- | --- | --- | --- |
| [`@securestamp/action-proof-verify`](https://www.npmjs.com/package/@securestamp/action-proof-verify) | Offline, dependency-free verifier for receipts and proof bundles. No network. **Start here.** | `0.3.0-beta.1` | `0.3.0-beta.2` |
| [`@securestamp/action-registry`](https://www.npmjs.com/package/@securestamp/action-registry) | Dependency-free declarative registry of operations. Consumers derive catalogs from it. | `0.3.0-beta.1` | `0.3.0-beta.2` |
| [`@securestamp/action-proof`](https://www.npmjs.com/package/@securestamp/action-proof) | Protocol primitives: source envelopes, canonical effects, grants, bundles, signing. | `0.2.0-beta.1` | `0.3.0-beta.2` |
| [`@securestamp/execution-guardian`](https://www.npmjs.com/package/@securestamp/execution-guardian) | The customer-controlled execution daemon. Holds *your* provider credentials. | `0.2.0-beta.2` | `0.3.0-beta.2` |
| [`@securestamp/execution-guardian-mcp`](https://www.npmjs.com/package/@securestamp/execution-guardian-mcp) | Credential-free stdio MCP bridge to your Guardian, over a Unix socket only. | `0.2.0-beta.1` | `0.3.0-beta.2` |
| [`@securestamp/mcp-guard`](https://www.npmjs.com/package/@securestamp/mcp-guard) | Local stdio wrapper for hosts that only speak stdio. | `0.1.0` | — |

The verifier and the registry are published **ahead of** what they verify and what consumes
them, so a released verifier accepts a new bundle version before anything emits one. That is
why their `latest` is further along than the emitter's.

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
| **Knowledge-pack version** (`ssfmlRulesVersion`) | the active signed pack | served from the `stable` channel | auto-updates in the background |

The pack channel is **data-only and signed**: a pack manifest carries an ES256 signature over
a SHA-256 of the artifact, the public key is pinned in the plugin, and a stable build rejects
an unsigned or invalid pack and falls back to the bundled data. A pack may only add literal
phrases to signal IDs the bundled engine already knows, within bounded weight limits — it can
never introduce a rule, an operator, or a category, and **it can never lower a verdict below
what the bundled engine would have produced.**

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

### What is live, and what is not

**Live and externally checkable (2026-09-06):** the MCP Guard production endpoint, its
nine-tool catalog and `2025-11-25` protocol version · all three auth modes, with OAuth
protected-resource metadata served · canonical Action Receipts with public verification ·
six Apache-2.0 packages on the public npm registry · the SSFML signed knowledge-pack
channel on `stable` · Telegram and WhatsApp Channel-Trust via configured channel
integrations · append-only Merkle transparency logs (Key Transparency, channel-trust).

**Published but explicitly unverified:** the `0.3.0-beta.2` line sits under the
`beta-unverified` dist-tag and a `server.json` that says so. Discoverable is not certified;
the verified beta still requires the external evidence gate.

**Roadmap / not claimed:** verified conformance with any *named* third-party MCP host or
client through its own GUI — protocol compatibility is verified against the official
`@modelcontextprotocol/sdk`, which is the library those hosts use, but we do not claim a
literal in-app smoke we have not run · submission to, or listing in, any MCP registry or
marketplace · certification of the full grant + Guardian + human-approval chain · a
permissioned distributed ledger (Fabric) as a multi-operator substrate.

*We do not assert compatibility we have not verified end-to-end.*

### Links

- 🌐 Foundation: [securestamp.org](https://securestamp.org)
- 🚀 Product: [securestamp.online](https://securestamp.online)
- 🎨 Marketplace: [securestamp.store](https://securestamp.store)
- 🛠 Developer quickstart: [GETTING-STARTED.en.md](developers/GETTING-STARTED.en.md)
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
| **Conectar un agente, copiloto o host MCP** | **[Guía de inicio](developers/GETTING-STARTED.es.md)** |
| Instalar los paquetes | [Paquetes npm](#paquetes-npm) — *leé primero la nota de dist-tags* |
| Verificar un receipt o un proof bundle offline | [Guía de inicio §5](developers/GETTING-STARTED.es.md#5-verificar-sin-cuenta) |
| Leer la spec normativa | [Protocolo v0.2](protocol/SECURESTAMP-PROTOCOL-v0.2.es.md) |
| Entender la capa de ejecución | [Action Proof y el Execution Guardian](#action-proof-y-el-execution-guardian) |

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

`read_message_request` existe sólo en el wrapper stdio local y deliberadamente nunca se
expone de forma remota.

```bash
curl -sS https://mcp.securestamp.online/mcp \
  -H "Authorization: Bearer ss_live_..." \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Recetas completas de conexión para los tres modos: **[Guía de inicio](developers/GETTING-STARTED.es.md)**.

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

Seis paquetes públicos en npm bajo **Apache-2.0**.

> **Leé esto antes de `npm install`.** La línea `0.3.0-beta.2` está publicada bajo el
> dist-tag **`beta-unverified`**, no bajo `latest`. Ese tag es deliberado: significa que el
> código es instalable y descubrible, pero **no** pasó el gate de evidencia externa. Un
> `npm install` pelado te da entonces una versión **más vieja** en la mayoría de los
> paquetes. Pedí el tag explícitamente si querés la línea 0.3.

| Paquete | Qué es | `latest` | `beta-unverified` |
| --- | --- | --- | --- |
| [`@securestamp/action-proof-verify`](https://www.npmjs.com/package/@securestamp/action-proof-verify) | Verificador offline y sin dependencias de receipts y proof bundles. Sin red. **Empezá acá.** | `0.3.0-beta.1` | `0.3.0-beta.2` |
| [`@securestamp/action-registry`](https://www.npmjs.com/package/@securestamp/action-registry) | Registro declarativo de operaciones, sin dependencias. Los consumidores derivan su catálogo de acá. | `0.3.0-beta.1` | `0.3.0-beta.2` |
| [`@securestamp/action-proof`](https://www.npmjs.com/package/@securestamp/action-proof) | Primitivas del protocolo: source envelopes, efectos canónicos, grants, bundles, firma. | `0.2.0-beta.1` | `0.3.0-beta.2` |
| [`@securestamp/execution-guardian`](https://www.npmjs.com/package/@securestamp/execution-guardian) | El daemon de ejecución controlado por el cliente. Tiene *tus* credenciales de proveedor. | `0.2.0-beta.2` | `0.3.0-beta.2` |
| [`@securestamp/execution-guardian-mcp`](https://www.npmjs.com/package/@securestamp/execution-guardian-mcp) | Bridge MCP stdio credential-free hacia tu Guardian, sólo por Unix socket. | `0.2.0-beta.1` | `0.3.0-beta.2` |
| [`@securestamp/mcp-guard`](https://www.npmjs.com/package/@securestamp/mcp-guard) | Wrapper stdio local para hosts que sólo hablan stdio. | `0.1.0` | — |

El verificador y el registry se publican **antes** de lo que verifican y de lo que los
consume, así que un verificador ya liberado acepta una versión nueva de bundle antes de que
alguien la emita. Por eso su `latest` va más adelante que el del emisor.

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
| **Versión del knowledge-pack** (`ssfmlRulesVersion`) | el pack activo | servida desde el canal `stable` | auto-actualiza en background |

El canal de packs es **sólo datos y firmado**: el manifiesto lleva una firma ES256 sobre un
SHA-256 del artefacto, la clave pública está pinneada en el plugin, y un build estable rechaza
un pack sin firma o inválido y cae al dato empaquetado. Un pack sólo puede agregar frases
literales a signal IDs que el motor empaquetado ya conoce, dentro de límites acotados de peso
— nunca puede introducir una regla, un operador ni una categoría, y **nunca puede bajar un
veredicto por debajo de lo que habría producido el motor empaquetado.**

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

### Qué está live, y qué no

**Live y comprobable desde afuera (2026-09-06):** el endpoint de producción de MCP Guard, su
catálogo de nueve tools y la versión de protocolo `2025-11-25` · los tres modos de auth, con
metadata OAuth de protected-resource servida · Action Receipts canónicos con verificación
pública · seis paquetes Apache-2.0 en el registro público de npm · el canal firmado de
knowledge-packs de SSFML en `stable` · Channel-Trust de Telegram y WhatsApp mediante
integraciones de canal configuradas · logs de transparencia append-only Merkle (Key
Transparency, channel-trust).

**Publicado pero explícitamente no verificado:** la línea `0.3.0-beta.2` está bajo el dist-tag
`beta-unverified` y un `server.json` que lo declara. Descubrible no es certificado; la beta
verificada sigue exigiendo el gate de evidencia externa.

**Roadmap / no reclamado:** conformidad verificada con cualquier host o cliente MCP de
terceros *nombrado* a través de su propia GUI — la compatibilidad de protocolo está verificada
contra el `@modelcontextprotocol/sdk` oficial, que es la librería que esos hosts usan, pero no
afirmamos un smoke literal dentro de la app que no corrimos · envío o listado en cualquier
registro o marketplace MCP · certificación de la cadena completa grant + Guardian + aprobación
humana · un ledger distribuido permisionado (Fabric) como sustrato multi-operador.

*No afirmamos compatibilidad que no hayamos verificado end-to-end.*

### Links

- 🌐 Fundación: [securestamp.org](https://securestamp.org)
- 🚀 Producto: [securestamp.online](https://securestamp.online)
- 🎨 Marketplace: [securestamp.store](https://securestamp.store)
- 🛠 Guía de inicio: [GETTING-STARTED.es.md](developers/GETTING-STARTED.es.md)
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
