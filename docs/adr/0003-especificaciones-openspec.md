# ADR-0003: Especificaciones con OpenSpec y controles de calidad propios

- **Estado:** Aceptado
- **Fecha:** 2026-11-23
- **Principios relacionados:** I (la especificación va primero), II (monolito modular), VII (pruebas)

## Contexto

El proyecto sigue desarrollo guiado por especificaciones con el flujo **spec → plan → tasks**. Se evaluaron OpenSpec y Spec Kit. La constitución pide dos cosas que pesan en la decisión:

- **Principio I.3:** las specs se actualizan cuando cambia el comportamiento; una spec desactualizada es un defecto.
- **Principio II:** el sistema se organiza por módulos.

## Decisión

Se usa **OpenSpec**, complementado con **controles de calidad propios** inspirados en los de Spec Kit, más ligeros. El flujo y los controles están en [`docs/proceso/flujo-especificaciones.md`](../proceso/flujo-especificaciones.md).

1. **Specs vivas por módulo** en `openspec/specs/<módulo>/spec.md`, que siempre describen el estado actual.
2. **Cambios** en `openspec/changes/<cambio>/`, con la correspondencia al flujo actual:
   - spec → `proposal.md` y deltas en `specs/`;
   - plan → `design.md`;
   - tasks → `tasks.md`.
3. **Seis controles** entre pasos (C1 a C6): spec clara, spec de calidad, revisión de la constitución, consistencia, convergencia y listo para archivar.
4. **Idioma:** el contenido va en español. Los encabezados de estructura de OpenSpec (`## ADDED Requirements`, `### Requirement:`, `#### Scenario:`) se dejan en inglés porque la herramienta los usa al archivar.
5. **Escenarios:** formato GIVEN / WHEN / THEN, que equivale al *Dado / Cuando / Entonces* de la constitución.
6. La carpeta `docs/specs/` se elimina: las specs viven solo en `openspec/`.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **Spec Kit** | Coincide exactamente con el flujo actual, tiene la constitución integrada y más controles de calidad (`clarify`, `checklist`, `analyze`, `converge`). Pero organiza las specs por funcionalidad, y no tiene un proceso formal para actualizar una spec después de implementarla: con el tiempo, la regla vigente de un módulo queda repartida entre varias carpetas. Eso choca con los Principios I.3 y II. |
| **Kiro** | Requiere cambiar de IDE. |
| **Sin herramienta** (plantillas propias) | Habría que construir a mano la aplicación de deltas sobre la spec viva, que es justo lo que resuelve OpenSpec. |

## Consecuencias

**Positivas**

- La spec vigente de cada módulo se lee en un solo archivo.
- El historial de cambios queda en `openspec/changes/archive/`, con fecha.
- Se conserva el flujo spec → plan → tasks, con un paso extra al final (archivar).
- Los controles se pueden relajar en cambios pequeños, como indica el documento del proceso.

**Negativas o riesgos**

- Los controles C1 a C6 son propios: OpenSpec no los hace cumplir. **Mitigación:** convertirlos en plantillas de OpenSpec o en comandos personalizados del agente al configurar la herramienta. Hay que revisar en la documentación de OpenSpec (openspec.dev/docs) qué permite su configuración de proyecto y sus *schemas*.
- La constitución no es un concepto nativo de OpenSpec. **Mitigación:** referenciarla en la configuración del proyecto y en las instrucciones del agente, y exigir la tabla del control C3 en cada `design.md`.

## Decisiones derivadas

- Configuración de OpenSpec al armar el esqueleto: `openspec init`, referencia a la constitución y plantillas con los controles.
- Primera especificación: `registrar-recibo-manual`.
