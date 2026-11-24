# Handoff

> **Última actualización:** 2026-11-23

## Estado actual

Fase de diseño terminada. No hay código todavía.

**Documentos:** constitución v0.11.0, mapa de módulos v0.4.3, glosario v0.1.0 y flujo de especificaciones v0.3.0.

**ADRs:**
- Aceptados: 0001 (.NET 10), 0002 (PWA con React), 0003 (OpenSpec), 0004 (código en inglés), 0005 (Wolverine), 0007 (hosting) y 0008 (trunk-based development).
- **Propuesto: 0006** (extracción con Claude). No tiene `[POR DEFINIR]`; espera la prueba con recibos reales.

**Git:** todo en `main` con commit. `main` local va 7 commits adelante de `origin/main`, **sin push**. La rama `docs/adr-0005-event-bus` ya está incluida en `main` y se puede borrar.

## Decisiones tomadas en la última sesión

- **Gobernanza:** un ADR nace *Propuesto* y solo se acepta sin `[POR DEFINIR]` y con la lista *Listo para aceptar* (ECADR) de la plantilla completa. Los ADR-0005 a 0008 se regresaron a *Propuesto*, se cerraron sus pendientes y se aceptaron, salvo el 0006.
- **ADR-0005:** trabajos programados con los trabajos recurrentes de Wolverine, diseñados por estado; reintentos en dos fases para fallas transitorias y ninguno para las permanentes.
- **ADR-0006:** Claude a través de `IChatClient`, con el adaptador en `AdminRentas.Ingestion.Adapters`. Las fotos se normalizan en la PWA y en el servidor, a 2,576 px en JPEG.
- **ADR-0007:** Hetzner CX23 con Docker Compose, Cloudflare Tunnel en `<subdominio>.saulgmx.com` y Tailscale. Infraestructura con OpenTofu y Ansible. Despliegue por etiqueta desde GitHub Actions; respaldos cifrados en R2; Aspire en local.
- **ADR-0008:** ramas de uno o dos días, integración con *merge* y sin rebasear ramas subidas; la interfaz va al final e interruptores solo para comportamiento visible.
- **Bug resuelto:** el límite de documento es el de la constitución (10 MB); el ADR-0006 ya lo dice.

## Siguiente paso

**Armar el esqueleto**, con ramas cortas y PRs (ADR-0008; desde aquí ya no se trabaja directo en `main`):
- solución .NET 10 con los proyectos por módulo y `openspec init`;
- *AppHost* de Aspire, `Dockerfile` y `docker-compose.yml` de producción;
- modo "solo migrar" y verificación de salud;
- `IEventPublisher`, configuración de Wolverine (reintentos y trabajos recurrentes) y prueba de integración publicar → consumir con Testcontainers;
- pruebas de arquitectura de las reglas 1 a 8;
- infraestructura: módulos de OpenTofu, `cloud-init` y *playbooks* de Ansible;
- GitHub Actions: CI, despliegue y prueba de restauración semanal; protección de `main` y solo integración con *merge*.

Después: el primer cambio, `register-bill-manual`.

## Notas para retomar

- **Antes de usar la abstracción para declarar trabajos programados** (ADR-0005), proponer su nombre en el glosario.
- **Al instalar OpenSpec,** revisar en openspec.dev/docs qué permiten su configuración de proyecto y sus *schemas*, para convertir los controles C1 a C6 en plantillas.
- **Al crear el servidor:** confirmar en la consola de Hetzner que hay CX23 en Núremberg (si no, Helsinki; si tampoco, DigitalOcean). Crear la red de Tailscale con la **cuenta personal de Gmail o GitHub**, no con una de `saulgmx.com`.
- **Antes del cambio de `Ingestion` con extracción:** hacer la prueba comparativa del ADR-0006 (unos 10 recibos reales con Claude Haiku 5.5 y Gemini 3.5 Flash-Lite, y una llave de API de cada proveedor) y aceptar el ADR.

## Prompt sugerido para el siguiente chat

> Lee `CLAUDE.md` y `docs/handoff.md`. Empezamos a armar el esqueleto: propón las tareas y los puntos de control, y pregúntame antes de empezar.
