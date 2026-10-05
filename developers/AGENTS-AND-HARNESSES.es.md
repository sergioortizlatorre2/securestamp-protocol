# Agentes y harnesses — Execution Governance para developers

**[English](AGENTS-AND-HARNESSES.en.md) · [Español](AGENTS-AND-HARNESSES.es.md)**

Proteger un servidor MCP es necesario y no alcanza. Un agente de código también llega a la
shell, al sistema de archivos, a scripts, a llamadas directas a APIs y a las automatizaciones
que su harness le deja alcanzar. Esta guía cubre el modelo técnico que usa SecureStamp para esa
superficie más amplia y las herramientas que lo implementan.

> **Disponibilidad, verificada el 2026-09-30.** El paquete publicado `@securestamp/mcp-guard@0.1.0`
> expone sólo el binario de servidor `securestamp-mcp-guard`, y `@securestamp/execution-governance`
> **todavía no está en npm**. Todo lo de [§3](#3-doctor-diagnosticar-los-archivos-que-ya-tenés) y
> [§4](#4-el-escenario-portable) describe la próxima release de esos paquetes. Los comandos se
> muestran tal como van a correr; hasta que la release esté en el registro no se instalan. Los
> conceptos de §1, §2 y §5–§7 aplican hoy a los paquetes publicados de Action Proof.

## 1. El modelo en un párrafo

**Los agentes pueden proponer. La autoridad queda fuera del agente.** Un efecto que importa —un
push, un deploy, un pago, un cambio de permisos— lo admite un Execution Guardian que corrés vos y
que tiene las credenciales del proveedor. El agente nunca las tiene. La cobertura se limita a las
**rutas declaradas y probadas**: una ruta que elude al Guardian no está controlada, y una ruta que
nunca se sondeó se informa como *no evaluada*, nunca como protegida por herencia de otra ruta.

## 2. Tres superficies

| Superficie | Qué sale mal | Control |
| --- | --- | --- |
| **Servidores MCP** | Un servidor lanzado por shell, un secreto escrito en la configuración o un paquete sin versión fijada convierten una herramienta en una ruta de autoridad que nadie revisó. | Doctor sobre el archivo de configuración; MCP Guard media la llamada; Action Proof vincula la autorización al efecto exacto. |
| **Agentes** | El mismo intento puede llegar por MCP, la shell, un script o una llamada directa a la API. Los logs del agente describen lo que dice que hizo, no lo que ocurrió. | El Guardian admite o retiene cada efecto mediado; un observador fuera del agente registra el resultado; un Task Contract acota pasos, recursos, presupuesto y vigencia. |
| **Harnesses** | Mounts, sockets, helpers de credenciales, proxies y la red efectiva deciden la autoridad real del agente, diga lo que diga la configuración declarada. | Un laboratorio reproducible sondea el perfil del harness ruta por ruta contra un baseline permisivo. |

## 3. Doctor: diagnosticar los archivos que ya tenés

Doctor es un diagnóstico **estático**. Lee sólo los archivos que le nombrás, nunca ejecuta
comandos, hooks ni expresiones de ellos, nunca resuelve secretos ni valida tokens, y nunca
sobrescribe el original: una corrección se propone como copia revisable.

```bash
npm install --save-dev @securestamp/mcp-guard   # próxima release

# JSON con un objeto mcpServers (stdio)
npx securestamp-mcp-doctor scan .mcp.json --propose

# Configuración nativa de un cliente de agentes; --format nombra la forma del archivo
npx securestamp-mcp-doctor scan-native ruta/a/settings.json --format=<forma-settings-json>
npx securestamp-mcp-doctor scan-native ruta/a/config.toml   --format=<forma-mcp-toml>

# Un workflow de GitHub Actions que corre un agente
npx securestamp-mcp-doctor scan-workflow .github/workflows/agent.yml

# Un perfil de harness completo (HarnessProfileV1)
npx securestamp-mcp-doctor scan-harness harness-profile.json --propose
```

Corré `securestamp-mcp-doctor` sin argumentos para ver los valores aceptados de `--format`. Nombran una **forma de archivo** —un settings JSON con permisos y herramientas, o un TOML con una tabla de servidores MCP—, no respaldan a ningún cliente. Un
formato que Doctor no reconoce se informa como `UNSUPPORTED_FORMAT`; nunca se omite en silencio.

Hallazgos para configuración de servidores MCP:

| Código | Significado |
| --- | --- |
| `INLINE_SECRET` | Un token, clave o contraseña escrito en la configuración. El valor se redacta y se propone un placeholder de variable de entorno. |
| `SHELL_EXECUTION` | Un servidor lanzado a través de `sh`, `bash`, `pwsh` o `cmd` con `-c`. |
| `UNPINNED_PACKAGE` | Un runner de paquetes invocado sin versión fijada (incluido `@latest`). |
| `NON_STDIO_TRANSPORT` | Un transporte fuera de la familia que cubre este diagnóstico. Se informa, no se adivina. |
| `SUSPICIOUS_ARGUMENT` | Un argumento con forma de credencial o de escape del comando declarado. |
| `MISSING_COMMAND` | Una entrada de servidor sin nada que lanzar. |
| `INVALID_CONFIG` | El archivo no se pudo leer en el formato declarado. |
| `UNSUPPORTED_FORMAT` | No es un formato que cubra este diagnóstico. |

Los modos de configuración nativa y de workflow agregan sus propios códigos —por ejemplo
permisos sin acotar, capas de configuración que un archivo aislado no revela, triggers no
confiables que alcanzan secrets, checkout de contenido externo, actions sin versión fijada y
workflows reutilizables que no se pueden resolver de forma estática. El comando imprime cada
hallazgo con su archivo, campo y corrección propuesta.

**Límites que tenés que conocer.** Un archivo aislado no revela políticas administradas,
overrides por línea de comandos ni configuración heredada; Doctor lista qué capas leyó y cuáles
quedan desconocidas. Una versión fijada no prueba integridad: comparar el artefacto que realmente
corrió con el aprobado corresponde al ejecutor. La limitación conocida de la propuesta actual:
sólo reemplaza variables de entorno cuyo nombre parece un secreto, así que un valor en un
argumento, una URL o un header todavía puede aparecer en ella. **Revisá el patch antes de
aplicarlo.** Un diagnóstico estático nunca acredita aislamiento ni protección.

## 4. El escenario portable

Una corrida se describe con un único documento JSON versionado, que valida el mismo schema en la
CLI, en la biblioteca y en cualquier dashboard que lo consuma. Editarlo crea una revisión nueva;
una corrida es inmutable respecto de la revisión con la que empezó.

```json
{
  "typ": "SSPI-execution-scenario",
  "version": "1",
  "scenarioId": "exact-export-local",
  "revision": 1,
  "name": "Exact Export local fixture",
  "description": "Synthetic reference scenario; it never executes project code.",
  "parameters": { "target": "fixture://project-a", "dryRun": false },
  "source": {
    "kind": "external-project",
    "projectRef": "fixture://project-a",
    "candidateDigest": "sha256:<64 hex>",
    "baseDigest": "sha256:<64 hex>"
  },
  "runner": {
    "runnerId": "runner-lr0-reference",
    "profileId": "lr0-reference",
    "backend": "synthetic-process",
    "observerLeaseSeconds": 3,
    "controlLeaseSeconds": 10
  },
  "limits": { "maxEffects": 1, "timeoutSeconds": 30 },
  "steps": [
    {
      "stepId": "step-1",
      "operation": "customer.securestamp.draft.update",
      "effectDigest": "eff_v1:<64 hex>",
      "resourceDigest": "sha256:<64 hex>",
      "expected": "succeeded"
    }
  ],
  "createdAt": "1970-01-01T00:00:00.000Z"
}
```

```bash
npm install --save-dev @securestamp/execution-governance   # próxima release

npx securestamp-execution-governance validate scenario.json
npx securestamp-execution-governance run scenario.json --hold-before=step-1
```

`validate` imprime el id del escenario, la revisión y el digest. `run` ejecuta la referencia
**sintética**: nunca corre código del proyecto y su reporte se etiqueta como evidencia simulada.
`--hold-before` pone una retención antes de un paso para ver un efecto retenido que nunca se
admite.

El mismo paquete expone los contratos como biblioteca: schemas y digests del escenario, un plano
de control con órdenes idempotentes `hold` / `resume` / `stop`, y reportes redactados que se
pueden construir, verificar y comparar. Importarlo dentro de un agente no le entrega claves, ni
autoridad de aprobación, ni control sobre el observador.

## 5. Exact Export — sólo sale el candidato revisado

1. **Preparar** el cambio en cuarentena, sobre un snapshot del proyecto. El `.git` de la copia de
   trabajo nunca se reutiliza como base confiable.
2. **Congelar** el candidato antes de revisar: base, bytes y digests, ref, destino y preestado.
3. **Autorizar** ese candidato. Un cambio posterior invalida la aprobación; un flag `--yes` no
   reemplaza MFA ni quórum.
4. **Ejecutar** a través del Guardian, comprobando el estado del destino para que el drift y las
   carreras nunca se pisen.
5. **Verificar** la postcondición de forma independiente y conservar el receipt.

El destino inicial es un repositorio Git local. Con un perfil y un destino autorizados, el
adaptador de GitHub crea **sólo una ref nueva**: no actualiza ramas, no hace force-push, merge ni
deploy; abrir un pull request es otro efecto con su propia autorización. La custodia de la
credencial de exportación se afirma sólo para un perfil que la demuestra con identidad, mounts,
sockets y red efectivos, más canarios desde el contexto del agente.

## 6. Hold, resume, stop

| Orden | Qué hace |
| --- | --- |
| `hold` | El Guardian cierra la admisión de nuevos efectos mediados y conserva presupuesto, grants consumidos e historia. No congela llamadas ya enviadas ni encola efectos viejos para después. |
| `resume` | Antes de reabrir la admisión se revalidan mandato, vigencia, política, perfil, observador y presupuestos; cada nueva solicitud se evalúa de nuevo. |
| `stop` | Terminal para la corrida: cierra la admisión de forma durable, cancela lo pendiente y termina los procesos supervisados y sus hijos. Prevalece sobre `resume` y sobre una reconexión. |

El agente puede seguir razonando mientras sus efectos están retenidos; lo correcto es mostrar
*efectos retenidos / proceso en ejecución*. Un HTTP `200` sólo confirma que la orden se recibió:
lo solicitado, lo aplicado y lo observado se informan por separado. Si cae el canal de control
remoto con el observador sano, las admisiones quedan retenidas localmente; si falla el observador,
el perfil corta la salida y termina el contenedor. La parada local nunca depende de un dashboard.

Estados de corrida: `queued · running · held · stopping · stopped · completed · failed · incomplete`.

## 7. Reportes y niveles de evidencia

| Dimensión | Valores |
| --- | --- |
| Resultado de ruta | `PASS · FAIL · SKIP` |
| Cobertura | `protected · contradicted · partial · not_evaluated` |
| Integración | `simulated · integration_real · no_evaluated` |
| Completitud | `complete · incomplete` |
| Outcome del reporte | `PASS · FAIL · INCOMPLETE` |
| Outcome de un efecto | `succeeded · failed_no_effect · indeterminate` |

Tres afirmaciones se mantienen separadas:

- **Checksum informativo** — el digest comprueba los bytes presentados, pero quien pueda
  reescribir el reporte puede recalcularlo; no autentica al emisor.
- **Evidencia firmada confiable para este operador** — un bundle Action Proof completo se verifica
  offline sólo contra anchors de grant y transparencia instalados fuera del artefacto, y enlaza el
  candidato exacto, el destino, la autoridad y el resultado observado del receipt.
- **Reproducido por un tercero** — otra persona corrió el mismo pack y obtuvo el mismo resultado.

Un `FAIL`, un `SKIP` o una ruta sin evaluar quedan a la vista. Un fallo de instrumentación deja la
corrida `INCOMPLETE`, nunca `PASS`, y la ausencia de un evento nunca es evidencia de ausencia.

## 8. Lo que esto no hace

- No vuelve seguro a un modelo ni prueba alineación general; prueba límites de ejecución en un
  perfil declarado.
- No controla rutas que eluden al Guardian.
- No deshace un efecto que ya ocurrió: `hold` y `stop` no son rollback.
- Un receipt demuestra integridad y alcance bajo sus anchors; no demuestra que el código aprobado
  sea benigno ni que alguien independiente lo haya auditado.
- La evidencia ausente o indeterminada conserva su limitación y no cierra el gate. Un resumen
  redactado identifica la evidencia que respalda, pero nunca hereda la firma del bundle completo.
- No es un kill switch para toda la organización: la primera versión controla la corrida elegida
  y sus runners declarados.
- La compatibilidad se declara por perfil, versión y entorno. Un resultado en un harness no se
  transfiere a otro sistema operativo, ruta o cliente.

## 9. Por dónde seguir

- [Guía de inicio](GETTING-STARTED.es.md) — conectarse a MCP Guard, instalar los paquetes
  publicados, verificar un receipt sin conexión.
- [Protocolo v0.2](../protocol/SECURESTAMP-PROTOCOL-v0.2.es.md) — la especificación normativa de
  Proof-of-Intent.
- [securestamp.org/es/docs/action-proof](https://securestamp.org/es/docs/action-proof),
  [/agent-lab](https://securestamp.org/es/agent-lab), [/kit](https://securestamp.org/es/kit) y
  [/evidence](https://securestamp.org/es/evidence) — Action Proof, los perfiles y la matriz de
  evidencia del laboratorio, el kit de configuración y los niveles de evidencia.
- Los contraejemplos son bienvenidos como issues con seed, perfil y resultado observado. Las
  vulnerabilidades se reportan en privado a **security@securestamp.org**.
