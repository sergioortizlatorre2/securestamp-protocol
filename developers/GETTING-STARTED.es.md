# Guía de inicio — SecureStamp para developers

**[English](GETTING-STARTED.en.md) · [Español](GETTING-STARTED.es.md)**

> **Ninguna acción de agente sin Proof-of-Intent.**

Este es el camino práctico: conectar algo, instalar algo, verificar algo.
Para los conceptos, leé el [README](../README.md) y el
[Protocolo v0.2](../protocol/SECURESTAMP-PROTOCOL-v0.2.es.md).

<sub>Endpoints, nombres de tools, dist-tags y versiones de esta página se verificaron en vivo el **2026-09-06**.</sub>

---

## 0. Tres cosas antes de escribir código

1. **SecureStamp autoriza; no ejecuta.** Ninguna tool de SecureStamp mueve dinero, borra datos
   ni realiza una acción destructiva. Si buscás algo que *haga* la cosa, esta no es la capa.
2. **Nunca mandes el texto crudo del mensaje a una tool remota.** Las tools remotas reciben
   *señales abstractas*. La lectura del cuerpo pasa en el dispositivo (SSFML) o en el wrapper
   stdio local (`read_message_request`), nunca por la red.
3. **`npm install` resuelve `latest`, que está deliberadamente atrás del prerelease más nuevo**
   en la línea Action Proof. Ver [§4](#4-instalar-los-paquetes) — y revisá los defectos conocidos
   en [§8](#8-errores-que-deberías-esperar) antes de construir sobre un binario publicado.

---

## 1. Mirá el servicio antes de autenticarte

Todo esto es público — no hace falta clave.

```bash
# versión del servicio, commit desplegado, hash del catálogo de tools
curl -sS https://mcp.securestamp.online/version

# el catálogo completo de tools, scopes, límites y doctrina
curl -sS https://mcp.securestamp.online/.well-known/securestamp-mcp.json

# salud
curl -sS -o /dev/null -w '%{http_code}\n' https://mcp.securestamp.online/healthz
```

`/version` informa la versión de protocolo MCP (`2025-11-25`), el `commitSha` desplegado y un
`catalogHash`. Pinneá el `catalogHash` si querés detectar un cambio en el catálogo de tools.

Una llamada sin autenticar a `/mcp` falla, como corresponde, y te dice cómo autenticarte:

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

## 2. Conectarse al MCP Guard remoto

Tres modos de auth. Elegí según *quién* autoriza.

### 2a. API key — `ss_live_…` (una máquina se autoriza a sí misma)

Creá un cliente y una clave desde `.online` → `/dashboard/mcp` (se muestra una sola vez).
Guardala en secreto; nunca la commitees.

```bash
BASE=https://mcp.securestamp.online
KEY=ss_live_...

curl -sS $BASE/mcp \
  -H "Authorization: Bearer $KEY" \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Pedir un veredicto:

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
        "counterparty":"proveedor@ejemplo.com",
        "sourceChannel":"email",
        "amount":18400,
        "currency":"EUR"
      }
    }
  }'
```

`actionType` es un enum cerrado — `payment_request`, `bank_account_change`,
`monetary_operation_change`, `credential_request`, `mfa_code_request`, `risky_attachment`,
`support_contact`, `software_install`, `document_upload`, `crypto_transfer`,
`identity_verification_request`, `unknown_sensitive_action`. Los campos desconocidos se
rechazan, no se ignoran.

### 2b. Login delegado — `ss_sess_…` (un humano autoriza un cliente)

Para cuando una persona debe autorizar una herramienta sin pegar una clave de larga vida.

1. En `.online` → `/dashboard/mcp` → **Conectar con login** → un código de pareo de un solo
   uso (TTL de 10 minutos).
2. El cliente canjea el código por un session token:
   ```bash
   curl -sS https://securestamp.online/api/mcp/delegated/exchange \
     -H 'content-type: application/json' \
     -d '{"code":"XXXX-XXXX-XXXX-XXXX-XXXX"}'
   # → { "sessionToken": "ss_sess_...", "expiresAt": "...", "allowedTools": [...] }
   ```
3. Usalo contra `/mcp` exactamente como una API key.
4. Revocá desde el dashboard. Las sesiones duran 30 días y no hay refresh — la revocación
   falla cerrada con `401`.

### 2c. OAuth 2.1 — authorization-code + PKCE

Para clientes MCP que hablan OAuth. La metadata de protected-resource se sirve en
`/.well-known/oauth-protected-resource`, y el `401` de arriba le entrega a tu cliente la URL
de descubrimiento y los scopes. Los bearer tokens se aceptan **sólo por header**.

### Configuración de un host remoto

La forma verificada contra el `@modelcontextprotocol/sdk` oficial
(`StreamableHTTPClientTransport`) — la librería que los hosts MCP remotos usan internamente:

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

> Verificamos contra el SDK oficial, no contra la GUI de ninguna aplicación en particular.
> Confirmá la forma exacta de la config contra la documentación y la versión de tu host.

---

## 3. Correrlo por stdio

Para hosts que sólo hablan stdio. Requiere **Node.js 20+**, cero dependencias de runtime.

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

**Lo que el `0.1.0` publicado realmente expone son seis tools**, no nueve: `authorize_action`,
`analyze_message_intent`, `verify_counterparty`, `create_action_challenge`, `get_safe_next_step`
e `issue_action_receipt`. Verificado abriendo el tarball publicado el 2026-09-06.

Las tres tools de la capa de ejecución (`request_execution_grant`, `get_execution_status`,
`get_source_envelope`) y el lector local `read_message_request` existen en el árbol de fuentes
pero **no** están en la build publicada. Si las necesitás, usá el endpoint remoto — o, para
ejecución, el bridge aparte `@securestamp/execution-guardian-mcp`.

El servicio remoto always-on es la superficie principal; el wrapper es el fallback stdio.

---

## 4. Instalar los paquetes

Siete paquetes públicos, **Apache-2.0**.

> **`npm install` resuelve `latest`, y `latest` suele estar atrás.** En cuatro paquetes el
> prerelease más nuevo está bajo **`beta-unverified`**, que significa instalable y descubrible
> pero **sin** pasar el gate de evidencia externa. Elegí el tag a propósito.
>
> El caso más filoso es el verificador: su `latest` es `0.3.0-beta.1`, cuyo `bin` no corre. La
> build reparada es `0.3.0-beta.3`, publicada sólo bajo `beta-unverified`.

```bash
# estable, sin tag
npm install -g @securestamp/cli                          # → 1.0.1
npm install @securestamp/mcp-guard                        # → 0.1.0

# lo que te da `latest` en la línea Action Proof
npm install @securestamp/action-proof                     # → 0.2.0-beta.1  (¡no 0.3!)
npm install @securestamp/action-proof-verify              # → 0.3.0-beta.1  (el bin no corre)

# hoy igual que latest, pero fijado al canal beta
npm install @securestamp/action-proof@beta                # → 0.2.0-beta.1

# la línea 0.3, explícita, sabiendo lo que el tag te retiene
npm install @securestamp/action-proof@beta-unverified     # → 0.3.0-beta.2
npm install @securestamp/action-proof-verify@beta-unverified  # → 0.3.0-beta.3  (bin funcionando)
```

| Paquete | Deps | Node | `latest` | `beta` | `beta-unverified` |
| --- | --- | --- | --- | --- | --- |
| `@securestamp/cli` | 3 | ≥20 | `1.0.1` | — | — |
| `@securestamp/mcp-guard` | 0 | ≥20 | `0.1.0` | — | — |
| `@securestamp/action-proof-verify` | 0 | ≥20 | `0.3.0-beta.1` | — | `0.3.0-beta.3` |
| `@securestamp/action-registry` | 0 | ≥20 | `0.3.0-beta.1` | — | `0.3.0-beta.2` |
| `@securestamp/action-proof` | 1 | ≥20 | `0.2.0-beta.1` | `0.2.0-beta.1` | `0.3.0-beta.2` |
| `@securestamp/execution-guardian` | 9 | ≥22.5 | `0.2.0-beta.2` | `0.2.0-beta.2` | `0.3.0-beta.2` |
| `@securestamp/execution-guardian-mcp` | 3 | ≥22.5 | `0.2.0-beta.1` | `0.2.0-beta.1` | `0.3.0-beta.2` |

Un `—` significa que el tag no existe en ese paquete, no que apunte a otro lado. Ninguna versión
publicada de ningún paquete lleva attestation de provenance de npm. El `LICENSE` ya viaja en
`@securestamp/cli@1.0.1` y en `@securestamp/action-proof-verify@0.3.0-beta.3`; el `latest` del
verificador (`0.3.0-beta.1`) sigue sin ninguno.

### El CLI

```bash
npm install -g @securestamp/cli
ss check securestamp.org --json      # sin API key
ss check alguien@ejemplo.com
ss status                            # necesita clave
```

`ss check` corre contra la API pública de trust y devuelve score, estado, nivel, señales
SPF/DKIM/DMARC y razones. Las claves se guardan con permisos `0600` en
`~/.securestamp/config.json`; `SS_API_KEY` y `SS_API_BASE` pisan el archivo.

`1.0.1` está en `latest`, así que un install pelado ya lo trae. Reparó tres comandos que estaban
rotos en `1.0.0`, verificados ejecutándolos: `ss check` sin `--json` y `ss batch` crasheaban con
un `TypeError` por un campo de confianza que la API no devuelve, y `ss registry` respondía
`HTTP 404` porque su ruta la sirve `.org`, no la base `.online` por defecto. `1.0.1` además deja
de contar un estado desconocido como confiable, lee su versión de `package.json` en vez de un
string hardcodeado, incluye `LICENSE`, y baja el piso de `engines` de `>=22.22.2` —que dejaba
afuera a Node 20 LTS— a `>=20.0.0`.

> **Fijá `>=1.0.1`** si scripteás contra el CLI. En `1.0.0` sólo se portan bien
> `ss check --json`, `ss status`, `ss login` y `ss logout`.

**Empezá por el verificador.** Es la puerta de entrada del ecosistema, no un accesorio: sin
acceso a red, sin dependencias, y se publica *antes* de lo que verifica, así que un
verificador ya liberado acepta una versión nueva de bundle antes de que alguien la emita. Por
eso su `latest` va más adelante que el del emisor.

---

## 5. Verificar sin cuenta

Traer un receipt por HTTP — sin clave, sin login:

```bash
curl https://securestamp.online/api/action/receipts/<receiptId>
# id inexistente → HTTP 404 {"error":"Receipt not found"}
```

Para verificación criptográfica de verdad, hacela **offline**.
`@securestamp/action-proof-verify` nunca contacta a SecureStamp, nunca resuelve JWKS por red y
nunca confía en un SDK de proveedor. Vos aportás los trust anchors; él comprueba localmente
cada firma, digest, audience, grant de un solo uso, binding de política local e inclusión en el
log de transparencia.

Usá la **API de librería**. El paquete declara un binario `action-proof-verify`, pero el
`dist/cli.js` publicado no tiene shebang en ninguna de las dos versiones liberadas, así que el
ejecutable no corre — corregirlo requiere un release.

```js
import { verifyActionProofBundleOffline, jcs } from '@securestamp/action-proof-verify'

// JSON canónico RFC 8785 — claves ordenadas, bytes deterministas para hashear
jcs({ b: 1, a: [2, { d: 4, c: 3 }] })  // {"a":[2,{"c":3,"d":4}],"b":1}

// LANZA excepción ante un bundle malformado en vez de devolver { ok: false } — capturala.
try {
  const result = await verifyActionProofBundleOffline({ bundle, trustAnchors })
  console.log(result)
} catch (err) {
  console.error('rechazado:', err.message)   // p. ej. INVALID_ACTION_PROOF_BUNDLE
}
```

Fallar cerrado es el punto: un bundle imparseable levanta excepción en vez de resolver a un
"no" blando.

Acepta `ActionReceiptV2` y `ActionReceiptV3`, y `ActionProofBundleV1` y `ActionProofBundleV2`.
Informa qué pudo y qué no pudo establecer — `policyEvidence` como `verified` o
`legacy_untrusted`, `adapterManifestEvidence` como `verified_v1` o `legacy_opaque` — en vez de
colapsar las dos cosas en un solo booleano.

El paquete además publica el corpus de interoperabilidad RFC 8785 (JCS) en
`@securestamp/action-proof-verify/vectors/jcs-rfc8785.json`, byte-idéntico al del emisor. Podés
testear una implementación de terceros contra el corpus sin confiar en ninguno de los dos
paquetes.

**SecureStamp nunca firma un veredicto declarado por el llamante.** `issue_action_receipt` sólo
re-afirma un receipt que SecureStamp ya calculó y firmó, por su `receiptId`, después de
comprobar que pertenece a tu tenant. Ver [ADR-006](../adr/ADR-006-canonical-action-receipts.es.md).

---

## 6. El Execution Guardian — lo corrés vos, las credenciales son tuyas

El Guardian es la parte de la arquitectura que en runtime *no* es nuestra.

```
SourceEnvelope firmado por el dispositivo → ActionEffect canónico → ExecutionGrant de la nube
      → TU Guardian lo reclama, lo ejecuta, firma el claim → ActionReceipt
```

- El daemon corre en **tu** entorno y es el único proceso que lee tus credenciales de
  proveedor. **SecureStamp Cloud nunca las recibe ni ejecuta una operación del proveedor.**
- El bridge (`@securestamp/execution-guardian-mcp`) es **credential-free**. Habla con el daemon
  sólo por un Unix socket — sin listener HTTP, sin URL arbitraria, sin SDK de proveedor, sin
  clave de firma. Directorio del socket `0700`, socket `0600`. Si el daemon no está, falla
  cerrado con `GUARDIAN_DAEMON_UNAVAILABLE`.
- Un grant de la nube es **necesario pero nunca suficiente**: el permiso efectivo es la
  intersección del grant, la política local que *vos* firmás, las restricciones del adapter y
  los kill switches. La nube puede angostarlo; no puede ensancharlo.
- Los grants son audience-bound, cortos y `maxUses=1`. El daemon reclama uno de forma durable
  *antes* de mutar y reconcilia resultados ambiguos en vez de reintentar a ciegas.
- Los límites temporales de la política son gates de ejecución, no advertencias: pasado
  `reviewAfter` obtenés `LOCAL_POLICY_REVIEW_REQUIRED`, pasado `expiresAt` obtenés
  `LOCAL_POLICY_EXPIRED` — ambos sin acceso al proveedor.
- La clave de firma de la política es **del cliente y SecureStamp no puede recuperarla**. El
  CLI del daemon provee una ceremonia local M-de-N (`policy init` / `approve` / `recover` /
  `rotate`).

Sandbox es el default. Producción es un opt-in explícito que exige
`GUARDIAN_MODE=production_opt_in` más una `GuardianLocalPolicyV1` firmada por el cliente, y
sigue siendo tu responsabilidad. Una política firmada cuyo entorno declarado no coincide con el
modo efectivo se rechaza; omitir el modo nunca sube el default seguro.

**SecureStamp prueba lo que se ejecutó a través del Guardian enrolado. No puede impedir
acciones fuera de banda en el proveedor.** No uses credenciales que no controlás.

---

## 7. Límites de privacidad — qué se envía y qué nunca sale del dispositivo

**SSFML** (SecureStamp ML) es la capa de reconocimiento on-device dentro de los plugins de
email y la extensión de navegador. Lee el cuerpo **localmente** — tokenización, extracción de
entidades (dinero, intents de autenticación, URLs, adjuntos, QR), el motor híbrido de
señales/reglas, y detección de instrucciones candidatas (referencias IBAN/CBU/CLABE/cripto).

**El cuerpo nunca sale del dispositivo.** Lo que llega a la API son señales abstractas, intents
y metadata de versión.

En concreto, por superficie:

| Superficie | Sale del dispositivo | Nunca sale |
| --- | --- | --- |
| SSFML en un plugin | señales abstractas, intents, ids de regla, flags de adjunto/QR, versiones de motor y pack | cuerpo del mensaje, texto del asunto, contenido de adjuntos |
| `analyze_message_intent` | un objeto `signals` que construís vos | texto crudo del mensaje |
| `authorize_action` | tipo de acción, contraparte, canal, monto/moneda, strings opacos de evidencia | el mensaje que lo motivó |
| `get_source_envelope` | hashes opacos `ref_v1:` / `acct_v1:`, digest de contenido, id de clave de dispositivo | identificadores de casilla, números de cuenta, cuerpo |
| `read_message_request` (wrapper stdio) | nada — corre localmente | todo queda en tu proceso |
| `ss check` | el dominio o email que pasás | nada más |

Los localizadores de origen y cuenta se hashean a `ref_v1:<sha256>` y `acct_v1:<sha256>`
**antes** de firmar; el verificador rechaza de plano identificadores de casilla y números de
cuenta crudos. Los source envelopes además rechazan recursivamente campos con forma de cuerpo
crudo, así que un campo futuro no puede colar un cuerpo dentro de un objeto firmado por accidente.

Las instrucciones monetarias en texto plano no deben mandarse a tools remotas; el matching
monetario se fingerprintea del lado del servidor.

Viajan dos versiones: el **motor** (`2.5.0`, compilado en el plugin, cambia sólo cuando se
publica el plugin) y el **knowledge pack** (auto-actualiza desde el canal `stable`).

El canal de packs es sólo datos y firmado: una firma ES256 sobre un SHA-256 del artefacto,
clave pública pinneada en el plugin, y un build estable **rechaza un pack sin firma o inválido**
y cae al dato empaquetado. Un pack sólo puede agregar frases literales a signal IDs que el motor
empaquetado ya conoce, dentro de límites acotados de peso. No puede introducir una regla, un
operador ni una categoría — y **no puede bajar un veredicto por debajo de lo que habría
producido el motor empaquetado**.

La salida de SSFML es una *entrada* a la acción propuesta, nunca autoridad. No puede elevar la
procedencia; Action Proof y el Guardian siguen exigiendo por su cuenta la operación registrada,
el efecto, la política y las aprobaciones.

**El alcance lingüístico de SSFML v2 es el español**, y eso es una decisión, no un hueco. El
mecanismo de entrega está completo — schema, compilación, escapado contra inyección de regex,
límites de tamaño y costo, firma, allowlist, versionado, activación atómica, rollback, fallback
fail-closed. Un segundo idioma no necesita mecanismo: necesita corpus medido. Escribir sus
tablas gramaticales sin él sería inventar la medición misma que el canal existe para transportar.

---

## 8. Errores que deberías esperar

| Qué ves | Qué significa |
| --- | --- |
| `401` + `WWW-Authenticate` en `/mcp` | No hay token, o es inválido. Seguí `resource_metadata`. |
| `404 {"error":"Receipt not found"}` | El `receiptId` no existe, o no es tuyo. |
| `409` en un `requestFingerprint` | El fingerprint no coincide con el receipt referenciado. Es sólo un id de **correlación** — nunca una afirmación de procedencia, y nunca cambia un veredicto. |
| `GUARDIAN_DAEMON_UNAVAILABLE` | El bridge no pudo alcanzar tu daemon por el socket. Falla cerrado por diseño. |
| `LOCAL_POLICY_REVIEW_REQUIRED` / `LOCAL_POLICY_EXPIRED` | Tu política local firmada está vencida o expirada. La ejecución se detiene antes del acceso al proveedor. |
| `RESOURCE_BINDING_MISMATCH` | Se sustituyó un recurso, tenant, destino, cuenta, región, rol o manifiesto de tool después de autorizado el efecto. |
| `BUNDLE_VERSION_MISMATCH` | Un `ActionProofBundleV2` se apareó con una versión de receipt que no corresponde. |
| `INVALID_ACTION_PROOF_BUNDLE` | El verificador offline **lanza** esto ante un bundle malformado. Capturalo; no devuelve `{ ok: false }`. |
| `action-proof-verify: import: command not found` | El binario publicado del verificador no tiene shebang. Usá la API de librería ([§5](#5-verificar-sin-cuenta)). |
| `ss registry` → `HTTP 404` | Defecto del CLI publicado: apunta a `.online`, donde esa ruta no existe. Usá `SS_API_BASE=https://securestamp.org`. |
| `Invalid API key format` en `ss login` | Las claves empiezan con `ss_live_` o `ss_test_`, no con el `sk_live_` que muestra el README del CLI publicado. |

Cada llamada de respaldo es tenant-scoped, rate-limited y auditada. Las fallas de auth, los
requests inválidos, los rate limits, `initialize`/`tools/list` y los health checks nunca se
facturan.

---

## 9. Adónde ir después

- [README](../README.md) — la idea, los pilares, qué está live y qué no
- [Protocolo v0.2](../protocol/SECURESTAMP-PROTOCOL-v0.2.es.md) — la spec normativa
- [Whitepaper v0.2](../whitepaper/SECURESTAMP-WHITEPAPER-v0.2.es.md) — el manifiesto y el modelo
- [ADR-005](../adr/ADR-005-proof-of-intent-pivot.es.md) — por qué pivotó el protocolo
- [ADR-006](../adr/ADR-006-canonical-action-receipts.es.md) — por qué un receipt no puede declararlo el llamante
- [securestamp.org/es/docs/action-proof](https://securestamp.org/es/docs/action-proof) — la documentación de la capa de ejecución
- npm: [`@securestamp`](https://www.npmjs.com/org/securestamp)

Issues sobre el protocolo, propuestas de ADR y pull requests son bienvenidos — ver
[CONTRIBUTING.md](../CONTRIBUTING.md). Reportá vulnerabilidades en privado a
**security@securestamp.org**; no abras un issue público.
