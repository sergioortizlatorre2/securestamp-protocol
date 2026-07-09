# SecureStamp Protocol

**[English](#english) · [Español](#español)**

> **Do not trust the message. Verify the action.**
> **No agent action without Proof-of-Intent.**

---

## English

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

### MCP Guard — the Agent Trust Layer (production-live)

A remote [Model Context Protocol](https://modelcontextprotocol.io) server agents consult
before acting.

- **Endpoint:** `https://mcp.securestamp.online/mcp`
- **Tools:** `authorize_action`, `analyze_message_intent`, `verify_counterparty`,
  `create_action_challenge`, `get_safe_next_step`, `issue_action_receipt`
- **Auth:** API key (`ss_live_…`) for machines · delegated session (`ss_sess_…`) for a
  human authorizing a client from their `.online` account · stdio wrapper for stdio-only
  hosts

```bash
curl -sS https://mcp.securestamp.online/mcp \
  -H "Authorization: Bearer ss_live_..." \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

### Verifiable receipts you can trust

SecureStamp **never signs a verdict declared by the caller.** A receipt can only
originate from a canonical verdict (`authorize_action` or an Action Challenge
resolution). Verify one without an account:

```bash
curl https://securestamp.online/api/action/receipts/<receiptId>
```

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
| [`protocol/SECURESTAMP-PROTOCOL-v0.2.en.md`](protocol/SECURESTAMP-PROTOCOL-v0.2.en.md) | **Current** protocol spec — Proof-of-Intent (English) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.2.es.md`](protocol/SECURESTAMP-PROTOCOL-v0.2.es.md) | Current protocol spec (Spanish) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md) | **Current** whitepaper + manifesto (English) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md) | Current whitepaper + manifesto (Spanish) |
| [`adr/ADR-005-proof-of-intent-pivot.en.md`](adr/ADR-005-proof-of-intent-pivot.en.md) | ADR — the pivot to Proof-of-Intent |
| [`adr/ADR-006-canonical-action-receipts.en.md`](adr/ADR-006-canonical-action-receipts.en.md) | ADR — canonical Action Receipts (no caller-declared verdicts) |
| [`adr/ADR-004-hyperledger-fabric-ledger.en.md`](adr/ADR-004-hyperledger-fabric-ledger.en.md) | ADR — Fabric ledger (**roadmap / non-normative**) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.1.en.md`](protocol/SECURESTAMP-PROTOCOL-v0.1.en.md) | v0.1 email trust layer (**historical**) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.1.en.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.1.en.md) | v0.1 whitepaper (**historical**) |

### What is live, and what is not

**Live:** MCP Guard production endpoint (both auth modes) · the six tools and verdict
model · canonical Action Receipts with public verification · Telegram & WhatsApp
Channel-Trust via configured channel integrations · append-only Merkle transparency logs
(Key Transparency, channel-trust).

**Roadmap / not claimed:** verified conformance with any specific third-party MCP host or
client (desktop agents, IDE copilots) — not independently smoke-tested · a published
stdio wrapper on a public registry · marketplace listings · a permissioned distributed
ledger (Fabric) as a multi-operator substrate.

*We do not assert compatibility we have not verified end-to-end.*

### Links

- 🌐 Foundation: [securestamp.org](https://securestamp.org)
- 🚀 Product: [securestamp.online](https://securestamp.online)
- 🎨 Marketplace: [securestamp.store](https://securestamp.store)
- 📖 Current protocol: [SECURESTAMP-PROTOCOL-v0.2.en.md](protocol/SECURESTAMP-PROTOCOL-v0.2.en.md)
- 📄 Current whitepaper: [SECURESTAMP-WHITEPAPER-v0.2.en.md](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md)

### Contributing

This repository is open. Protocol issues, ADR proposals, and pull requests are welcome.
See [CONTRIBUTING.md](CONTRIBUTING.md).

### License

Protocol specification and documentation: [CC BY 4.0](LICENSE).
Technical collaboration: Ivan.

---

## Español

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

### MCP Guard — el Agent Trust Layer (productivo)

Un servidor [Model Context Protocol](https://modelcontextprotocol.io) remoto que los
agentes consultan antes de actuar.

- **Endpoint:** `https://mcp.securestamp.online/mcp`
- **Tools:** `authorize_action`, `analyze_message_intent`, `verify_counterparty`,
  `create_action_challenge`, `get_safe_next_step`, `issue_action_receipt`
- **Auth:** API key (`ss_live_…`) para máquinas · sesión delegada (`ss_sess_…`) para un
  humano que autoriza un cliente desde su cuenta de `.online` · wrapper stdio para hosts
  que sólo hablan stdio

```bash
curl -sS https://mcp.securestamp.online/mcp \
  -H "Authorization: Bearer ss_live_..." \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

### Receipts verificables en los que confiar

SecureStamp **nunca firma un veredicto declarado por el llamante.** Un receipt sólo puede
originarse en un veredicto canónico (`authorize_action` o la resolución de un Action
Challenge). Verificá uno sin cuenta:

```bash
curl https://securestamp.online/api/action/receipts/<receiptId>
```

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
| [`protocol/SECURESTAMP-PROTOCOL-v0.2.es.md`](protocol/SECURESTAMP-PROTOCOL-v0.2.es.md) | Spec **actual** del protocolo — Proof-of-Intent (español) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.2.en.md`](protocol/SECURESTAMP-PROTOCOL-v0.2.en.md) | Spec actual del protocolo (inglés) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md) | Whitepaper + manifiesto **actual** (español) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.en.md) | Whitepaper + manifiesto actual (inglés) |
| [`adr/ADR-005-proof-of-intent-pivot.es.md`](adr/ADR-005-proof-of-intent-pivot.es.md) | ADR — el pivot a Proof-of-Intent |
| [`adr/ADR-006-canonical-action-receipts.es.md`](adr/ADR-006-canonical-action-receipts.es.md) | ADR — Action Receipts canónicos (sin veredictos del llamante) |
| [`adr/ADR-004-hyperledger-fabric-ledger.es.md`](adr/ADR-004-hyperledger-fabric-ledger.es.md) | ADR — ledger Fabric (**roadmap / no-normativo**) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.1.es.md`](protocol/SECURESTAMP-PROTOCOL-v0.1.es.md) | v0.1 capa de confianza de email (**histórico**) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.1.es.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.1.es.md) | Whitepaper v0.1 (**histórico**) |

### Qué está live, y qué no

**Live:** endpoint de producción de MCP Guard (ambos modos de auth) · las seis tools y el
modelo de veredicto · Action Receipts canónicos con verificación pública · Channel-Trust
de Telegram y WhatsApp mediante integraciones de canal configuradas · logs de
transparencia append-only Merkle (Key
Transparency, channel-trust).

**Roadmap / no reclamado:** conformidad verificada con cualquier host o cliente MCP de
terceros específico (agentes de escritorio, copilotos de IDE) — no probado con smoke
independiente · un wrapper stdio publicado en un registro público · listados en
marketplaces · un ledger distribuido permisionado (Fabric) como sustrato multi-operador.

*No afirmamos compatibilidad que no hayamos verificado end-to-end.*

### Links

- 🌐 Fundación: [securestamp.org](https://securestamp.org)
- 🚀 Producto: [securestamp.online](https://securestamp.online)
- 🎨 Marketplace: [securestamp.store](https://securestamp.store)
- 📖 Protocolo actual: [SECURESTAMP-PROTOCOL-v0.2.es.md](protocol/SECURESTAMP-PROTOCOL-v0.2.es.md)
- 📄 Whitepaper actual: [SECURESTAMP-WHITEPAPER-v0.2.es.md](whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md)

### Contribuir

Este repositorio es abierto. Issues sobre el protocolo, propuestas de ADR y pull requests
son bienvenidos. Ver [CONTRIBUTING.md](CONTRIBUTING.md).

### Licencia

Especificación del protocolo y documentación: [CC BY 4.0](LICENSE).
Colaboración técnica: Ivan.
