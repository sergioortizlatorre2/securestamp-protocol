# SecureStamp Protocol v0.1

**Estado:** Borrador  
**Fecha:** 2026-05-24  
**Mantenido por:** SecureStamp Foundation  
**Repositorio:** https://github.com/securestamp/protocol

---

## Abstract

El protocolo SecureStamp define un mecanismo estándar para que remitentes de email publiquen un sello verificable de confianza (`stamp`), y para que receptores o sistemas intermedios verifiquen ese sello de forma independiente. El protocolo opera sobre infraestructura DNS existente, headers SMTP estándar, y una API HTTP pública. Un ledger distribuido basado en Hyperledger Fabric garantiza la inmutabilidad del historial de stamps emitidos y revocados.

---

## 1. Terminología

| Término | Definición |
|---|---|
| **Stamp** | Sello criptográfico emitido por un nodo aprobado para un par (orgId, domain) |
| **StampId** | UUID v4 que identifica unívocamente un stamp en el ledger |
| **Token** | JWT firmado con la clave privada del nodo emisor; referencia al stamp |
| **Nodo** | Servidor operado por una organización aprobada por la fundación; escribe al ledger |
| **Score** | Entero 0–100 que representa el nivel de confianza del dominio |
| **Channel** | Canal Hyperledger Fabric `securestamp-main` donde residen todos los eventos |

---

## 2. Integración con email

El protocolo define tres puntos de enganche independientes. Los tres pueden coexistir; la presencia de cualquiera de ellos permite verificación.

### 2.1 DNS TXT record

El propietario del dominio publica un registro TXT en su zona DNS:

```
_securestamp.example.com.  IN  TXT  "securestamp=v=1; id=<stamp_id>; url=https://securestamp.org/verify/<token>"
```

**Campos:**

| Campo | Descripción |
|---|---|
| `v=1` | Versión del protocolo. Obligatorio. |
| `id=<stamp_id>` | UUID v4 del stamp en el ledger. Obligatorio. |
| `url=<url>` | URL canónica de verificación pública. Obligatorio. |

El registro se publica bajo el subdominio `_securestamp.<dominio>` para no interferir con registros TXT existentes (SPF, DKIM, etc.).

**Ejemplo completo:**
```
_securestamp.acmecorp.com.  3600  IN  TXT  "securestamp=v=1; id=f47ac10b-58cc-4372-a567-0e02b2c3d479; url=https://securestamp.org/verify/eyJhbGciOiJFUzI1NiJ9..."
```

**Proceso de verificación via DNS:**
1. Resolver `_securestamp.<dominio>` → obtener TXT
2. Parsear campos; validar `v=1`
3. Llamar `GET https://securestamp.org/v1/trust/<stamp_id>` con el `id` obtenido
4. Verificar que el token JWT en `url` firma el `stamp_id` y no está expirado ni revocado

### 2.2 Email Header

El servidor de correo del remitente inyecta un header personalizado en cada mensaje:

```
X-SecureStamp: v=1; token=<signed_jwt>; verify=https://securestamp.org/verify/<token>
```

**Campos:**

| Campo | Descripción |
|---|---|
| `v=1` | Versión del protocolo. Obligatorio. |
| `token=<jwt>` | JWT firmado por el nodo emisor. Contiene `stampId`, `domain`, `score`, `iat`, `exp`. Obligatorio. |
| `verify=<url>` | URL de verificación pública del stamp. Obligatorio. |

**Estructura del JWT payload:**
```json
{
  "stampId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "domain": "acmecorp.com",
  "orgId": "org_01hwx4...",
  "score": 92,
  "iat": 1716480000,
  "exp": 1748016000,
  "iss": "https://securestamp.org"
}
```

El JWT se firma con `ES256` (ECDSA P-256). La clave pública del nodo emisor se publica en `https://securestamp.org/v1/keys/<node_id>`.

**Proceso de verificación via header:**
1. Extraer header `X-SecureStamp` del mensaje
2. Parsear campos; validar `v=1`
3. Verificar firma JWT contra la clave pública del nodo emisor
4. Validar que `domain` en el JWT coincide con el dominio del `From:` header
5. Verificar que el `stampId` no está revocado en el ledger (via API o caché local)

### 2.3 API query

Cualquier sistema puede consultar el estado de confianza de un dominio sin depender de DNS ni del mensaje:

```
GET https://securestamp.org/v1/trust/<domain>
```

**Respuesta exitosa (200):**
```json
{
  "domain": "acmecorp.com",
  "stamp": {
    "stampId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "score": 92,
    "status": "active",
    "issuedAt": "2026-01-15T10:00:00Z",
    "expiresAt": "2027-01-15T10:00:00Z",
    "issuedBy": "node_foundation_01",
    "signals": {
      "spf": "pass",
      "dkim": "pass",
      "dmarc": "pass"
    }
  },
  "ledgerRef": "https://securestamp.org/v1/ledger/tx/abc123...",
  "verifiedAt": "2026-05-24T12:00:00Z"
}
```

**Respuesta sin stamp registrado (200, no stamp):**
```json
{
  "domain": "unknowndomain.com",
  "stamp": null,
  "verifiedAt": "2026-05-24T12:00:00Z"
}
```

**Respuesta con stamp revocado (200, revocado):**
```json
{
  "domain": "compromised.com",
  "stamp": {
    "stampId": "...",
    "score": 0,
    "status": "revoked",
    "revokedAt": "2026-05-20T08:00:00Z",
    "revocationReason": "PHISHING_DETECTED"
  },
  "verifiedAt": "2026-05-24T12:00:00Z"
}
```

**Rate limits:** 1000 req/hora para IPs sin autenticación. Con `Authorization: Bearer <api_key>` el límite se eleva según plan.

---

## 3. Red federada de nodos

### 3.1 Principios

- Solo nodos aprobados por la SecureStamp Foundation pueden escribir al ledger.
- Cada nodo mantiene su propia base de datos local de correlaciones (PostgreSQL).
- Los nodos se sincronizan a través del ledger Hyperledger Fabric (channel `securestamp-main`).
- La lectura del ledger es pública via API REST expuesta por cualquier nodo.
- Un nodo comprometido puede ser revocado por la fundación via Certificate Authority.

### 3.2 Proceso de aprobación de nodos

```
1. APLICACIÓN
   El operador envía: nombre de organización, ASN, región geográfica,
   capacidad técnica (uptime SLA), uso declarado del nodo.
   Formulario: https://securestamp.org/node-application

2. REVISIÓN
   El comité técnico de la fundación evalúa:
   - Reputación de la organización solicitante
   - Capacidad técnica (uptime histórico, equipo)
   - Ausencia de conflictos de interés con nodos existentes
   - Cobertura geográfica que el nodo aportaría a la red
   Plazo: 30 días hábiles.

3. CERTIFICADO CA
   Si aprobado: la CA de la fundación emite un certificado X.509 para el nodo.
   El certificado identifica al nodo en el channel de Fabric.
   Validity: 1 año, renovable.

4. JOIN CHANNEL
   El operador ejecuta:
     peer channel join -b securestamp-main.block
   con el certificado emitido. El nodo queda como peer del channel
   y recibe el ledger completo (sync desde genesis block).

5. OPERACIÓN
   El nodo puede ahora:
   - Endorsar transacciones
   - Emitir stamps para sus clientes registrados
   - Acceder al world state completo
   - Publicar su API pública de verificación
```

### 3.3 Obligaciones de un nodo operativo

- Mantener uptime ≥ 99% en ventanas de 30 días.
- Correr la versión de chaincode dentro del rango `[current - 1, current]`.
- Reportar anomalías detectadas en su base de correlaciones a la fundación.
- No modificar políticas de endorsement localmente sin aprobación del channel.
- Publicar endpoint público: `https://<node_domain>/v1/health` con respuesta dentro de 2s.

---

## 4. Estructura de bloques (chaincode)

El chaincode `securestamp-core` gestiona tres tipos de assets en el world state del channel:

### 4.1 StampIssuance

```go
type StampIssuance struct {
    StampId   string `json:"stampId"`   // UUID v4
    OrgId     string `json:"orgId"`     // ID de organización en el registro
    Domain    string `json:"domain"`    // dominio normalizado
    Score     uint8  `json:"score"`     // 0–100
    Signals   string `json:"signals"`   // JSON: { spf, dkim, dmarc, mx, rdns, ... }
    Timestamp int64  `json:"timestamp"` // Unix epoch ms
    IssuedBy  string `json:"issuedBy"`  // fingerprint SHA-256 del cert del nodo
}
```

**Key en el world state:** `STAMP:<stampId>`

**Validaciones del chaincode al emitir:**
- El `domain` debe estar normalizado (lowercase, sin trailing dot, sin protocolo).
- El nodo emisor (`issuedBy`) debe tener el atributo `role=node` en su certificado MSP.
- No puede existir un stamp activo previo para el mismo `(orgId, domain)`. Si existe, debe revocarse primero.

### 4.2 StampRevocation

```go
type StampRevocation struct {
    StampId   string `json:"stampId"`   // referencia a StampIssuance.stampId
    Reason    string `json:"reason"`    // enum: PHISHING_DETECTED | ORG_REQUEST | POLICY_VIOLATION | NODE_COMPROMISE | EXPIRED
    RevokedBy string `json:"revokedBy"` // fingerprint del cert del nodo o la fundación
    Timestamp int64  `json:"timestamp"`
}
```

**Key en el world state:** `REVOCATION:<stampId>`

**Efecto:** El chaincode actualiza el asset `STAMP:<stampId>` con `status=revoked`. La revocación es irreversible; un nuevo stamp requiere una nueva `StampIssuance`.

### 4.3 ScoreChange

```go
type ScoreChange struct {
    Domain    string `json:"domain"`
    OldScore  uint8  `json:"oldScore"`
    NewScore  uint8  `json:"newScore"`
    Reason    string `json:"reason"`   // enum: DMARC_ADDED | PHISHING_CAMPAIGN | TYPOSQUATTING_CLUSTER | MANUAL_REVIEW | SIGNAL_DRIFT
    Evidence  string `json:"evidence"` // JSON array de señales con pesos
    Timestamp int64  `json:"timestamp"`
}
```

**Key en el world state:** `SCORE_HISTORY:<domain>:<timestamp>`

Los cambios de score se registran como historial append-only. El score efectivo actual es el `newScore` del registro con mayor `timestamp`.

---

## 5. Correlaciones en la DB local de cada nodo

Cada nodo mantiene una base de datos PostgreSQL local independiente del ledger. Esta DB no está replicada entre nodos — es el espacio de trabajo privado de análisis de cada operador. Sin embargo, los eventos **confirmados** (stamps emitidos, revocaciones, score changes) siempre se originan en el ledger compartido.

### 5.1 Historial de comportamiento temporal por dominio

```sql
domain_behavior (
  domain          TEXT NOT NULL,
  observed_at     TIMESTAMPTZ NOT NULL,
  signal_type     TEXT NOT NULL,  -- SPF_CHANGE | DKIM_ADDED | MX_CHANGE | NS_CHANGE | ...
  old_value       TEXT,
  new_value       TEXT,
  source_node_id  TEXT NOT NULL,
  PRIMARY KEY (domain, observed_at, signal_type)
)
```

Propósito: detectar dominios que modifican su infraestructura DNS de forma abrupta (indicador de takeover o preparación de campaña).

### 5.2 Typosquatting detection

El motor de correlación calcula la **distancia Levenshtein** entre todos los dominios de la base (sin TLD) y los dominios registrados en la fundación.

**Regla:** Si `levenshtein(dominio_nuevo, dominio_registrado) ≤ 3`, el dominio nuevo se marca como candidato a typosquatting y se genera una alerta de nivel `WARNING`.

```sql
typosquatting_candidates (
  suspect_domain      TEXT NOT NULL,
  target_domain       TEXT NOT NULL,  -- dominio legítimo al que apunta
  levenshtein_dist    INT NOT NULL,
  detected_at         TIMESTAMPTZ NOT NULL,
  status              TEXT NOT NULL DEFAULT 'PENDING',  -- PENDING | CONFIRMED | FALSE_POSITIVE
  PRIMARY KEY (suspect_domain, target_domain)
)
```

### 5.3 Reputación por remitente individual

```sql
sender_reputation (
  email_address   TEXT NOT NULL PRIMARY KEY,
  domain          TEXT NOT NULL,
  seen_count      INT NOT NULL DEFAULT 0,
  last_seen       TIMESTAMPTZ,
  complaint_count INT NOT NULL DEFAULT 0,  -- reportes de spam/phishing recibidos
  score_override  INT,  -- score explícito asignado por analista; NULL = usar score del dominio
  notes           TEXT
)
```

Un remitente puede tener un score diferente al de su dominio. Ej: un dominio legítimo con un empleado cuya cuenta fue comprometida.

### 5.4 IPs → dominios → campañas coordinadas de phishing

```sql
ip_domain_map (
  ip_address   INET NOT NULL,
  domain       TEXT NOT NULL,
  first_seen   TIMESTAMPTZ NOT NULL,
  last_seen    TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (ip_address, domain)
)

phishing_campaigns (
  campaign_id   TEXT NOT NULL PRIMARY KEY,  -- UUID v4
  detected_at   TIMESTAMPTZ NOT NULL,
  confidence    NUMERIC(4,3) NOT NULL,  -- 0.000–1.000
  domains       TEXT[] NOT NULL,        -- dominios participantes
  ips           INET[] NOT NULL,        -- IPs compartidas
  evidence      JSONB NOT NULL,
  status        TEXT NOT NULL DEFAULT 'ACTIVE'  -- ACTIVE | MITIGATED | FALSE_POSITIVE
)
```

**Heurística de detección de campaña:** dos o más dominios que comparten el mismo bloque `/24` de IP, tienen Levenshtein ≤ 3 contra el mismo dominio objetivo, y comenzaron a aparecer en una ventana de 72 horas.

---

## 6. Sistema de alertas en caliente

### 6.1 Triggers

El motor de correlación genera una alerta inmediata cuando detecta cualquiera de los siguientes patrones:

| Tipo de alerta | Condición | Confianza mínima para emitir |
|---|---|---|
| `TYPOSQUATTING_DETECTED` | Levenshtein ≤ 3 contra dominio registrado | 0.7 |
| `PHISHING_CAMPAIGN_DETECTED` | Cluster de dominios con IPs compartidas + Levenshtein | 0.8 |
| `SCORE_DROP_CRITICAL` | ScoreChange donde `newScore < 30` y `oldScore ≥ 60` | — (siempre) |
| `STAMP_REVOKED` | StampRevocation confirmada en el ledger | — (siempre) |
| `DNS_INFRASTRUCTURE_CHANGE` | Cambio de NS o MX en dominio con stamp activo | 0.6 |
| `SENDER_COMPROMISE_SUSPECTED` | Remitente con historial limpio súbitamente reportado por ≥ 3 receptores | 0.75 |

### 6.2 Canales de entrega

Las alertas se entregan simultáneamente por tres canales:

**Email (SMTP)**
```
Subject: [SecureStamp Alert] TYPOSQUATTING_DETECTED — acmecorp.com
```

**Webhook (HTTP POST)**
```
POST https://<org_webhook_url>
Content-Type: application/json
X-SecureStamp-Signature: <hmac-sha256>
```

**WebSocket (dashboard en tiempo real)**
```
ws://securestamp.org/v1/ws/alerts
```
Los clientes autenticados reciben un frame JSON por cada alerta generada para sus dominios suscritos.

### 6.3 Payload de alerta

```json
{
  "alertId": "ale_01hwx4...",
  "alertType": "TYPOSQUATTING_DETECTED",
  "affectedDomain": "acmecorp.com",
  "attackerDomain": "acmec0rp.com",
  "detectionConfidence": 0.91,
  "evidence": [
    {
      "type": "LEVENSHTEIN_DISTANCE",
      "value": 1,
      "description": "Character 'o' replaced by '0' at position 6"
    },
    {
      "type": "IP_SHARED",
      "value": "185.220.101.0/24",
      "description": "IP in same /24 as a previously registered campaign"
    }
  ],
  "timestamp": "2026-05-24T12:00:00Z",
  "recommendedAction": "Verify WHOIS records and report to registrar",
  "ledgerRef": null
}
```

`ledgerRef` es `null` si la alerta proviene del motor de correlación local antes de que un evento sea confirmado en el ledger. Se popula con la TX hash una vez confirmado.

### 6.4 Suscripción a alertas

```
POST https://securestamp.org/v1/alerts/subscribe
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "domains": ["acmecorp.com", "acmecorp.io"],
  "channels": ["email", "webhook", "websocket"],
  "webhook_url": "https://hooks.acmecorp.com/securestamp",
  "webhook_secret": "<hmac_secret>",
  "min_confidence": 0.7
}
```

---

## 7. Versionado del protocolo

El protocolo usa versionado semántico con prefijo `v`. La versión se incluye en todos los puntos de integración:
- DNS TXT: campo `v=1`
- Email header: campo `v=1`
- API: path `/v1/`

Cambios que incrementan la versión mayor:
- Cambios incompatibles hacia atrás en el formato de token JWT
- Cambios en el formato de los assets del chaincode
- Cambios en la estructura del DNS TXT record

La fundación mantendrá compatibilidad con la versión anterior durante mínimo 12 meses tras publicar una versión mayor nueva.

---

## 8. Consideraciones de seguridad

- Los tokens JWT deben verificarse contra la lista de claves públicas activas. Una clave rotada invalida todos los tokens firmados con ella.
- El `domain` en el JWT debe coincidir exactamente con el dominio del header `From:` del email después de normalización (lowercase, IDN-decoded).
- Los webhooks deben verificar `X-SecureStamp-Signature` antes de procesar el payload.
- Los nodos no deben exponer el world state de Fabric directamente. Toda lectura pasa por la API REST que filtra campos sensibles (fingerprints de certificados internos).
- Las alertas de alta confianza (`>= 0.9`) deben tratarse como bloqueantes en pipelines de filtrado de email.

---

*Fin del documento — SecureStamp Protocol v0.1*
