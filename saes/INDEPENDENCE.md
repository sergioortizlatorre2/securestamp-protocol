# Política de independencia de SAES

Estado: borrador (Beta). Pasa por revisión legal antes de su versión estable.

SecureStamp escribe SAES y también vende evaluaciones. Ese doble papel es un conflicto de interés,
y esta política dice cómo lo acotamos y cómo se puede comprobar desde afuera. No lo elimina.

## 1. Pagar no compra un resultado

1. **La misma evidencia válida recibe la misma conclusión técnica, en cualquier plan.** El
   evaluador local y gratuito calcula cualquier nivel con la evidencia que se le presente. Lo que
   se cobra es servicio y capacidad operativa: la evaluación gestionada, la firma de SecureStamp
   como evaluador, el historial, los equipos, la retención y el soporte. La conclusión no se cobra.
2. **Nunca se cobra por saber qué falta.** Cualquier usuario, incluido el plan gratuito, ve todos
   sus knockouts y qué hace falta para cumplirlos.
3. **El plan no figura en la evaluación.** El evaluador no recibe el plan, la facturación ni la
   identidad comercial del cliente, y la ficha no los menciona.
4. **Un resultado no se negocia.** Un cliente puede impugnarlo con evidencia (§3). No puede
   cambiarlo con un descuento, una renovación ni una amenaza de baja.

## 2. Cómo se comprueba desde afuera

- El texto del estándar, el pack E público, el generador de pruebas con seed y el evaluador local
  son públicos. Cualquiera puede recalcular un nivel con la misma evidencia y comparar.
- La evidencia viaja en un [paquete portable](EVIDENCE-BUNDLE.md) que se verifica sin nuestra
  interfaz y separa integridad, fuente y observación.
- La firma de SecureStamp como evaluador vale lo que decida cada verificador. Que una clave esté
  publicada no la vuelve confiable: cada verificador elige qué evaluadores acepta.
- Las variantes de holdout se revelan o se depositan al cerrar su ventana, para que un tercero
  pueda volver a correrlas.

## 3. Impugnar y evaluar sin nosotros

- **Impugnar un requisito** (que está mal, que es injusto o que no se puede cumplir): se abre un
  RFC público. La respuesta queda escrita, con su razón, aunque sea un rechazo.
- **Impugnar una evaluación**: se presenta la evidencia y se pide el recálculo. Si la conclusión
  cambia, se corrige la ficha y queda registrado el cambio y su causa.
- **Evaluadores alternativos**: cualquiera puede evaluar SAES. Un evaluador se registra si cumple
  estos criterios (compromiso previo, todos los intentos con denominadores,
  consultas limitadas, revelación al cerrar la ventana y firma aceptada según la política del
  verificador). SecureStamp no tiene exclusividad sobre la evaluación.

## 4. Lo que esta política no resuelve

- SecureStamp sigue siendo autor y evaluador. La independencia completa requeriría que el
  estándar lo gobierne alguien más, y eso no existe hoy.
- Quien escribe el estándar decide qué se mide. Un sesgo en esa elección no lo corrige el
  recálculo, sólo la crítica externa y los RFC.
- Esta política no reemplaza una auditoría independiente de SecureStamp, que no tenemos.

## 5. Cambios

Toda modificación de esta política queda en el historial público del repositorio, con fecha y
razón. Un cambio que debilite §1 se anuncia antes de entrar en vigor.
