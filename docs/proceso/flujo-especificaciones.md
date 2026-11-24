# Flujo de especificaciones y controles de calidad

> **Versión:** 0.2.0 · **Fecha:** 2026-11-23
>
> Cómo se usa OpenSpec en este proyecto ([ADR-0003](../adr/0003-especificaciones-openspec.md)) y qué controles debe pasar cada cambio. Los controles son una versión ligera de los de Spec Kit (`clarify`, `checklist`, `analyze`, `converge`), con prácticas que funcionaron en un proyecto anterior (gym-app).

---

## Dónde vive cada cosa

| Ruta | Contenido |
|---|---|
| `openspec/specs/<module>/spec.md` | **Spec viva:** cómo funciona hoy cada módulo. No se edita a mano; se actualiza al archivar cambios. |
| `openspec/changes/<change>/` | Un cambio en curso: `proposal.md`, `specs/` (deltas), `design.md`, `tasks.md`. |
| `openspec/changes/archive/` | Cambios terminados, con fecha. |
| `docs/constitution.md` | Reglas del proyecto. Todo cambio se revisa contra ella. |
| `docs/arquitectura/mapa-modulos.md` | Módulos, contratos y eventos. Se actualiza **antes** del código si un cambio agrega o modifica contratos. |
| `docs/arquitectura/glosario.md` | Traducción español → código. Un concepto nuevo se agrega antes de usarlo. |
| `docs/handoff.md` | Estado actual para retomar en un chat nuevo. Se actualiza al final de cada sesión. |
| `docs/retros/` | Retrospectivas por hito. |

Las carpetas de `openspec/specs/` corresponden a los **módulos** del mapa, con su nombre en inglés ([ADR-0004](../adr/0004-idioma-del-codigo.md)): `identity`, `properties`, `ingestion`, `utilities`, `occupancy`, `notifications`. El contenido de las specs va en español.

---

## IDs estables

Cada requisito y escenario tiene un ID con el prefijo de su módulo ([glosario](../arquitectura/glosario.md)). El ID va **dentro del encabezado**, que es lo que OpenSpec usa para identificar el requisito al aplicar deltas:

```markdown
### Requirement: UTL-R01 Registrar recibo manual
El sistema DEBE (SHALL) permitir al dueño registrar un recibo capturando…

#### Scenario: UTL-R01-S1 Recibo con fecha límite futura
- **GIVEN** un servicio de luz registrado
- **WHEN** el dueño registra un recibo de $842.50 con fecha límite 2026-12-03
- **THEN** el recibo queda en estado `Unpaid`
```

- Los requisitos se numeran por módulo (`R01`, `R02`…) y los escenarios por requisito (`S1`, `S2`…). **Un ID nunca se reutiliza**, aunque el requisito se elimine.
- Las pruebas citan el ID en su nombre: `UTL_R01_S1_recibo_con_fecha_limite_futura` (o en inglés: `UTL_R01_S1_bill_with_future_due_date`).
- Con `grep UTL-R01` (o `UTL_R01`) se encuentra la spec, el diseño, las tareas y las pruebas.

## El marcador `[POR DEFINIR]`

Si la spec o el diseño no definen algo, se escribe `[POR DEFINIR]` con la pregunta, en lugar de decidirlo en el código. **Mientras exista un `[POR DEFINIR]` en un artefacto, no se pasa a la siguiente etapa.** Se revisa con:

```bash
grep -rn "\[POR DEFINIR\]" openspec/changes/<change>/
```

---

## El flujo

| # | Paso | Artefacto | Comando | Control al terminar |
|---|---|---|---|---|
| 1 | **Explorar** (opcional) | — | `/opsx:explore` | — |
| 2 | **Spec** | `proposal.md` y deltas en `specs/` | `/opsx:propose <name>` | **C1. Spec clara** y **C2. Spec de calidad** |
| 3 | **Pantalla** (si tiene interfaz) | Diseño elegido, enlazado en `proposal.md` | — | El diseño está aprobado |
| 4 | **Plan** | `design.md` | (parte de la propuesta) | **C3. Revisión de la constitución** |
| 5 | **Tasks** | `tasks.md` | (parte de la propuesta) | **C4. Consistencia** |
| 6 | **Implementar** | Código y pruebas, por puntos de control | `/opsx:apply` | Mutación al cerrar cada punto de control |
| 7 | **Verificar** | — | `/opsx:verify` | **C5. Convergencia** |
| 8 | **Archivar** | Spec viva actualizada | `/opsx:archive` | **C6. Listo para archivar** |

Un control que no pasa **regresa al paso anterior**: no se avanza con pendientes.

---

## Los controles

### C1. Spec clara *(equivalente ligero de `clarify`)*

- [ ] No queda ningún `[POR DEFINIR]`: todas las preguntas están resueltas o explícitamente pospuestas, con el motivo.
- [ ] Están claros los actores (dueño, administrador, sistema) de cada requisito.

### C2. Spec de calidad *(equivalente ligero de `checklist`)*

- [ ] Cada requisito tiene **ID**, usa **SHALL / DEBE** y describe **un solo** comportamiento.
- [ ] Cada requisito tiene **al menos un escenario con ID**, y los escenarios cubren el caso feliz y al menos un caso de error o límite.
- [ ] Los escenarios son **verificables**: sin palabras vagas como "rápido", "fácil" o "adecuado" sin una medida.
- [ ] Los montos, fechas y periodos de los escenarios usan valores concretos (por ejemplo, "$842.50", "2026-12-03").
- [ ] La propuesta declara **qué módulos toca** y **qué contratos o eventos agrega o modifica** (constitución, Principio I.4).
- [ ] Hay un delta en `specs/<module>/` por cada módulo que cambia de comportamiento.
- [ ] Los conceptos nuevos están en el glosario.

### C3. Revisión de la constitución *(lo que Spec Kit hace en `plan`)*

`design.md` incluye **los casos de borde de cada regla** y termina con esta tabla, una fila por principio:

| Principio | Cumple / No aplica / Excepción | Nota |
|---|---|---|
| I. Spec primero | | |
| II. Monolito modular | | |
| III. Simplicidad para el administrador | | |
| IV. La IA sugiere y una persona confirma | | |
| V. Dinero, fechas y periodos | | |
| VI. Trazabilidad e inmutabilidad | | |
| VII. Pruebas | | |
| VIII. Seguridad y privacidad | | |
| IX. Integraciones detrás de puertos | | |
| X. Operación | | |

- [ ] No queda ningún `[POR DEFINIR]`, y cada regla tiene sus casos de borde cerrados.
- [ ] Ningún principio queda vacío.
- [ ] Toda **excepción** está justificada. Si la excepción es permanente, primero se modifica la constitución con un ADR.
- [ ] Los contratos y eventos usan los nombres del glosario.

### C4. Consistencia *(equivalente ligero de `analyze`)*

Antes de implementar:

- [ ] Se revisó la spec buscando **comportamientos sin definir**. Los que aparezcan se proponen y se escriben en los documentos antes de seguir.
- [ ] Cada requisito tiene al menos una tarea que lo implementa.
- [ ] Cada escenario tiene una tarea de **prueba**, que va **antes** de la tarea que la pone en verde. Las excepciones a "pruebas primero" están declaradas con su motivo.
- [ ] Si el cambio toca contratos, **la primera tarea** actualiza `.Contracts` y el mapa de módulos.
- [ ] No hay tareas que no correspondan a ningún requisito.
- [ ] Las tareas están agrupadas en **puntos de control** (ver abajo).

### C5. Convergencia *(equivalente ligero de `converge`)*

Después de implementar, con `/opsx:verify` y revisión propia:

- [ ] Todas las tareas están marcadas.
- [ ] Todas las pruebas pasan en CI, incluidas las de arquitectura.
- [ ] Las pruebas de mutación no dejaron mutantes vivos sin explicación en el código que cambió.
- [ ] El comportamiento implementado coincide con los escenarios. Si hay diferencias, **se corrige el código o se corrige el delta**, nunca se deja la diferencia.

Si aparecen huecos, se agregan como tareas nuevas en una sección **"Convergencia"** de `tasks.md` y se vuelve a implementar.

### C6. Listo para archivar

- [ ] C5 pasó.
- [ ] Los deltas describen el comportamiento **final**, no el planeado originalmente.
- [ ] El mapa de módulos, el glosario y los ADRs están actualizados, si aplica.
- [ ] El archivo (`/opsx:archive`) va **en el mismo PR** que cierra el cambio.

---

## Puntos de control, ramas y commits

- Las tareas de `tasks.md` se agrupan en **puntos de control** de pocas tareas (orientativo: 3 a 6), cada uno con un resultado verificable.
- **Una rama y un PR por punto de control.** Nombre de rama en inglés: `<change>/<checkpoint>`, por ejemplo `register-bill-manual/cp1-contracts`.
- **Un commit por tarea**, en inglés, con el ID cuando aplique: `test(utilities): UTL-R01-S1 bill with future due date`.
- El commit de una **prueba en rojo se queda local** y se sube junto con el commit que la pone en verde, para que el CI del PR no esté en rojo.
- **Tras cada prueba en verde,** se rompe el código a propósito para confirmar que falla la prueba correcta. Al cerrar el punto de control, las pruebas de mutación (Stryker) lo verifican de forma sistemática.

---

## Vía ligera

Para cambios pequeños, **spec, plan y tareas van en un solo cambio y un solo PR**, sin `design.md` (la revisión de la constitución se resume en una línea dentro de `proposal.md`).

Se puede usar **solo si se cumplen todas** estas condiciones:

- [ ] No toca dinero, confirmaciones ni prorrateos.
- [ ] No agrega ni modifica contratos o eventos entre módulos.
- [ ] Toca un solo módulo.
- [ ] Tiene como máximo 5 tareas.
- [ ] No agrega pantallas nuevas (puede modificar una existente).

En la vía ligera se aplican C2 y C5. Los IDs, las pruebas primero y el archivo en el mismo PR siguen siendo obligatorios.

---

## Continuidad entre sesiones

- **Al empezar** una sesión, se lee `docs/handoff.md`.
- **Al terminar**, se actualiza con: estado actual, decisiones tomadas, siguiente paso, notas para retomar y un prompt sugerido para el siguiente chat.
- Se recomienda **un chat nuevo por punto de control**, con el handoff como puente.
- **Al cerrar un hito,** se escribe una retrospectiva en `docs/retros/AAAA-MM-DD-<hito>.md` con hallazgos numerados (`H1`, `H2`…) y su estado: aplicado o pendiente.
