# Handoff

> **Última actualización:** 2026-11-24
>
> Solo contiene el estado y lo que necesita la siguiente tarea. Los pendientes viven como `[POR DEFINIR]` en su documento (búscalos con `grep -rn "POR DEFINIR" docs`).

## Estado actual

Fase de diseño. No hay código todavía.

- **Documentos:** constitución v0.12.0, mapa de módulos v0.4.4, glosario v0.1.0 y flujo de especificaciones v0.4.0.
- **ADRs:** todos aceptados, salvo el 0006 (propuesto), que espera la prueba con recibos reales.
- **Git:** todo en `main`. La historia publicada se reescribió el 2026-11-24 para quitar datos sensibles ([ADR-0009](adr/0009-subdominio-fuera-del-repositorio.md), punto 8): cualquier clon anterior se vuelve a clonar.

## Siguiente tarea

**Armar el esqueleto**, con ramas cortas y PRs ([ADR-0008](adr/0008-trunk-based-development.md)). Empieza con `openspec init` y la creación de su cambio en OpenSpec. Su `tasks.md` se arma con:
- las *Decisiones derivadas* de los ADRs;
- la sección "Se decide al armar el esqueleto" del [ADR-0002](adr/0002-frontend-react-vite.md);
- los `[POR DEFINIR]` de los documentos;
- la protección de GitHub para un repositorio público, que todavía no vive en ningún documento: un *ruleset* para las etiquetas `v*`, un *environment* `production` con los secretos de despliegue, `GITHUB_TOKEN` en solo lectura, *actions* fijadas por SHA, Dependabot y logs de la prueba de restauración sin datos.

## Prompt sugerido para el siguiente chat

> Lee `CLAUDE.md` y `docs/handoff.md`. Empezamos a armar el esqueleto: propón las tareas y los puntos de control, y pregúntame antes de empezar.
