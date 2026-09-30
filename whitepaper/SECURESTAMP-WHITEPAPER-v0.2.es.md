# SecureStamp: Proof-of-Intent para la era de la IA

**Versión:** 0.2 — Borrador
**Fecha:** Julio 2026
**Autores:** SecureStamp Foundation
**Reemplaza a:** [Whitepaper v0.1](SECURESTAMP-WHITEPAPER-v0.1.es.md) *(capa de confianza de email — se conserva como histórico)*
**Licencia:** CC BY 4.0

---

## Manifiesto

Durante cuarenta años intentamos hacer confiable el **mensaje**. Autenticamos
remitentes, firmamos headers, puntuamos dominios. Ayudó — y nunca alcanzó, porque lo que
te daña no es el mensaje. Es la **acción** a la que el mensaje te convence: la
transferencia, los datos bancarios cambiados, la aprobación que clickeaste, la
instrucción que tu agente de IA ejecutó.

La llegada de agentes autónomos vuelve esto ineludible. Un agente lee un mensaje y
*actúa*. No hay pausa humana entre "esto parece legítimo" y "el dinero ya no está".

Así que SecureStamp cambió su pregunta.

> **No se confía en el mensaje. Se verifica la acción.**
> **No agent action without Proof-of-Intent.**

---

## 1. El problema se movió

SecureStamp empezó (v0.1) como una capa de confianza para email: un sello verificable
que respondía *"¿este remitente es quien dice ser?"*. Ese problema es real y el sello
sigue en producción. Pero el centro de gravedad del daño digital se movió de la
**suplantación** a la **acción inducida** — y, cada vez más, a la **acción inducida
ejecutada por software sin humano en el loop.**

Considerá la falla moderna:

1. Un mensaje (email, chat, una tool call a un agente de IA) pide una operación
   sensible: *"actualizá la cuenta de payout"*, *"aprobá esta invoice"*, *"enviá el
   pago"*.
2. Todo en el mensaje parece bien. Puede incluso pasar SPF, DKIM y DMARC.
3. Un humano —o un agente actuando en su nombre— ejecuta.
4. La contraparte era incorrecta, la instrucción fue manipulada, el canal era hostil o
   la política fue violada. La acción suele ser irreversible.

Ninguna cantidad de autenticación del mensaje atrapa esto, porque el mensaje era,
técnicamente, auténtico. Lo que nunca se verificó fue **la acción**.

---

## 2. La idea: Proof-of-Intent

SecureStamp verifica una acción en cinco dimensiones antes de que ocurra:

```
intención · contraparte · canal · política · acción
```

- **Intención** — ¿qué intenta hacer realmente este pedido?
- **Contraparte** — ¿la otra parte es conocida y consistente?
- **Canal** — ¿llegó por un medio esperado y confiable?
- **Política** — ¿la política propia del tenant lo permite, ahora, por este actor?
- **Acción** — ¿la operación concreta en sí es segura (y reversible)?

El resultado es un **veredicto**: `allow`, `needs_confirmation` o `block`. Cuando el
veredicto es autoritativo, SecureStamp puede emitir un **Action Receipt verificable** —
evidencia, comprobable por un tercero, de que la acción fue verificada.

SecureStamp **autoriza; no ejecuta.** Es el checkpoint, no el actor. Nunca mueve dinero
ni realiza la operación.

---

## 3. Tres pilares

El mismo motor de verificación se entrega a través de tres productos, para tres
audiencias.

### VendorShield — para finanzas y operaciones
Verificación de contrapartes y movimiento de dinero sensible: proveedores, invoices,
pagos, cambios de cuenta bancaria y de payout, aprobaciones, cambios de instrucciones.
*"¿Es seguro actuar sobre esta contraparte y esta transferencia?"*

### Guardian — para personas y canales
Protección de humanos a través de canales —email, web, Telegram, WhatsApp— con Brand
Claim Requests, alertas y educación. *"¿Es seguro que una persona confíe y actúe sobre
esto que llegó?"*

### MCP Guard — para agentes de IA
Un servidor [Model Context Protocol](https://modelcontextprotocol.io) remoto que
agentes, asistentes y workflows consultan **antes** de actuar. *"Como agente autónomo,
¿tengo permitido hacer esto?"* Este es el pilar que la era de la IA exige, y está
productivo.

---

## 4. MCP Guard: el Agent Trust Layer

Un agente de IA a punto de tomar una acción sensible llama primero a MCP Guard. El Guard
verifica las cinco dimensiones y devuelve un veredicto — y, cuando corresponde, un
receipt.

- **Endpoint (producción, live):** `https://mcp.securestamp.online/mcp`
- **Seis tools:** `authorize_action`, `analyze_message_intent`, `verify_counterparty`,
  `create_action_challenge`, `get_safe_next_step`, `issue_action_receipt`.
- **Dos modos de auth:** una API key (`ss_live_…`) para máquinas y backends, y una
  sesión delegada (`ss_sess_…`) para un humano que autoriza un cliente desde su cuenta
  de `.online`. Un wrapper stdio local cubre hosts que sólo hablan stdio.
- **Autoriza; no ejecuta.** No pasa dinero por el Guard.

La spec del protocolo detalla la interfaz:
[SECURESTAMP-PROTOCOL-v0.2.es.md](../protocol/SECURESTAMP-PROTOCOL-v0.2.es.md).

---

## 5. Receipts en los que realmente se puede confiar

Un receipt vale exactamente lo que vale la regla detrás de él. La regla de SecureStamp:

> **SecureStamp nunca firma un veredicto declarado por el llamante.**

Un Action Receipt verificable puede originarse en exactamente dos lugares: un veredicto
computado por `authorize_action` sobre datos verificados por el tenant, o la resolución
de un Action Challenge real y single-use. El tool `issue_action_receipt` no *acuña*
receipts a partir de claims provistos por el llamante — **busca y re-afirma** un receipt
que SecureStamp ya computó y persistió, y devuelve el token que ya estaba firmado.

Cualquiera puede verificar un receipt sin cuenta:

```
GET https://securestamp.online/api/action/receipts/<receiptId>
```

Como los únicos escritores de receipts son los flujos de veredicto canónicos, un
"SecureStamp Action Receipt" siempre remonta a una decisión que SecureStamp realmente
tomó — no a un veredicto que alguien declaró sobre sí mismo.

---

## 6. Canales

Guardian y la verificación llegan a las personas donde están:

- **Email / Web** — el sello original más páginas públicas de verificación.
- **Telegram** y **WhatsApp** — Channel-Trust, vivo mediante integraciones de canal
  configuradas, respondiendo preguntas de intención y channel-trust para humanos dentro
  de esas apps. La disponibilidad depende del conector de canal desplegado por
  SecureStamp y de las políticas de cada plataforma; SecureStamp no reclama estatus de
  partner oficial ni nativo.
- **Hosts MCP** — el Agent Trust Layer remoto para software.

---

## 7. Transparencia y registros

El historial relevante para seguridad se guarda en **logs de transparencia append-only
tipo Merkle** con pruebas de inclusión y consistencia — la misma familia de construcción
que certificate transparency. Esto es lo que está en producción: respalda **Key
Transparency** para claves de identidad end-to-end-encrypted y transparencia para eventos
de channel-trust, de modo que un cliente puede probar que una entrada existe y que el log
nunca fue reescrito.

Un ledger distribuido permisionado (Hyperledger Fabric) sigue siendo una idea de
**roadmap** para una federación multi-operador futura; no es el sustrato actual y no lo
reclamamos como productivo. Ver [ADR-004](../adr/ADR-004-hyperledger-fabric-ledger.es.md).

---

## 8. Gobernanza

La SecureStamp Foundation mantiene la especificación abierta del protocolo, publica los
ADRs que registran sus decisiones y custodia los contratos de receipt y veredicto que
hacen confiables a los receipts. `securestamp.online` es la implementación comercial de
referencia; el protocolo en sí es abierto y libre de implementar. La competencia sobre un
estándar compartido es intencional.

---

## 9. Qué está live, y qué no

Nos sostenemos a una regla sobre los claims: **no afirmamos compatibilidad que no hayamos
verificado end-to-end.**

**Live hoy:**
- Endpoint de producción de MCP Guard y ambos modos de auth.
- Las seis tools y el modelo de veredicto `allow` / `needs_confirmation` / `block`.
- Action Receipts canónicos con verificación pública.
- Channel-Trust de Telegram y WhatsApp, vivo mediante integraciones de canal configuradas.
- Logs de transparencia append-only Merkle (Key Transparency y channel-trust).

**Roadmap / aún no reclamado:**
- Conformidad verificada con cualquier host o cliente MCP de terceros específico
  (agentes de escritorio, asistentes de IDE) — no probado con smoke independiente; no se
  afirma.
- Un wrapper stdio publicado en un registro público, y listados en marketplaces.
- Un ledger distribuido permisionado como sustrato multi-operador (ADR-004).

---

## 10. Cómo participar

- **Como implementador** — el protocolo es abierto; construí un verificador, una
  integración de agente o un adaptador de canal. Empezá por
  [SECURESTAMP-PROTOCOL-v0.2.es.md](../protocol/SECURESTAMP-PROTOCOL-v0.2.es.md).
- **Como organización** — registrate en [securestamp.online](https://securestamp.online)
  para proteger tus contrapartes, personas y agentes.
- **Como contribuidor** — issues del protocolo, propuestas de ADR y pull requests son
  bienvenidos. Ver [CONTRIBUTING.md](../CONTRIBUTING.md).

---

*SecureStamp Foundation — securestamp.org*
*Licenciado bajo Creative Commons Attribution 4.0 International (CC BY 4.0).*
*Colaboración técnica: Ivan.*
