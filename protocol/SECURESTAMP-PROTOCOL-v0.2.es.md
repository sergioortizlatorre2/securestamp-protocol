# Protocolo SecureStamp v0.2 — Proof-of-Intent

**Estado:** Borrador
**Fecha:** 2026-07-09
**Mantenido por:** SecureStamp Foundation
**Reemplaza a:** [Protocolo SecureStamp v0.1](SECURESTAMP-PROTOCOL-v0.1.es.md) *(capa de confianza de email — se conserva como histórico)*
**Repositorio:** https://github.com/sergioortizlatorre2/securestamp-protocol

---

## Resumen

SecureStamp v0.1 definía una **capa de confianza para email**: un sello verificable que
publica un remitente y comprueba un receptor. v0.2 generaliza esa idea a un **protocolo
de Proof-of-Intent para la era de la IA**.

La doctrina cabe en una frase:

> **No se confía en el mensaje. Se verifica la acción.**

Antes de que una persona, una organización o un agente de IA ejecute una operación
digital sensible —un pago, un cambio de instrucciones bancarias o de payout, una
aprobación, un cambio de cuenta irreversible— SecureStamp verifica cinco dimensiones de
esa operación:

```
intención · contraparte · canal · política · acción
```

y devuelve un **veredicto**. Cuando la operación está respaldada por un veredicto
canónico, SecureStamp puede emitir un **Action Receipt verificable** que un tercero
puede comprobar de forma independiente.

El claim central del protocolo es:

> **No agent action without Proof-of-Intent.**
> (Ninguna acción de agente sin Proof-of-Intent.)

Este documento especifica el modelo de verificación, los tres pilares de producto que lo
exponen, la interfaz remota **MCP Guard** para agentes de IA, la regla del **Action
Receipt canónico**, los adaptadores de canal y el modelo de transparencia/registros.
También declara con claridad qué está productivo hoy y qué es roadmap.

---

## 1. Relación con v0.1

v0.2 **no** descarta a v0.1. El sello de confianza de email (`stamp`), el registro DNS
TXT `_securestamp.<dominio>`, el header `X-SecureStamp` y el score de confianza del
dominio siguen siendo válidos y están implementados. Bajo v0.2 se reencuadran:

- Las señales de confianza de email/dominio de v0.1 son **entradas** de las dimensiones
  `contraparte` y `canal` de abajo — no el protocolo entero.
- v0.2 es **aditivo**: un verificador v0.1 sigue funcionando; un verificador v0.2
  además entiende acciones, veredictos, receipts y la interfaz MCP.

Leé v0.1 cuando te importa *"¿este remitente es quien dice ser?"*. Leé v0.2 cuando te
importa *"¿debería permitirse que esta acción ocurra?"*.

---

## 2. Terminología

| Término | Definición |
|---|---|
| **Acción** | Operación digital sensible a punto de ejecutarse (pago, cambio de payout, aprobación, cambio de instrucciones, cambio de cuenta irreversible). |
| **Intención** | El propósito inferido de un mensaje o pedido que llevaría a una acción. |
| **Contraparte** | La otra parte de la acción (un proveedor, una cuenta bancaria, un contacto, una organización). |
| **Canal** | El medio por el que llegó el pedido (email, web, Telegram, WhatsApp, un host MCP). |
| **Política** | Las reglas del tenant que restringen qué se permite, cuándo y por quién. |
| **Veredicto** | El resultado de la verificación: `allow`, `needs_confirmation` o `block`. |
| **Action Challenge** | Paso de doble control single-use, token-gated, para confirmar o rechazar una acción riesgosa fuera de banda. |
| **Action Receipt** | Registro firmado y verificable públicamente que refiere a un veredicto o resolución de challenge **canónico** computado por SecureStamp. |
| **MCP Guard** | El servidor Model Context Protocol remoto que expone la verificación a agentes y hosts de IA. |
| **Motor de veredictos** | El componente server-side que computa un veredicto a partir de datos verificados por el tenant. Es el **único** productor de veredictos autoritativos. |

---

## 3. Doctrina y claim central

Dos invariantes gobiernan todo el protocolo:

1. **La verificación es sobre acciones, no mensajes.** Un mensaje que "parece legítimo"
   no prueba nada. SecureStamp verifica *la acción que el mensaje causaría*.
2. **SecureStamp nunca firma un veredicto declarado por el llamante.** Un veredicto
   autoritativo sólo puede producirlo el motor de veredictos de SecureStamp sobre datos
   verificados por el tenant, o la resolución de un Action Challenge real. Esto es lo
   que hace significativo a un Action Receipt (ver §7).

De esto se desprende el claim impreso en toda superficie: **No agent action without
Proof-of-Intent.**

---

## 4. Las cinco dimensiones de verificación

Un pedido de verificación se evalúa en cinco dimensiones. Cada una aporta al veredicto;
una falla en una dimensión de alto riesgo puede forzar `needs_confirmation` o `block`
sin importar las demás.

| Dimensión | Pregunta que responde | Señales de ejemplo |
|---|---|---|
| **Intención** | ¿Qué intenta hacer realmente este pedido? | clasificación de intención del mensaje, categoría de riesgo, señales de urgencia/presión |
| **Contraparte** | ¿La otra parte es conocida y consistente? | grafo de contrapartes del tenant, interacciones previas, confianza de dominio v0.1, match de identidad |
| **Canal** | ¿Llegó por un medio esperado y confiable? | origen email/web/Telegram/WhatsApp/MCP, reglas de channel-trust, brand-claim boundary |
| **Política** | ¿La política del tenant lo permite ahora? | umbrales de aprobación, reglas de doble control, allowlists, límites de plan |
| **Acción** | ¿La operación concreta en sí es segura? | fingerprint de instrucciones, detección de cambio de payout/cuenta, irreversibilidad |

La verificación es **consultiva y no ejecutora**: SecureStamp autoriza; no ejecuta la
acción.

---

## 5. Los tres pilares

El modelo de verificación se expone a través de tres productos. Comparten el mismo motor
de veredictos y el mismo modelo de receipts.

### 5.1 VendorShield — contraparte y operaciones sensibles
Verificación de contrapartes y operaciones de alto riesgo: pagos, proveedores,
invoices, cambios de cuenta bancaria y de payout, cambios de instrucciones y
aprobaciones. Responde *"¿es seguro actuar sobre esta contraparte y este movimiento de
dinero?"*.

### 5.2 Guardian — protección humana y de canales
Protección de personas y canales: Brand Claim Requests, alertas, educación del usuario y
confianza a nivel de canal en email, web, Telegram y WhatsApp. Responde *"¿es seguro que
un humano confíe y actúe sobre esto que llegó?"*.

### 5.3 MCP Guard — Agent Trust Layer
Interfaz MCP remota que agentes de IA, copilotos y workflows consultan **antes** de
ejecutar una acción sensible. Responde *"como agente autónomo, ¿tengo permitido hacer
esto?"*. Se especifica en §6.

---

## 6. MCP Guard (Agent Trust Layer)

MCP Guard es un servidor [Model Context Protocol](https://modelcontextprotocol.io)
remoto y always-on. Un agente a punto de actuar lo consulta primero; el Guard devuelve un
veredicto y, opcionalmente, un receipt. El Guard **autoriza; no ejecuta** — nunca mueve
dinero, borra datos ni realiza operaciones destructivas.

### 6.1 Endpoint

| Entorno | Endpoint | Estado |
|---|---|---|
| Producción | `https://mcp.securestamp.online/mcp` | **Live** (HTTPS) |
| Staging | `https://mcp-staging.securestamp.online/mcp` | Live (testing de integración) |

El manifest se sirve en `/.well-known/securestamp-mcp.json`.

### 6.2 Tools

| Tool | Propósito |
|---|---|
| `authorize_action` | Computa un veredicto (`allow` / `needs_confirmation` / `block`) para una acción. |
| `analyze_message_intent` | Clasifica la intención y el riesgo de un mensaje. |
| `verify_counterparty` | Comprueba una contraparte contra el grafo conocido del tenant. |
| `create_action_challenge` | Crea un challenge de doble control para una acción riesgosa. |
| `get_safe_next_step` | Recomienda el siguiente paso seguro dado el contexto. |
| `issue_action_receipt` | Re-afirma un receipt ya computado por `receiptId` — nunca firma un veredicto declarado por el llamante (ver §7). |

### 6.3 Veredictos

`authorize_action` devuelve uno de:

- `allow` — la acción es consistente con contraparte, canal y política.
- `needs_confirmation` — la acción requiere un Action Challenge fuera de banda antes de
  continuar.
- `block` — la acción no debe continuar.

### 6.4 Autenticación

Un único header `Authorization: Bearer <token>` lleva la credencial; el server enruta
por prefijo.

1. **API key — `ss_live_…`** — servidor-a-servidor (agentes, backends, máquinas). Se
   almacena sólo como hash SHA-256; reveal una sola vez; se crea desde el dashboard de
   `.online`.
2. **Sesión delegada — `ss_sess_…`** — un usuario logueado en `.online` autoriza un
   cliente MCP mediante un pairing code de corta vida, que el cliente intercambia por un
   token de sesión delegada. Sólo hash, revocable, tenant-scoped.
3. **Wrapper stdio** — para hosts que sólo hablan stdio, un wrapper local
   (`securestamp-mcp-guard`) expone las mismas tools sobre el proceso local con una key
   `ss_live_`.

Ambos modos bearer resuelven a la misma identidad interna (`userId` / `orgId` /
`allowedTools` / rate limit); el pipeline downstream es idéntico.

### 6.5 Non-goals (V1)

- **No** ejecuta la acción, mueve dinero ni realiza operaciones destructivas.
- **No** actúa sobre el contenido del mensaje — verifica la acción.

---

## 7. Action Receipts canónicos

Un **Action Receipt** es un registro firmado (ES256) y verificable públicamente de una
acción verificada. La verificación pública está disponible sin autenticación en:

```
GET https://securestamp.online/api/action/receipts/<receiptId>
```

La regla que define el protocolo:

> Un receipt verificable sólo puede originarse en un veredicto canónico que SecureStamp
> mismo computó — nunca en un veredicto declarado por el llamante.

### 7.1 Las dos —y sólo dos— fuentes canónicas

1. **`authorize_action`** computa un veredicto sobre datos verificados por el tenant
   (registro de contrapartes, política, fingerprint de instrucciones), lo firma, lo
   persiste y devuelve un `receiptId`.
2. **Resolución de Action Challenge** — cuando un challenge se confirma o rechaza vía su
   endpoint single-use, token-gated, el veredicto resultante
   (`challenge_confirmed` / `challenge_rejected`) se computa a partir de una transición
   de estado real, se firma y se persiste.

Ambos emiten su receipt internamente, con datos que ellos mismos computaron.

### 7.2 `issue_action_receipt` es lookup, no emisión

`issue_action_receipt` acepta un `receiptId` (y, opcionalmente, un `detectedIntent` que
debe coincidir) y **re-afirma** un receipt canónico existente. Este tool:

- busca el receipt por su `receiptId` generado por el servidor;
- aplica tenant scoping (un receipt de otro tenant es indistinguible de "no existe");
- devuelve el token **ya firmado** — nunca vuelve a firmar nada;
- devuelve un único error unificado para no-existe / otro-tenant / fingerprint no
  coincide, para que no pueda usarse como oráculo de existencia.

**No** acepta un `verdict`, `reasons` ni `safeNextStep` declarados por el llamante. Esto
es lo que permite a un tercero confiar en un receipt: siempre remonta a un veredicto que
el motor de SecureStamp produjo.

---

## 8. Adaptadores de canal

La verificación y la protección Guardian se entregan por múltiples canales. Un adaptador
de canal mapea un pedido entrante de ese medio al modelo de cinco dimensiones, y mapea
los veredictos/alertas de vuelta.

| Canal | Estado |
|---|---|
| **Email / Web** | Live (sello v0.1 + verificación v0.2 y páginas públicas de verify). |
| **Telegram** | Vivo mediante integraciones de canal configuradas — conector Channel-Trust. |
| **WhatsApp** | Vivo mediante integraciones de canal configuradas — conector Channel-Trust. |
| **Hosts MCP** | **Live** — endpoint MCP Guard remoto (§6). |

Telegram y WhatsApp están soportados mediante integraciones de canal configuradas para
flujos de Proof-of-Intent. La disponibilidad depende del conector de canal desplegado por
SecureStamp y de las políticas de cada plataforma de mensajería. Responden preguntas de
channel-trust e intención para humanos en esas apps, y SecureStamp no reclama estatus de
partner oficial ni nativo con ninguna de las dos plataformas. La conformidad de un host o
cliente MCP *de terceros* específico es un asunto aparte — ver §12.

---

## 9. Transparencia y registros (el "ledger")

El historial relevante para seguridad en SecureStamp se guarda en **logs de
transparencia append-only tipo Merkle** que soportan **pruebas de inclusión** (esta
entrada está en el log) y **pruebas de consistencia** (el log sólo se agregó, nunca se
reescribió). Este es el mecanismo que efectivamente está en producción hoy. Respalda,
por ejemplo:

- **Key Transparency** para claves de identidad end-to-end-encrypted (publicar/revocar
  se agregan; los clientes pueden probar inclusión y monitorear su propio historial de
  claves).
- **Transparencia de abuso / consultas** para eventos de channel-trust.

> **Roadmap, no-normativo:** un ledger distribuido permisionado (Hyperledger Fabric) se
> propuso en [ADR-004](../adr/ADR-004-hyperledger-fabric-ledger.es.md) como sustrato
> multi-operador futuro. **No** es la implementación actual y no se reclama conformidad
> con él. El sustrato normativo de v0.2 es el log de transparencia append-only Merkle de
> arriba. Ver el banner de roadmap de ADR-004.

---

## 10. Consideraciones de seguridad

- **Fail-closed en todo.** Credenciales faltantes/expiradas/revocadas, scopes faltantes
  o veredictos ausentes resultan en denegación, nunca en un default permisivo.
- **Sin veredictos declarados por el llamante.** Ver §3 y §7 — la invariante que
  sostiene todo.
- **Credenciales sólo-hash.** API keys y tokens de sesión delegada se almacenan como
  hashes SHA-256; reveal una sola vez; revocación inmediata.
- **Tenant scoping.** Todo lookup está scopeado al tenant que llama; nunca se divulga
  existencia cross-tenant.
- **Rate limiting** por tool y por tenant/sesión.
- **Auditoría sin payload sensible.** Los eventos de auditoría registran que una acción
  se verificó, no el contenido sensible de la acción; IP/UA hasheados.
- **Least-privilege** en la identidad operativa y aislamiento de entornos
  (producción/staging).

---

## 11. Versionado y compatibilidad

- El protocolo usa versionado semántico con prefijo `v`. v0.2 es **aditivo** sobre v0.1.
- Los puntos de integración conservan sus marcadores de versión (DNS TXT `v=1`,
  `X-SecureStamp` `v=1`, REST `/v1/`); la interfaz MCP anuncia su propia versión de
  protocolo en el manifest.
- Un cambio es **mayor** si rompe un contrato de receipt/veredicto, un formato de
  credencial o un punto de integración existente.

---

## 12. Productivo vs roadmap (dicho sin vueltas)

**Live hoy:**

- Endpoint de producción de MCP Guard (`https://mcp.securestamp.online/mcp`), ambos
  modos de auth.
- Las seis tools MCP y el modelo de veredicto `allow` / `needs_confirmation` / `block`.
- Action Receipts canónicos con verificación pública.
- Channel-Trust de Telegram y WhatsApp, vivo mediante integraciones de canal configuradas.
- Logs de transparencia append-only Merkle (Key Transparency, transparencia de
  channel-trust).

**Roadmap / aún no reclamado:**

- Conformidad con cualquier host o cliente MCP de terceros específico (p. ej. agentes de
  escritorio, copilotos de IDE) — **no** verificado con smoke independiente; no se
  afirma.
- Un paquete wrapper stdio publicado en un registro público.
- Listados en marketplaces.
- Un ledger distribuido permisionado (Hyperledger Fabric) como sustrato multi-operador
  (ADR-004) — sólo propuesta.

No se reclama ninguna compatibilidad que no haya sido verificada end-to-end.

---

*Fin del documento — Protocolo SecureStamp v0.2 (Proof-of-Intent).*
*Colaboración técnica: Ivan.*
