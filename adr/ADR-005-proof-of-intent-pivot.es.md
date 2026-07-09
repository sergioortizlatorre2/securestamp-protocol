# ADR-005: Pivot de capa de confianza de email a un protocolo de Proof-of-Intent

## Estado
Aceptado — 2026-07-09

## Contexto

SecureStamp v0.1 (ver [Protocolo v0.1](../protocol/SECURESTAMP-PROTOCOL-v0.1.es.md))
encuadraba el problema como **identidad del remitente para email**: un sello verificable
que responde *"¿este dominio es quien dice ser?"*. Ese problema es real y el sello sigue
en producción.

Dos fuerzas cambiaron el centro de gravedad del problema:

1. **El daño es la acción, no el mensaje.** Un mensaje que pasa SPF, DKIM y DMARC — o que
   genuinamente viene de un remitente conocido cuya cuenta fue comprometida — puede aun
   así inducir una acción dañina, a menudo irreversible (un pago, un cambio de datos de
   payout, una aprobación). La autenticación del mensaje no verifica la acción.
2. **Los agentes de IA eliminan la pausa humana.** Un agente autónomo lee un pedido y
   *actúa*. No hay un momento donde una persona inspeccione "esto parece legítimo" antes
   de que el dinero se mueva. La industria ahora necesita un checkpoint que el software
   pueda consultar **antes** de actuar.

El producto, de hecho, ya había desarrollado estas capacidades: verificación de
contrapartes (VendorShield), protección humana/de canales (Guardian) y una interfaz MCP
remota para agentes (MCP Guard), más Action Receipts canónicos. La documentación pública
del protocolo iba por detrás, describiendo aún sólo la capa de confianza de email de
v0.1.

## Decisión

Adoptar **Proof-of-Intent** como doctrina organizadora del protocolo y publicarla como
**Protocolo v0.2** ([spec](../protocol/SECURESTAMP-PROTOCOL-v0.2.es.md)), junto a — no en
reemplazo de — v0.1.

- **Doctrina:** *No se confía en el mensaje. Se verifica la acción.*
- **Claim central:** *No agent action without Proof-of-Intent.*
- **Modelo de verificación:** toda acción sensible se evalúa en cinco dimensiones —
  **intención, contraparte, canal, política, acción** — produciendo un veredicto de
  `allow`, `needs_confirmation` o `block`.
- **Tres pilares** exponen el mismo motor: **VendorShield** (contrapartes/movimiento de
  dinero), **Guardian** (personas/canales), **MCP Guard** (agentes de IA).
- **v0.1 se conserva** como spec histórico y se reencuadra como *entrada* de las
  dimensiones `contraparte` y `canal`, no como el protocolo entero. v0.2 es aditivo: un
  verificador v0.1 sigue funcionando.
- **SecureStamp autoriza; no ejecuta.** El protocolo es un checkpoint, nunca un actor.

## Consecuencias

**Positivas:**
- La documentación pública ahora coincide con lo que efectivamente está en producción
  (MCP Guard está productivo).
- El protocolo aborda la era de los agentes directamente, con un checkpoint que el
  software puede llamar.
- El sello de email se preserva y recibe un rol claro en lugar de descartarse.

**Negativas / costos:**
- Coexisten dos versiones del protocolo; hay que dirigir al lector a la correcta (se
  maneja con banners de "reemplaza a" y el README).
- Un alcance más amplio implica una superficie de conformidad mayor para documentar con
  honestidad (ver ADR-006 para la invariante de receipts, y las secciones "live vs
  roadmap" de v0.2).

## Alternativas consideradas

- **Reescribir v0.1 en su lugar.** Rechazado: borraría el spec de email-trust como
  artefacto distinto y sus puntos de integración aún válidos.
- **Publicar como v1.0.** Rechazado por ahora: v1.0 implica una garantía de
  estabilidad/compatibilidad hacia atrás que no estamos listos para dar mientras la
  superficie orientada a agentes sigue evolucionando.
- **Dejar los docs en v0.1 y actualizar sólo el marketing.** Rechazado: dejaría el
  protocolo abierto describiendo materialmente mal al producto.
