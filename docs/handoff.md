# Handoff

> **Última actualización:** 2026-11-23

## Estado actual

Fase de diseño. No hay código todavía.

**Documentos listos:**
- Constitución v0.10.0 (borrador), con las prácticas de gym-app incorporadas: jerarquía de documentos, ADRs inmutables, `[POR DEFINIR]`, IDs estables, pruebas primero, mutación, contratos antes que código y retrospectivas.
- Mapa de módulos v0.4.0, con identificadores en inglés.
- Glosario español → código v0.1.0.
- Flujo de especificaciones v0.2.0: OpenSpec con controles C1 a C6, IDs, puntos de control, vía ligera y continuidad.
- `CLAUDE.md` con las instrucciones para el agente.

**ADRs aceptados:**
- 0001: backend en .NET 10 con ASP.NET Core.
- 0002: frontend PWA con React, Vite y TypeScript.
- 0003: especificaciones con OpenSpec y controles propios.
- 0004: el código va en inglés.

## Decisiones tomadas en la última sesión

- Renta mixta; el administrador cobra por evento; el dueño reembolsa gastos.
- Primera versión enfocada en servicios y pagos.
- Avisos por Telegram y correo; WhatsApp solo con enlaces `wa.me`.
- La IA extrae y una persona confirma: el administrador confirma recibos de menos de $1,500 MXN (configurable); el dueño, los demás y los de baja confianza.
- Prorrateo por días; por consumo opcional, con lecturas de medidor.
- Sin almacenamiento permanente de archivos; el original se retiene hasta confirmar, máximo 7 días, en PostgreSQL.
- Las lecturas de medidor viven en `Utilities`.

## Siguiente paso

Decidir los ADRs pendientes, en este orden:

1. **Bus de eventos en proceso** (Wolverine, despachador propio u otro).
2. **Proveedor de IA** para la extracción de recibos.
3. **Hosting y entorno local** (.NET Aspire es candidato).

Después: armar el esqueleto (solución, `openspec init`, CI) y escribir el primer cambio, `register-bill-manual`.

## Notas para retomar

- Los números de ADR se asignan **al crear** el ADR, no antes.
- La carpeta vacía `docs/specs/` ya no se usa; hay que quitarla del repositorio.
- Al instalar OpenSpec, revisar en openspec.dev/docs qué permiten su configuración de proyecto y sus *schemas*, para convertir los controles C1 a C6 en plantillas.

## Prompt sugerido para el siguiente chat

> Lee `CLAUDE.md` y `docs/handoff.md`. Seguimos con el ADR del bus de eventos en proceso: compárame las opciones para un monolito modular en .NET 10 y pregúntame cuál elegir.
