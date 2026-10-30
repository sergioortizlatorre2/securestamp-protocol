# SAES evidence bundle v1

Estado: borrador público (Beta). Implementación de referencia: los comandos `export-bundle` y
`verify-bundle` del paquete `@securestamp/execution-governance`.

Un paquete de evidencia lleva lo que vio quien aprobó o rechazó una delegación (la ficha SAES, la
ejecución derivada, la atestación de configuración efectiva, la cadena del supervisor) en un solo
archivo JSON que cualquiera puede verificar sin SecureStamp: sin servicio, sin interfaz y sin
cuenta. Alcanzan el archivo, las claves públicas que el verificador decide confiar y herramientas
estándar.

## 1. Tres preguntas que no se mezclan

Un «verificado» único esconde cuál de las tres falló. El verificador responde cada una por
separado.

| Pregunta | Qué responde | Qué no responde |
| --- | --- | --- |
| **Integridad** | Los bytes de cada artefacto son los que figuran en la lista | Quién armó la lista |
| **Autenticidad de la fuente** | Una clave que el verificador instaló firmó la lista, y con qué rol la instaló | Que lo firmado sea cierto |
| **Validez de la observación** | Quién observó, qué cubrió y qué dejó sin observar, según lo declarado en la lista firmada | Lo que el observador no vio |

Una firma válida sobre una observación incompleta se informa así: **válida en integridad, válida
en autenticidad e incompleta en observación**. La firma prueba quién lo dijo, nunca que sea verdad.

## 2. Formato

```json
{
  "bundleVersion": "saes-evidence-bundle/v1",
  "artifacts": [
    {
      "name": "saes-report.json",
      "mediaType": "application/json",
      "digest": "sha256:<64 hex>",
      "contentBase64": "<bytes en base64 estricto>"
    }
  ],
  "envelope": { "payloadType": "application/vnd.in-toto+json", "payload": "...", "signatures": [ { "keyid": "sha256:...", "sig": "..." } ] }
}
```

- `artifacts`: de 1 a 64, 8 MiB en total. `name` es relativo, con `/`, sin `.` ni `..` y sin
  repetirse.
- Exactamente uno de `envelope` (firmado) o `statement` (sin firmar). Un paquete sin firmar sólo
  permite comprobar la integridad interna: cualquiera pudo rearmarlo.
- `envelope` es un sobre [DSSE](https://github.com/secure-systems-lab/dsse) con firma ECDSA P-256
  / SHA-256 (DER). `keyid` es `sha256:` más el SHA-256 de la clave pública en SPKI DER.
- El contenido del sobre es un [in-toto Statement v1](https://github.com/in-toto/attestation) en
  JSON canónico (claves ordenadas, sin espacios; coincide con RFC 8785 para estos datos):
  - `subject`: un elemento por artefacto, `{ "name", "digest": { "sha256" } }`.
  - `predicateType`: `https://securestamp.org/attestation/evidence-bundle/v1`.
  - `predicate`:

```json
{
  "signer": { "role": "supervisor | verifier | ci | operator | agent", "name": "..." },
  "observation": {
    "observer": "...",
    "observerIndependentOfAgent": true,
    "observed": ["..."],
    "notObserved": ["..."],
    "limitations": ["..."]
  },
  "producedAt": "2026-10-29T16:00:00.000Z",
  "runId": "opcional"
}
```

Un artefacto con `mediaType` `application/vnd.dsse.envelope.v1+json` es a su vez un sobre DSSE,
por ejemplo la atestación de la configuración efectiva del sandbox. El verificador comprueba también su
firma con el mismo almacén de claves y la informa aparte.

## 3. Reglas de verificación

1. **Almacén de claves.** Lo arma el verificador, nunca el paquete:
   `{ "keys": [{ "role": "supervisor", "publicKeyPem": "..." }] }`. Una clave privada en el almacén
   se rechaza.
2. **Autenticidad.**
   - `unsigned`: no hay sobre.
   - `unknown_key`: ninguna firma es de una clave instalada.
   - `invalid_signature`: la clave está instalada y la firma no verifica sobre el PAE de DSSE.
   - `role_mismatch`: la firma verifica, pero el rol que declara el paquete no es el rol con que se
     instaló la clave. Un agente que firma como supervisor cae acá.
   - `valid`: la firma verifica y el rol coincide.
3. **Integridad.** Para cada artefacto, el SHA-256 de los bytes decodificados tiene que coincidir
   con su `digest` y con el `subject` del mismo nombre. Un artefacto que no figura en la lista, un
   nombre de la lista sin artefacto o un base64 no estricto la invalidan. Si la firma no es
   `valid`, la integridad se calcula igual contra la lista sin autenticar, y el resultado lo dice
   (`listAuthenticated: false`).
4. **Observación.**
   - `complete` si `notObserved` está vacío, `incomplete` si no, `not_stated` si no hay predicado.
   - `authenticated` sólo si la autenticidad es `valid`.
   - Independencia: `self_reported` si firma una clave instalada como `agent` o el observador
     declara no ser independiente del agente; `independent` sólo con una firma válida de otro
     rol; `unknown` en el resto de los casos.

## 3 bis. Exportar sin escribir código

Un manifiesto nombra los archivos (rutas relativas al manifiesto, sin salir de su carpeta) y
declara la observación; una clave P-256 firma:

```json
{
  "artifacts": [{ "name": "ficha.json", "path": "ficha.json" }],
  "predicate": {
    "signer": { "role": "ci", "name": "acme ci" },
    "observation": {
      "observer": "acme ci runner",
      "observerIndependentOfAgent": true,
      "observed": ["diff"],
      "notObserved": ["later runs"],
      "limitations": []
    },
    "producedAt": "2026-10-29T18:00:00.000Z"
  }
}
```

```bash
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out key.pem
securestamp-execution-governance export-bundle manifest.json --key=key.pem > bundle.json
```

## 4. Verificar sin nuestro código

Con un shell y OpenSSL:

1. Decodificar `envelope.payload` (base64) a `payload.json`.
2. Armar el PAE: `DSSEv1 <len(payloadType)> <payloadType> <len(payload)> <payload>`, con las
   longitudes en bytes y en decimal.
3. Decodificar `sig` (base64, DER) y verificar:
   `openssl dgst -sha256 -verify supervisor.pub.pem -signature sig.der pae.bin`.
4. Para cada artefacto, decodificar `contentBase64` y comparar `shasum -a 256` con el `subject`
   del mismo nombre en `payload.json`.
5. Leer `predicate.observation` de `payload.json` y comparar `signer.role` con el rol que vos le
   asignaste a esa clave.

## 5. Límites

- La firma prueba la fuente, no la verdad del contenido ni la calidad de la observación.
- La lista de `notObserved` es lo que el observador declara; lo que no sabe que no vio no figura.
- El paquete no prueba que el revisor haya leído los artefactos, sólo qué tuvo a disposición.
- Sin una clave instalada por el verificador, ningún paquete pasa de integridad interna.
- v1 no incluye sello de tiempo de terceros ni transparencia; `producedAt` es lo que declara quien
  firma.
