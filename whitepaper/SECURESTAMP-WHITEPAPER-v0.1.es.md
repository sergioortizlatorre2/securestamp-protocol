# SecureStamp: Una capa de confianza para las comunicaciones digitales

> **⚠️ Reemplazado (histórico).** Este whitepaper describe la **capa de confianza de
> email** original. El protocolo pivoteó desde entonces a **Proof-of-Intent** —
> verificar la acción, no el mensaje. Para la narrativa y el manifiesto actuales, leé
> **[Whitepaper v0.2](SECURESTAMP-WHITEPAPER-v0.2.es.md)**. Ver
> [ADR-005](../adr/ADR-005-proof-of-intent-pivot.es.md) para el porqué.

**Versión:** 0.1 — Borrador (reemplazado por v0.2)  
**Fecha:** Mayo 2026  
**Autores:** SecureStamp Foundation  
**Licencia:** CC BY 4.0

---

## Abstract

El email fue diseñado para la apertura, no para la confianza. Cuarenta años después de su invención, cualquiera puede suplantar a cualquiera. SPF, DKIM y DMARC redujeron el spam pero no resolvieron la identidad: un email que pasa los tres controles puede seguir siendo un ataque de phishing. SecureStamp propone una capa complementaria — un sello de confianza verificable, criptográfico y legible por personas — que opera sobre la infraestructura existente sin reemplazarla. Este documento describe el problema, la solución propuesta, el protocolo abierto, el modelo de gobernanza y la visión a largo plazo.

---

## 1. El problema: la identidad no existe en el email

Cada día se envían 3.400 millones de emails de phishing. El daño financiero supera los 10.000 millones de dólares anuales. La causa raíz es arquitectónica: el email no tiene sistema de identidad nativo.

### Lo que existe hoy

- **SPF** verifica que la IP de envío está autorizada por el dominio.
- **DKIM** verifica que el mensaje no fue alterado en tránsito.
- **DMARC** define qué hacer cuando SPF o DKIM fallan.

Estos protocolos resuelven la *integridad del transporte*. No resuelven la *identidad del remitente*. Un atacante de phishing puede registrar `acmec0rp.com`, configurar SPF, DKIM y DMARC correctamente, y enviar emails que pasan los tres controles. El receptor no tiene mecanismo técnico para distinguir `acmecorp.com` de `acmec0rp.com`.

### Lo que no existe

No hay un registro abierto y descentralizado de "este dominio es quien dice ser". No hay una señal universal que diga: "este remitente fue verificado, tiene historial y se comprometió con un estándar de confianza". No hay un sello visible en el que un usuario no técnico pueda confiar de un vistazo.

---

## 2. La solución propuesta: el Stamp

SecureStamp introduce el concepto de **stamp** — un sello criptográfico, públicamente verificable, asociado a un dominio.

Un stamp es:
- **Criptográfico**: firmado con ECDSA P-256, vinculado a un ledger inmutable.
- **Verificable**: cualquiera puede revisar `securestamp.org/verify/<token>` en segundos.
- **Revocable**: si un dominio es comprometido, el stamp se revoca inmediatamente y la revocación queda registrada en el ledger.
- **Visual**: una PostalStamp — una estampilla vintage con bordes perforados — que aparece en firmas de email, extensiones de browser y la página pública de verificación.

```
┌┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┐  ← bordes perforados (verificables)
┊  SECURESTAMP       92¢   ┊  ← denominación = score de confianza
┊  ┌──────────────────────┐ ┊
┊  │      VERIFICADO      │ ┊  ← artwork (hoy: icono; futuro: diseño artista)
┊  └──────────────────────┘ ┊
┊  acmecorp.com             ┊  ← dominio verificado
┊  ✓ CONFIABLE              ┊  ← estado
└┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┘
```

El stamp tiene dos naturalezas:
- **Funcional**: verifica que el remitente es quien dice ser.
- **Visual**: un elemento de identidad distintivo y memorable.

---

## 3. Cómo funciona

### 3.1 Trust scoring

El score (0–100) se calcula a partir de múltiples señales:

| Señal | Peso | Descripción |
|---|---|---|
| SPF | 20% | Registro DNS correctamente configurado |
| DKIM | 25% | Firmas válidas en mensajes salientes |
| DMARC | 25% | Política estricta (reject/quarantine) |
| Antigüedad del dominio | 10% | Dominios registrados recientemente puntúan menos |
| Reputación del MX | 10% | Historial del servidor de correo |
| Historial de comportamiento | 10% | Ausencia de quejas en registros históricos |

| Score | Estado | Significado |
|---|---|---|
| 70–100 | ✓ CONFIABLE | Dominio verificado, historial limpio |
| 30–69 | ⚠ SOSPECHOSO | Configuración incompleta o dominio reciente |
| 0–29 | ✗ BLOQUEADO | Phishing detectado o stamp revocado |

### 3.2 Tres puntos de integración

Un stamp puede verificarse a través de:

1. **DNS TXT record**: `_securestamp.example.com` — verificación de latencia cero desde cualquier sistema.
2. **Header de email X-SecureStamp**: inyectado por el servidor del remitente en cada mensaje saliente.
3. **API pública**: `GET https://securestamp.org/v1/trust/<domain>` — para cualquier sistema sin acceso a DNS.

Estos tres métodos son independientes y complementarios. Cualquiera de ellos es suficiente para verificar.

### 3.3 El ledger inmutable

Cada emisión y revocación de stamp queda registrada en un ledger Hyperledger Fabric permisionado, operado por la SecureStamp Foundation y la red federada de nodos aprobados. El ledger garantiza:

- **Inmutabilidad**: nadie puede alterar el historial de un stamp.
- **Auditabilidad pública**: cualquiera puede consultar el historial completo de transacciones.
- **Descentralización**: ninguna organización única controla el ledger.
- **Sin criptomoneda**: sin tokens, sin gas. El protocolo es económicamente independiente.

---

## 4. El protocolo abierto

El protocolo SecureStamp es abierto, documentado y libre de implementar.

### Principios de diseño

- **Cero infraestructura nueva**: funciona sobre DNS y SMTP existentes.
- **Interoperable**: cualquier cliente de email, mail server o extensión de browser puede implementar la verificación.
- **Backwards-compatible**: no interfiere con SPF, DKIM ni DMARC.
- **Versionado**: versionado semántico con garantía de compatibilidad hacia atrás de 12 meses.
- **Sin punto único de falla**: cualquier nodo puede servir la API de verificación pública.

### Qué define el protocolo

- El formato del DNS TXT record `_securestamp.<dominio>`
- El formato del header de email `X-SecureStamp`
- La estructura del JWT firmado (ES256)
- La API REST de verificación y alertas
- Los tres assets del chaincode (StampIssuance, StampRevocation, ScoreChange)
- El algoritmo de trust scoring y sus pesos
- El proceso de aprobación y revocación de nodos

### Qué NO define el protocolo

- La interfaz de usuario (cada cliente decide)
- La implementación específica (Go, Node, Python, Rust — todas válidas)
- El modelo de negocio de los operadores (cada nodo elige cómo monetizar)

Protocolo completo: [protocol/SECURESTAMP-PROTOCOL-v0.1.es.md](../protocol/SECURESTAMP-PROTOCOL-v0.1.es.md)

---

## 5. La red federada

SecureStamp no es un servicio centralizado. Es una federación de nodos independientes, cada uno operado por una organización aprobada, todos compartiendo un ledger común.

### Qué es un nodo

Un nodo es un servidor que:
- Posee un certificado X.509 emitido por la CA de la fundación.
- Participa en el canal Hyperledger Fabric `securestamp-main`.
- Emite stamps para sus clientes registrados.
- Publica una API de verificación pública.
- Mantiene una base de datos local de correlaciones para detección de amenazas.

### Cómo unirse a la red

1. Enviar una solicitud en `securestamp.org/node-application`.
2. El comité técnico evalúa la organización dentro de 30 días hábiles.
3. Si se aprueba, la fundación emite un certificado X.509.
4. El nodo se une al canal y recibe el ledger completo.

Buscamos activamente organizaciones de regiones geográficas y sectores diversos: ISPs, empresas de seguridad, proveedores de email, instituciones académicas.

---

## 6. Detección de amenazas

Más allá de la emisión de stamps, cada nodo contribuye a un sistema de detección de amenazas en tiempo real:

### Detección de typosquatting
El sistema calcula la distancia Levenshtein entre todos los dominios observados y los dominios registrados. Una distancia ≤ 3 genera una alerta automática.

Ejemplo: `acmec0rp.com` → distancia 1 de `acmecorp.com` → alerta `TYPOSQUATTING_DETECTED`.

### Detección de campañas de phishing
Dominios que comparten el mismo bloque IP `/24`, tienen nombres similares y aparecen en una ventana de 72 horas se agrupan automáticamente como campaña coordinada.

### Alertas en tiempo real
Via email, webhook y WebSocket, las organizaciones reciben notificaciones instantáneas cuando:
- Se detecta un dominio de typosquatting que las apunta.
- Se identifica una campaña de phishing usando su nombre.
- Su score de confianza cae críticamente.
- Cambia la infraestructura DNS de su dominio.

---

## 7. Gobernanza

### La SecureStamp Foundation

La SecureStamp Foundation es el órgano de gobierno del protocolo. Es responsable de:

- Mantener la especificación del protocolo abierto.
- Operar la Certificate Authority raíz.
- Aprobar y revocar nodos en la red.
- Publicar implementaciones de referencia.
- Asegurar la compatibilidad hacia atrás.
- Gestionar disputas entre nodos.

### Principios de gobernanza

- **Transparencia**: todos los cambios al protocolo se documentan como ADRs (Architecture Decision Records) y son públicos.
- **Meritocracia**: las decisiones las toma el comité técnico, no los intereses comerciales.
- **Independencia**: la fundación no favorece ninguna implementación comercial.
- **Apertura**: cualquiera puede proponer cambios a través del repositorio público.

### Relación con lo comercial

La fundación define el protocolo y mantiene la red. `securestamp.online` es la implementación comercial de referencia. Otras organizaciones pueden construir productos comerciales competidores usando el mismo protocolo abierto. Esto es intencional: la competencia mejora el ecosistema sin fragmentar el estándar.

---

## 8. Visión: el stamp como identidad

Creemos que la confianza en las comunicaciones digitales debe ser tan clara y universal como un sello físico en un documento.

El stamp — inspirado en las estampillas postales que certificaron el origen de las cartas físicas durante siglos — es la metáfora visual que hace legible la confianza criptográfica para cualquier persona.

### El futuro coleccionable

Los stamps tienen doble naturaleza. Más allá de su rol funcional, son identidad visual. En el futuro, las organizaciones podrán elegir diseños artísticos de stamps creados por diseñadores — ediciones limitadas que hacen distintiva y memorable la identidad verificada.

Esto transforma el stamp de artefacto técnico de seguridad en elemento de marca: el stamp de una empresa pasa a ser parte de su identidad visual, como un logo o sello corporativo.

El marketplace `securestamp.store` es el hogar de estas colecciones. Los filatelistas digitales — personas que coleccionan stamps de marcas verificadas — completan "álbumes" de confianza. Las organizaciones emiten ediciones limitadas para construir comunidad alrededor de su identidad.

### Por qué esto importa

1. **Para las organizaciones**: un stamp verificado diferencia sus emails de los ataques de phishing que usan su nombre.
2. **Para los usuarios finales**: una señal de confianza visible y comprensible sin necesidad de entender SPF/DKIM/DMARC.
3. **Para la comunidad de seguridad**: un estándar abierto, auditable y descentralizado sin dependencias comerciales.
4. **Para los reguladores**: una infraestructura de cumplimiento de identidad digital que no requiere nueva legislación.

---

## 9. El camino hacia v1.0

| Fase | Descripción | Estado |
|---|---|---|
| **Protocolo v0.1** | Spec, integración DNS/header/API, ledger Fabric | Borrador |
| **Implementación de referencia** | securestamp.online, trust API, node SDK | En desarrollo |
| **Primer nodo federado** | Nodo externo se une a la red | Q3 2026 |
| **Extensiones de browser** | Gmail / Outlook / Firefox | Q4 2026 |
| **Protocolo v1.0** | Primera versión estable, compatibilidad garantizada | Q1 2027 |
| **Red de nodos (5+)** | Nodos en 3+ regiones geográficas | Q2 2027 |

---

## 10. Cómo participar

### Como implementador
El protocolo es abierto. Podés implementar un verificador, un emisor de stamps, una extensión de browser o una integración de mail server sin ningún permiso. La especificación está en [protocol/SECURESTAMP-PROTOCOL-v0.1.es.md](../protocol/SECURESTAMP-PROTOCOL-v0.1.es.md).

### Como organización (emisor de stamps)
Registrarse en [securestamp.online](https://securestamp.online) para obtener un stamp para tu dominio.

### Como operador de nodo
Si tu organización puede contribuir un nodo aprobado a la red, aplicar en `securestamp.org/node-application`.

### Como contribuidor
Este repositorio es abierto. Issues sobre el protocolo, propuestas de ADR y pull requests son bienvenidos.

---

## Apéndice: Referencias técnicas

- [SecureStamp Protocol v0.1](../protocol/SECURESTAMP-PROTOCOL-v0.1.es.md)
- [ADR-004: Hyperledger Fabric como ledger inmutable](../adr/ADR-004-hyperledger-fabric-ledger.es.md)
- API pública: `https://securestamp.org/v1/trust/<dominio>`
- Aplicación de nodo: `https://securestamp.org/node-application`

---

*SecureStamp Foundation — securestamp.org*  
*Este documento está licenciado bajo Creative Commons Attribution 4.0 International (CC BY 4.0).*
