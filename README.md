# SecureStamp Protocol

**[English](#english) · [Español](#español)**

---

## English

### A trust layer for digital communications

SecureStamp is an open protocol that allows email senders to publish a verifiable cryptographic trust seal (`stamp`), and allows recipients or intermediary systems to independently verify that seal.

The protocol operates on existing DNS and SMTP infrastructure, without replacing SPF, DKIM, or DMARC — it complements them.

### Why SecureStamp?

An email that passes SPF, DKIM, and DMARC can still be a phishing attack. An attacker registers `acmec0rp.com`, configures the three protocols correctly, and sends emails that look legitimate. There is no open standard for "this domain is who it claims to be."

SecureStamp adds a fourth layer: a verifiable, cryptographic, human-readable seal.

### How it works

```
Sender domain                     Recipient / System
──────────────────                ─────────────────────────────────
_securestamp.acme.com  ──DNS──►  Resolve TXT → get stampId
X-SecureStamp: token   ──SMTP──► Verify JWT signature → get score
                                  GET /v1/trust/acme.com → check ledger
```

Three independent integration points:
1. **DNS TXT** — `_securestamp.<domain>` record
2. **Email header** — `X-SecureStamp: v=1; token=<jwt>`
3. **REST API** — `GET https://securestamp.org/v1/trust/<domain>`

Trust scores (0–100) based on SPF, DKIM, DMARC, domain age, MX reputation, and behavioral history. All events (issuance, revocation, score changes) are recorded in a permissioned Hyperledger Fabric ledger — immutable, auditable, no cryptocurrency.

### Repository contents

| Path | Description |
|---|---|
| [`protocol/`](protocol/) | SecureStamp Protocol specification |
| [`protocol/SECURESTAMP-PROTOCOL-v0.1.en.md`](protocol/SECURESTAMP-PROTOCOL-v0.1.en.md) | Protocol spec (English) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.1.es.md`](protocol/SECURESTAMP-PROTOCOL-v0.1.es.md) | Protocol spec (Spanish) |
| [`adr/`](adr/) | Architecture Decision Records |
| [`adr/ADR-004-hyperledger-fabric-ledger.en.md`](adr/ADR-004-hyperledger-fabric-ledger.en.md) | Why Hyperledger Fabric (English) |
| [`adr/ADR-004-hyperledger-fabric-ledger.es.md`](adr/ADR-004-hyperledger-fabric-ledger.es.md) | Why Hyperledger Fabric (Spanish) |
| [`whitepaper/`](whitepaper/) | Whitepaper and manifesto |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.1.en.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.1.en.md) | Whitepaper (English) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.1.es.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.1.es.md) | Whitepaper (Spanish) |

### Quick start

**Verify a domain (no account needed):**
```bash
curl https://securestamp.org/v1/trust/acmecorp.com
```

**Add a DNS record for your domain:**
```dns
_securestamp.yourdomain.com.  3600  IN  TXT  "securestamp=v=1; id=<your-stamp-id>; url=https://securestamp.org/verify/<token>"
```

**Subscribe to threat alerts:**
```
POST https://securestamp.org/v1/alerts/subscribe
```

### Links

- 🌐 Foundation: [securestamp.org](https://securestamp.org)
- 🚀 Product: [securestamp.online](https://securestamp.online)
- 🎨 Marketplace: [securestamp.store](https://securestamp.store)
- 📖 Full protocol: [SECURESTAMP-PROTOCOL-v0.1.en.md](protocol/SECURESTAMP-PROTOCOL-v0.1.en.md)
- 📄 Whitepaper: [SECURESTAMP-WHITEPAPER-v0.1.en.md](whitepaper/SECURESTAMP-WHITEPAPER-v0.1.en.md)

### Contributing

This repository is open. Protocol issues, ADR proposals, and pull requests are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

### License

Protocol specification and documentation: [CC BY 4.0](LICENSE)

---

## Español

### Una capa de confianza para las comunicaciones digitales

SecureStamp es un protocolo abierto que permite a los remitentes de email publicar un sello de confianza criptográfico y verificable (`stamp`), y permite a los receptores o sistemas intermedios verificar ese sello de forma independiente.

El protocolo opera sobre la infraestructura existente de DNS y SMTP, sin reemplazar SPF, DKIM ni DMARC — los complementa.

### ¿Por qué SecureStamp?

Un email que pasa SPF, DKIM y DMARC puede seguir siendo un ataque de phishing. Un atacante registra `acmec0rp.com`, configura los tres protocolos correctamente y envía emails que parecen legítimos. No existe un estándar abierto para "este dominio es quien dice ser".

SecureStamp agrega una cuarta capa: un sello verificable, criptográfico y legible por personas.

### Cómo funciona

```
Dominio del remitente              Receptor / Sistema
──────────────────                 ────────────────────────────────
_securestamp.acme.com  ──DNS──►  Resolver TXT → obtener stampId
X-SecureStamp: token   ──SMTP──► Verificar firma JWT → obtener score
                                   GET /v1/trust/acme.com → check ledger
```

Tres puntos de integración independientes:
1. **DNS TXT** — registro `_securestamp.<dominio>`
2. **Header de email** — `X-SecureStamp: v=1; token=<jwt>`
3. **REST API** — `GET https://securestamp.org/v1/trust/<dominio>`

Scores de confianza (0–100) basados en SPF, DKIM, DMARC, antigüedad del dominio, reputación del MX e historial de comportamiento. Todos los eventos (emisión, revocación, cambios de score) se registran en un ledger Hyperledger Fabric permisionado — inmutable, auditable, sin criptomoneda.

### Contenido del repositorio

| Ruta | Descripción |
|---|---|
| [`protocol/`](protocol/) | Especificación del protocolo SecureStamp |
| [`protocol/SECURESTAMP-PROTOCOL-v0.1.en.md`](protocol/SECURESTAMP-PROTOCOL-v0.1.en.md) | Spec del protocolo (inglés) |
| [`protocol/SECURESTAMP-PROTOCOL-v0.1.es.md`](protocol/SECURESTAMP-PROTOCOL-v0.1.es.md) | Spec del protocolo (español) |
| [`adr/`](adr/) | Architecture Decision Records |
| [`adr/ADR-004-hyperledger-fabric-ledger.en.md`](adr/ADR-004-hyperledger-fabric-ledger.en.md) | Por qué Hyperledger Fabric (inglés) |
| [`adr/ADR-004-hyperledger-fabric-ledger.es.md`](adr/ADR-004-hyperledger-fabric-ledger.es.md) | Por qué Hyperledger Fabric (español) |
| [`whitepaper/`](whitepaper/) | Whitepaper y manifiesto |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.1.en.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.1.en.md) | Whitepaper (inglés) |
| [`whitepaper/SECURESTAMP-WHITEPAPER-v0.1.es.md`](whitepaper/SECURESTAMP-WHITEPAPER-v0.1.es.md) | Whitepaper (español) |

### Quick start

**Verificar un dominio (sin cuenta):**
```bash
curl https://securestamp.org/v1/trust/acmecorp.com
```

**Agregar un registro DNS para tu dominio:**
```dns
_securestamp.tudominio.com.  3600  IN  TXT  "securestamp=v=1; id=<tu-stamp-id>; url=https://securestamp.org/verify/<token>"
```

**Suscribirse a alertas de amenazas:**
```
POST https://securestamp.org/v1/alerts/subscribe
```

### Links

- 🌐 Fundación: [securestamp.org](https://securestamp.org)
- 🚀 Producto: [securestamp.online](https://securestamp.online)
- 🎨 Marketplace: [securestamp.store](https://securestamp.store)
- 📖 Protocolo completo: [SECURESTAMP-PROTOCOL-v0.1.es.md](protocol/SECURESTAMP-PROTOCOL-v0.1.es.md)
- 📄 Whitepaper: [SECURESTAMP-WHITEPAPER-v0.1.es.md](whitepaper/SECURESTAMP-WHITEPAPER-v0.1.es.md)

### Contribuir

Este repositorio es abierto. Issues sobre el protocolo, propuestas de ADR y pull requests son bienvenidos.

### Licencia

Especificación del protocolo y documentación: [CC BY 4.0](LICENSE)
