# ADR-004: Hyperledger Fabric como ledger inmutable para el protocolo SecureStamp

## Estado
Aceptado — 2026-05-24

## Contexto

El protocolo SecureStamp requiere un ledger compartido entre los nodos de la red federada para registrar de forma inmutable:
- Emisiones de stamps (StampIssuance)
- Revocaciones (StampRevocation)
- Cambios de score con evidencia (ScoreChange)

El ledger debe ser auditable públicamente pero escribible solo por nodos aprobados por la fundación. Los eventos registrados son el **estado canónico** del protocolo: cualquier disputa sobre si un stamp fue emitido, cuándo, y por quién, se resuelve contra el ledger.

Las opciones evaluadas:

| Opción | Motivo de descarte |
|---|---|
| **Ethereum / Polygon (blockchain pública)** | Latencia de confirmación inaceptable (>2s) para un protocolo de email en tiempo real. Costos de gas impredecibles. Governance pública no deseada: cualquiera podría escribir. |
| **Hyperledger Besu (Ethereum privado)** | Compatible con EVM pero hereda la complejidad del modelo de cuentas y el tooling de Solidity. La red federada no necesita contratos compatibles con EVM. |
| **Custom append-only chain** | Reinventar consenso, CA management, y gossip protocol sería scope inmanejable y una superficie de ataque sin auditoría externa. |
| **PostgreSQL append-only (insert-only tables)** | No distribuido por naturaleza. Un nodo malo puede alterar su copia local. No provee prueba criptográfica de no-alteración para terceros. |
| **Hyperledger Fabric** | Blockchain permisionado, CA integrada, chaincode en Go/Node, canal único configurable, sin token/gas, consenso Raft maduro, adoptado por Linux Foundation. |

## Decisión

Usar **Hyperledger Fabric** como ledger distribuido e inmutable para todos los eventos del protocolo SecureStamp que requieren consenso entre nodos.

### Componentes y roles

**Certificate Authority (CA)**
- Emitida y operada por la SecureStamp Foundation.
- Firma los certificados de identidad de cada Peer node antes de su admisión al channel.
- Sin certificado CA válido, un nodo no puede leer ni escribir el ledger.
- Permite revocación de nodos comprometidos via CRL.

**Orderer (Raft)**
- Serializa las transacciones y produce bloques.
- MVP: un solo orderer operado por la fundación (tolerable para fase inicial).
- Producción: mínimo 3 orderers en Raft para tolerancia a fallas (F = 1 con 3 nodos, F = 2 con 5).
- No almacena world state; solo ordena y distribuye bloques.

**Peer nodes**
- Cada nodo aprobado de la red federada opera un Peer.
- Mantienen una copia local del ledger y el world state (LevelDB o CouchDB).
- Ejecutan chaincode para validar transacciones antes de endorsar.
- Policy de endorsement: `AND('Foundation.member', 'Org1.peer')` en MVP, expandible a majority en producción.

**Chaincode (smart contracts)**
- Escrito en Go, desplegado en el channel `securestamp-main`.
- Tres assets: `StampIssuance`, `StampRevocation`, `ScoreChange`.
- Toda escritura pasa por chaincode — no hay acceso directo al ledger desde los nodos.
- El chaincode valida que el emisor tenga el rol correcto antes de crear el asset.

### Eventos que registra el ledger

```
StampIssuance {
  stampId:    string   // UUID v4, generado por el nodo emisor
  orgId:      string   // ID de organización registrada en la fundación
  domain:     string   // dominio normalizado (lowercase, sin trailing dot)
  score:      uint8    // 0–100
  signals:    string   // JSON serializado: { spf, dkim, dmarc, ... }
  timestamp:  int64    // Unix epoch ms
  issuedBy:   string   // fingerprint del certificado del nodo emisor
}

StampRevocation {
  stampId:    string   // referencia a StampIssuance.stampId
  reason:     string   // enum: PHISHING_DETECTED | ORG_REQUEST | POLICY_VIOLATION | NODE_COMPROMISE
  revokedBy:  string   // fingerprint del certificado del nodo revocador
  timestamp:  int64
}

ScoreChange {
  domain:     string
  oldScore:   uint8
  newScore:   uint8
  reason:     string   // enum: DMARC_ADDED | PHISHING_CAMPAIGN | TYPOSQUATTING_CLUSTER | MANUAL_REVIEW
  evidence:   string   // JSON array de señales que justifican el cambio
  timestamp:  int64
}
```

### Setup para el MVP

El MVP opera con topología mínima para desarrollo y staging:

```
1 CA (fundación)
1 Orderer (Raft, nodo único)
1 Peer (fundación, org = Foundation)
1 Channel: securestamp-main
Chaincode: securestamp-core v0.1 (Go)
```

Esta configuración no es fault-tolerant pero es funcional, auditablemente correcta (el ledger sigue siendo inmutable y firmado criptográficamente), y suficiente para integrar el primer nodo externo aprobado.

Infraestructura: Docker Compose para desarrollo local, Kubernetes (EKS) para staging/producción.

### Escalado a producción

- Añadir orderers hasta mínimo 3 (Raft majority).
- Cada nueva organización aprobada por la fundación incorpora su Peer al channel.
- Policy de endorsement migra a `MAJORITY` de peers endorsadores.
- CouchDB como state database para queries complejas sobre el world state (ej: todos los stamps activos de un dominio).

## Consecuencias

**Positivo:**
- Ledger inmutable y criptográficamente auditable sin custodio central.
- CA integrada permite revocar nodos comprometidos sin reestructurar la red.
- Sin token, sin gas: la economía del protocolo es independiente de criptomonedas.
- Chaincode en Go con testing estándar.
- Permisionado por diseño: exactamente el modelo que necesita una red federada con nodos aprobados.
- Adoptado por Linux Foundation — credibilidad institucional para el protocolo abierto.

**Negativo:**
- Operacional complejo: CA, orderers, peers, channel config, y chaincode son piezas que requieren DevOps dedicado.
- El MVP con orderer único es un single point of failure operacional (no de integridad — el ledger firmado permanece auditable aunque el orderer caiga).
- Curva de aprendizaje alta para desarrolladores que no conocen Fabric.
- El tooling (peer CLI, configtx) no es tan ergonómico como otras soluciones.
- Latencia de confirmación ~500ms–2s dependiendo del batch timeout del orderer — aceptable para registro de eventos, no para verificación en tiempo real (la verificación usa la DB local de cada nodo, no el ledger directamente).

## Alternativas consideradas

- **Ethereum / Polygon**: descartado por razones económicas y de governance.
- **Hyperledger Besu**: viable técnicamente pero tooling EVM innecesariamente complejo para este caso.
- **Custom append-only chain**: fuera de scope y sin auditoría externa.
- **PostgreSQL append-only**: no distribuido, no provee prueba criptográfica para terceros auditores.
