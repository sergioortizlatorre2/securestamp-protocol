# SAES v0: SecureStamp Agent Ethics Standard (Beta)

> **Beta · en prueba.** SAES v0 es un borrador abierto a comentarios y puede cambiar entre
> versiones. Califica la ética de cómo se delega en un agente: qué salvaguardas se configuraron y
> cuáles se demostraron. No certifica que un modelo, un agente o una organización sea ético o
> seguro, ni que un efecto aprobado sea benigno, ni que se cumpla una ley. SecureStamp es autor del
> estándar y también ofrece evaluaciones; cualquiera puede impugnar un requisito o una evaluación.

## Qué evalúa

Una tarea definida y el efecto que pide: qué puede alcanzar el agente, quién autoriza cambios,
cómo se prueban los límites y qué ocurrió. Evalúa la delegación, no el carácter moral de un modelo.

- **Siete principios:** alcance, autoridad legítima, supervisión humana significativa,
  proporcionalidad, honestidad y evidencia, privacidad y moderación.
- **Niveles de evidencia:** N1 revisa el diseño; N2 demuestra salvaguardas en simulación; N3 exige
  integración con el destino real.
- **Knockouts:** siete requisitos sin excepción (D1-01, D1-02, D2-01, D2-02, D3-01, D4-01 y D5-01).
- **Planos que no se mezclan:** lo declarado, lo aplicado y atestado, lo demostrado y la conducta
  observada. Lo simulado sigue siendo simulado, y la conducta observada nunca sube un nivel.

## Dos usuarios, la misma evidencia

Desarrollo usa SAES para entregar un cambio; seguridad, para decidir qué puede autorizar. Ven los
mismos hechos y responden preguntas distintas.

## Lo que se ejecuta después del cambio

Un cambio que parece local (un workflow, una dependencia, un script o instrucciones para agentes)
puede preparar otra ejecución con más permisos. SAES señala qué se ejecuta después, cuándo y con
qué permisos declarados, y deja como desconocido lo que no se observó. **La garantía termina donde
termina la observación.**

## Evidencia que se verifica sin SecureStamp

Una revisión se exporta como un paquete que se verifica sin conexión, incluso con OpenSSL, y que
informa por separado la integridad, la fuente y la observación. Formato y procedimiento:
[EVIDENCE-BUNDLE.md](EVIDENCE-BUNDLE.md).

## Independencia

Pagar no compra un resultado: la misma evidencia recibe la misma conclusión en cualquier
modalidad de servicio, nunca se cobra por saber qué falta y cualquiera puede evaluar con SAES.
Detalle y límites: [INDEPENDENCE.md](INDEPENDENCE.md).

## Participar

- **Proponer un cambio (RFC) o impugnar un requisito o una evaluación:** abrí un issue en este
  repositorio con el requisito o la evaluación, la afirmación cuestionada y la evidencia.
- **Registrar un evaluador alternativo:** abrí un issue con el método, la procedencia de la
  evidencia, la reproducibilidad y los conflictos de interés.

Las respuestas quedan escritas, con su razón, aunque sean un rechazo.

## Licencia

El texto de SAES (este documento, [EVIDENCE-BUNDLE.md](EVIDENCE-BUNDLE.md) e
[INDEPENDENCE.md](INDEPENDENCE.md)) se publica bajo [CC BY 4.0](../LICENSE): se puede copiar,
traducir y adaptar, también con fines comerciales, citando la fuente e indicando los cambios. El
código y las herramientas de SAES siguen la licencia declarada por cada paquete; los paquetes
publicados actualmente declaran Apache-2.0. Estas licencias no incluyen las marcas SecureStamp ni
SAES. Detalle y
atribución sugerida: [LICENSE-SPEC.md](LICENSE-SPEC.md).
