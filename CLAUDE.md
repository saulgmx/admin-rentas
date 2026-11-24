# admin-rentas — instrucciones para el agente

App personal para que el dueño (a distancia) y el administrador (en la ciudad de la propiedad) estén sincronizados sobre un departamento en renta mixta: servicios, pagos, ocupación y mantenimiento. Monolito modular en .NET 10 + PWA en React, con desarrollo guiado por especificaciones (OpenSpec).

## Al empezar cada sesión

1. Lee `docs/handoff.md` para saber dónde quedamos.
2. Si vas a trabajar en un cambio, lee su carpeta en `openspec/changes/<change>/`.
3. Ten a la mano `docs/constitution.md` y `docs/arquitectura/glosario.md`.

## Forma de trabajo

- **Paso a paso:** una tarea a la vez. Al terminar cada tarea, muestra el resultado y **espera mi confirmación** antes de seguir.
- **No decidas en el código.** Si la spec no define algo, propónlo y pregúntame. Mientras no se resuelva, márcalo como `[POR DEFINIR]` en el documento correspondiente.
- **Una pregunta a la vez**, con opciones cuando sea posible.
- **Idiomas:** la conversación y los documentos en español; el código, los identificadores, los commits, las ramas y los PRs en inglés. La traducción oficial está en el glosario; si falta un concepto, propón su traducción antes de usarlo.

## Autoridad de los documentos

De mayor a menor: **constitución > spec > ADRs > `design.md` > código**.

- Si dos documentos se contradicen, es un **bug de documentación**: avísame y se corrige en el documento de menor autoridad. No lo resuelvas en el código.
- **Un ADR aceptado no se edita.** Si una decisión cambia, se escribe un ADR nuevo que lo reemplaza.

## Reglas que más se olvidan

- **Contratos antes que código:** si un cambio toca `.Contracts` o eventos, la primera tarea actualiza los contratos y `docs/arquitectura/mapa-modulos.md`.
- **Pruebas primero** para todo comportamiento con ID. El nombre de la prueba cita el ID del escenario (`UTL_R01_S1_...`). Cada bug empieza con una prueba que lo reproduce.
- **Tras cada prueba en verde,** rompe el código a propósito para confirmar que falla la prueba correcta.
- **Un commit por tarea**, en inglés. Un commit con una prueba en rojo se queda local hasta que la tarea la pone en verde.
- **Dinero:** `decimal` / `numeric(12,2)`, nunca `double` ni `float`. **Fechas sin hora:** `DateOnly`, en la zona horaria de la propiedad , que es un dato de cada propiedad.
- **Límites entre módulos:** una implementación solo referencia los proyectos `.Contracts` de otros módulos, nunca su implementación.
- **No hagas commits ni push** si no te lo pido.

## Al terminar cada sesión

Actualiza `docs/handoff.md` con: estado actual, decisiones tomadas, siguiente paso, notas para retomar y un prompt sugerido para el siguiente chat.

## Documentos

| Documento | Para qué |
|---|---|
| `docs/constitution.md` | Reglas que no se negocian. |
| `docs/proceso/flujo-especificaciones.md` | Flujo con OpenSpec, controles C1 a C6, IDs, puntos de control y vía ligera. |
| `docs/arquitectura/mapa-modulos.md` | Módulos, contratos, eventos y reglas de dependencia. |
| `docs/arquitectura/glosario.md` | Español → código. |
| `docs/adr/` | Decisiones de arquitectura. |
| `docs/handoff.md` | Dónde quedamos. |
| `openspec/` | Specs vivas y cambios en curso. |
