# Licencias de SAES

Estado: confirmada por Sergio el 2026-10-30. Coincide con lo que ya publica el repositorio público
`securestamp-protocol`: CC BY 4.0 para la especificación y la documentación, Apache 2.0 para los
paquetes. La revisión legal general de las páginas de SAES es una decisión aparte y no cambia esta
licencia.

SAES tiene dos tipos de material, y cada uno lleva su licencia.

| Material | Licencia | Dónde |
| --- | --- | --- |
| **Texto del estándar**: principios, dimensiones, niveles, knockouts, requisitos, el pack E en prosa, las especificaciones públicas (como [EVIDENCE-BUNDLE.md](EVIDENCE-BUNDLE.md)) y la [política de independencia](INDEPENDENCE.md) | [Creative Commons Atribución 4.0 Internacional (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/deed.es) | Los documentos de SAES y las páginas `/saes` de securestamp.org |
| **Código**: evaluador, CLIs, generador de pruebas con seed, verificador del paquete de evidencia y SDK | Apache 2.0, la licencia que ya declara cada paquete | Los paquetes `@securestamp/saes-policy`, `@securestamp/saes-journey`, `@securestamp/saes-precedents` y `@securestamp/execution-governance`, y el SDK de Python `securestamp-custody` |

## Por qué CC BY 4.0 para el texto

- Permite copiar, traducir, adaptar y usar el estándar comercialmente, también para competir con
  SecureStamp como evaluador. Eso es lo que hace creíble la [independencia](INDEPENDENCE.md).
- Pide atribución, que se indiquen los cambios y que no se sugiera que SecureStamp respalda a
  quien lo usa ni a la versión modificada (CC BY 4.0, §3 y §2.a.6). Que una versión se llame
  «SAES» lo regula la marca, no la licencia (abajo).
- Descartamos CC BY-SA porque obligaría a quien lo adopte dentro de una norma interna a licenciar
  toda esa norma igual.

## Atribución sugerida

> Basado en SAES (SecureStamp Agent Ethics Standard) v0, de SecureStamp, disponible en
> https://securestamp.org/saes bajo CC BY 4.0. Cambios: <describirlos o «sin cambios»>.

## Lo que la licencia no da

- **La marca.** «SecureStamp» y «SAES», y que una evaluación lleve la firma de SecureStamp, no se
  licencian con el texto. Cualquiera puede evaluar con SAES; sólo SecureStamp firma como
  SecureStamp.
- **Un nivel.** Copiar el texto no certifica nada. Un nivel sale de la evidencia, la calcule quien
  la calcule.
