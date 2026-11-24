# Handoff

> **Última actualización:** 2026-11-23

## Estado actual

Fase de diseño. No hay código todavía.

**Documentos listos:**
- Constitución v0.10.1 (borrador), con las prácticas de gym-app incorporadas: jerarquía de documentos, ADRs inmutables, `[POR DEFINIR]`, IDs estables, pruebas primero, mutación, contratos antes que código y retrospectivas.
- Mapa de módulos v0.4.1, con identificadores en inglés.
- Glosario español → código v0.1.0.
- Flujo de especificaciones v0.2.0: OpenSpec con controles C1 a C6, IDs, puntos de control, vía ligera y continuidad.
- `CLAUDE.md` con las instrucciones para el agente.

**ADRs aceptados:**
- 0001: backend en .NET 10 con ASP.NET Core.
- 0002: frontend PWA con React, Vite y TypeScript.
- 0003: especificaciones con OpenSpec y controles propios.
- 0004: el código va en inglés.
- 0005: bus de eventos en proceso con Wolverine y *outbox* en PostgreSQL.

## Decisiones tomadas en la última sesión

- **Bus de eventos: Wolverine** (MIT, `WolverineFx` 6.x, compilado para `net10.0`), con *outbox* e *inbox* durables en PostgreSQL. Motivo: el Principio IX.3 exige reintentos, así que un bus solo en memoria no basta.
- Wolverine vive solo en `Infrastructure` y `Host`. Los módulos publican con `IEventPublisher` y sus *handlers* son clases simples, sin tipos de Wolverine. Se agregó la regla de dependencia 6 al mapa de módulos.
- Descubrimiento de *handlers* explícito por módulo y entrega asíncrona por colas locales durables.
- Un solo esquema compartido, `wolverine`, para las tablas de mensajes (en lugar de un *outbox* por módulo).
- Alternativas descartadas: despachador propio con *outbox* (queda como plan de salida), despachador solo en memoria, MassTransit, MediatR, Mediator y Brighter.

## Siguiente paso

Decidir los ADRs pendientes, en este orden:

1. **Proveedor de IA** para la extracción de recibos.
2. **Hosting y entorno local** (.NET Aspire es candidato).

Después: armar el esqueleto (solución, `openspec init`, CI) y escribir el primer cambio, `register-bill-manual`.

## Notas para retomar

- Los números de ADR se asignan **al crear** el ADR, no antes. La constitución ya menciona números tentativos (IA = 0006, hosting = 0007); si se crean en otro orden, se corrigen esas referencias.
- La carpeta vacía `docs/specs/` ya no se usa; hay que quitarla del repositorio.
- Al instalar OpenSpec, revisar en openspec.dev/docs qué permiten su configuración de proyecto y sus *schemas*, para convertir los controles C1 a C6 en plantillas.
- **`[POR DEFINIR]` del ADR-0005:** número de reintentos y tiempos de espera por consumidor; si los trabajos programados usan los mensajes programados de Wolverine o una librería aparte.
- **Bug de documentación pendiente:** la Constitución II.7 nombra el núcleo compartido `Dinero`, `Periodo` y `Resultado/Error`; el mapa y el glosario usan `Money`, `DateRange` y `Result` / `Error`. Falta decidir cómo se corrige.
- Al armar el esqueleto: `IEventPublisher`, configuración de Wolverine, prueba de arquitectura (regla 6) y prueba de integración del flujo publicar → consumir con Testcontainers.

## Prompt sugerido para el siguiente chat

> Lee `CLAUDE.md` y `docs/handoff.md`. Seguimos con el ADR del proveedor de IA para la extracción de recibos: compárame las opciones y pregúntame cuál elegir.
