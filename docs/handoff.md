# Handoff

> **Última actualización:** 2026-11-24
>
> Solo contiene el estado y lo que necesita la siguiente tarea. Los pendientes viven como `[POR DEFINIR]` en su documento (búscalos con `grep -rn "POR DEFINIR" docs`).

## Estado actual

Fase de diseño. No hay código todavía.

- **Documentos:** constitución v0.12.0, mapa de módulos v0.4.4, glosario v0.1.0 y flujo de especificaciones v0.4.0.
- **ADRs:** todos aceptados, salvo el 0006 (propuesto), que espera la prueba con recibos reales.
- **Git:** `main` va 3 commits adelante de `origin/main`, **sin push**. La historia se reescribe antes de publicarlos.

## Siguiente tarea

**Cerrar la limpieza de seguridad del repositorio** antes del esqueleto:

1. Reescribir la historia con `git filter-repo` ([ADR-0009](adr/0009-subdominio-fuera-del-repositorio.md), punto 8). Se hace sobre una copia, se verifica que no queden datos sensibles y se **confirma con el dueño antes del *force push***.

**Después: armar el esqueleto.** Empieza con `openspec init` y la creación de su cambio en OpenSpec. Su `tasks.md` se arma con:
- las *Decisiones derivadas* de los ADRs;
- la sección "Se decide al armar el esqueleto" del [ADR-0002](adr/0002-frontend-react-vite.md);
- los `[POR DEFINIR]` de los documentos;
- la protección de GitHub para un repositorio público, que todavía no vive en ningún documento: un *ruleset* para las etiquetas `v*`, un *environment* `production` con los secretos de despliegue, `GITHUB_TOKEN` en solo lectura, *actions* fijadas por SHA, Dependabot y logs de la prueba de restauración sin datos.

## Prompt sugerido para el siguiente chat

> Lee `CLAUDE.md` y `docs/handoff.md`. Cerremos la limpieza de seguridad del repositorio, una tarea a la vez.
