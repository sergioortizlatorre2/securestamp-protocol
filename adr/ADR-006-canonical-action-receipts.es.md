# ADR-006: Action Receipts canónicos — SecureStamp nunca firma un veredicto declarado por el llamante

## Estado
Aceptado — 2026-07-09

## Contexto

Un entregable central de Proof-of-Intent (ver
[ADR-005](ADR-005-proof-of-intent-pivot.es.md) y
[Protocolo v0.2](../protocol/SECURESTAMP-PROTOCOL-v0.2.es.md)) es un **Action Receipt
verificable**: un registro firmado (ES256) de una acción verificada que un tercero puede
comprobar sin cuenta en `GET /api/action/receipts/<receiptId>`.

Un receipt es tan confiable como la regla que gobierna qué se firma. Si la superficie que
emite receipts aceptara un veredicto, una intención o un "siguiente paso seguro"
*declarado por el llamante* y lo firmara, entonces cualquier llamante autenticado podría
fabricar un "SecureStamp Action Receipt" criptográficamente válido afirmando
`verified_action` — incluso para un `payment_request` — indistinguible de uno legítimo.
Eso permitiría defraudar a un tercero con un receipt "verificado por SecureStamp" que
nunca pasó por ningún cómputo de veredicto. Contradice directamente la doctrina *No agent
action without Proof-of-Intent*.

## Decisión

**SecureStamp nunca firma un veredicto declarado por el llamante.** Un Action Receipt
verificable puede originarse en exactamente dos fuentes canónicas, y en ninguna otra:

1. **`authorize_action`** — computa un veredicto con el motor de veredictos sobre datos
   verificados por el tenant (registro de contrapartes, política, fingerprint de
   instrucciones), lo firma, lo persiste y devuelve un `receiptId` generado por el
   servidor.
2. **Resolución de Action Challenge** — una transición confirmar/rechazar single-use,
   token-gated, computa `challenge_confirmed` / `challenge_rejected` a partir de un
   cambio de estado real, lo firma y lo persiste.

El tool MCP `issue_action_receipt` es, por lo tanto, una operación de
**lookup-y-re-afirmación**, no de emisión:

- Su única entrada real es un `receiptId` (más un `detectedIntent` opcional que debe
  coincidir con el registrado, actuando como fingerprint de la acción).
- Aplica tenant scoping: un receipt de otro tenant devuelve la misma respuesta que "no
  existe", por lo que no es un oráculo de existencia.
- Devuelve el token **ya firmado**; nunca vuelve a llamar al firmante.
- **No** acepta un `verdict`, `reasons` ni `safeNextStep` declarados por el llamante; el
  input schema los rechaza.
- Devuelve un único error unificado para no-existe / otro-tenant / fingerprint no
  coincide.
- Tanto la re-afirmación exitosa como la denegación quedan auditadas, sin payload
  sensible.

La procedencia es entonces una invariante del sistema: sólo los dos flujos canónicos
escriben receipts, así que todo receipt remonta a un veredicto que SecureStamp realmente
computó.

## Consecuencias

**Positivas:**
- Un receipt público es significativo: no puede ser falsificado por un llamante que
  afirme un veredicto sobre sí mismo.
- No se introduce ninguna nueva superficie de firma, tabla ni PKI — la re-afirmación
  reutiliza el storage existente y devuelve un token ya firmado.
- `issue_action_receipt` sigue visible y habilitada en `tools/list`, pero su superficie
  de entrada ya no acepta nada que el llamante pueda inventar.

**Negativas / costos:**
- Los llamantes no pueden "auto-emitir" un receipt para una acción que nunca pasó por el
  motor de veredictos — por diseño; no hay caso de uso legítimo para eso.
- Cualquier flujo futuro que necesite *iniciar* un veredicto fuera de `authorize_action`
  debe agregar esa lógica al motor de veredictos detrás de un endpoint computado
  server-side — nunca reabriendo `issue_action_receipt` a datos declarados por el
  llamante.

## Alternativas consideradas

- **Firmar veredictos declarados por el llamante (comportamiento original).** Rechazado:
  hace falsificables a los receipts y anula la doctrina.
- **Deshabilitar `issue_action_receipt` por completo / devolver `RECEIPT_SOURCE_REQUIRED`.**
  Considerado como fallback de emergencia; innecesario porque el storage existente
  soportaba el modelo canónico completo de lookup.
- **Agregar un tercer camino de escritura de receipts para batch/otros canales.**
  Diferido: la extensión correcta es computar esos veredictos en el motor detrás de un
  endpoint server-side, no relajar la superficie de receipts.
