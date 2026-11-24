# Handoff

> **Última actualización:** 2026-11-23

## Estado actual

Fase de diseño terminada en lo que toca a las decisiones de arquitectura. No hay código todavía.

**Documentos listos:**
- Constitución v0.10.4 (borrador), con las prácticas de gym-app incorporadas: jerarquía de documentos, ADRs inmutables, `[POR DEFINIR]`, IDs estables, pruebas primero, mutación, contratos antes que código y retrospectivas.
- Mapa de módulos v0.4.2, con identificadores en inglés, la convención `AdminRentas.<Module>.Adapters` y las reglas de dependencia 1 a 8.
- Glosario español → código v0.1.0.
- Flujo de especificaciones v0.3.0: OpenSpec con controles C1 a C6, IDs, puntos de control, vía ligera, continuidad y trunk-based development.
- `CLAUDE.md` con las instrucciones para el agente.

**ADRs aceptados:**
- 0001: backend en .NET 10 con ASP.NET Core.
- 0002: frontend PWA con React, Vite y TypeScript.
- 0003: especificaciones con OpenSpec y controles propios.
- 0004: el código va en inglés.
- 0005: bus de eventos en proceso con Wolverine y *outbox* en PostgreSQL.
- 0006: extracción de recibos con Claude a través de `Microsoft.Extensions.AI`.
- 0007: hosting en un VPS con Docker Compose y Cloudflare Tunnel, y entorno local con Aspire.
- 0008: trunk-based development con ramas cortas e interruptores de funciones solo cuando hacen falta.

**Git:** todo está en `main` con commit. `main` local va varios commits adelante de `origin/main` (ADR-0005 a ADR-0008), **sin push**. La rama `docs/adr-0005-event-bus` sigue existiendo y apunta a un commit ya incluido en `main`; se puede borrar.

## Decisiones tomadas en la última sesión

**ADR-0006, extracción con IA:**
- **Claude:** Haiku 5.5 como modelo principal y Sonnet 5.5 como reintento cuando hay advertencias, con los modelos en la configuración.
- **`IChatClient`** de `Microsoft.Extensions.AI`, con el SDK oficial `Anthropic`, detrás de `IDocumentExtractor`. Se verificó en el código del SDK (13.13.0) que el PDF y las fotos viajan como bloques nativos y que el esquema JSON se traduce a la salida estructurada nativa.
- **Adaptador en `AdminRentas.Ingestion.Adapters`,** referenciado solo por `Host`. Es el patrón para todos los adaptadores externos (reglas de dependencia 7 y 8).
- **Verificación contra el texto del documento** solo en PDFs con texto (candidata: PdfPig). Tesseract para fotos queda como evolución prevista.
- Descartados: Gemini (queda en la prueba comparativa), OpenAI, OCR en la nube, modelo local, Tesseract como lector principal, SDK directo y Microsoft Agent Framework.

**ADR-0007, hosting:**
- **Servidor:** Hetzner CX23 en Europa (Núremberg, o Helsinki si no hay disponibilidad), unos €6 al mes, con Ubuntu Server LTS. Plan B: DigitalOcean de 2 GB en EE. UU.
- **Servicios:** Docker Compose de producción escrito a mano, con `api`, `postgres` y `cloudflared`, sin puertos publicados.
- **Acceso de los usuarios:** Cloudflare Tunnel en un subdominio de `saulgmx.com`.
- **Administración:** Tailscale, con la red creada con una **cuenta personal de Gmail o GitHub**, no de dominio propio, para quedar en el plan Personal gratuito.
- **Entorno local:** Aspire. El Compose de producción no se genera con Aspire.
- **Despliegue:** GitHub Actions al crear una etiqueta `v*`, más un botón manual para volver a desplegar una etiqueta. Pasos: respaldo previo, paso "solo migrar" y después la API; aviso por Telegram si algo falla.
- **Migraciones:** compatibles con la versión anterior ("expandir y contraer").
- **Respaldos:** diarios, cifrados con `age`, en Cloudflare R2, con retención de 7 diarios, 4 semanales y 6 mensuales. Prueba de restauración automática cada semana.
- **Monitoreo:** UptimeRobot (por el camino público, alertas por correo) y Healthchecks.io (trabajos y respaldos, alertas por Telegram). Logs en Docker con rotación de tamaño.
- **Descartados:** Supabase Pro, plataformas administradas, nubes grandes, Oracle gratis, equipo en casa, Caddy directo, Cloudflare como proxy, app solo por Tailscale, Kamal, Coolify, entre otros.

**ADR-0008, proceso:**
- **Ramas:** de uno o dos días, una por punto de control y en secuencia. Todo entra por PR con la CI en verde y la rama al día con `main`.
- **Integración con *merge*,** con commit de integración. Una rama ya subida no se rebasea ni se fuerza; se pone al día trayendo `main` con un *merge*. Se consideraron y descartaron *rebase* + *squash* y *rebase* en todo.
- **Migraciones en conflicto:** se vuelven a generar, no se mezclan a mano.
- **Funciones a medias:** la interfaz va al final. Solo los cambios que modifican comportamiento visible llevan interruptor (`Microsoft.FeatureManagement`), y se borra al cerrar el cambio.

**Bug de documentación del ADR-0006:** el ADR dice "límites de tamaño `[POR DEFINIR]`", pero la constitución (IV.3) fija 10 MB por documento. Se decidió **no editar el ADR**: manda la constitución. El límite propio del proveedor de IA para imágenes (y si hay que reducir las fotos) queda para la spec de `Ingestion`.

**Excepción de esta sesión:** a petición del dueño, la fase de diseño se trabajó directo en `main`, sin ramas ni PRs. La regla de ramas del flujo de especificaciones **sigue vigente** y aplica desde el esqueleto.

## Siguiente paso

**Armar el esqueleto**, con ramas cortas y PRs según el ADR-0008:
- solución .NET 10 con los proyectos por módulo;
- `openspec init`;
- *AppHost* de Aspire;
- `Dockerfile` y `docker-compose.yml` de producción;
- modo "solo migrar" y verificación de salud;
- CI, despliegue y prueba de restauración en GitHub Actions;
- protección de `main` en GitHub.

Después: escribir el primer cambio, `register-bill-manual`.

## Notas para retomar

- La carpeta vacía `docs/specs/` ya no se usa; hay que quitarla del repositorio.
- Al instalar OpenSpec, revisar en openspec.dev/docs qué permiten su configuración de proyecto y sus *schemas*, para convertir los controles C1 a C6 en plantillas.
- **Pendientes del ADR-0005:**
  - `[POR DEFINIR]` número de reintentos y tiempos de espera por consumidor;
  - `[POR DEFINIR]` si los trabajos programados usan los mensajes programados de Wolverine o una librería aparte.
- **Pendientes del ADR-0006:**
  - **prueba comparativa** de Claude Haiku 5.5 contra Gemini 3.5 Flash-Lite con unos 10 recibos reales: script desechable fuera del repositorio, con una llave de API de cada proveedor;
  - `[POR DEFINIR]` cálculo de la confianza y qué cuenta como "baja confianza" (IV.1), en la spec de `Ingestion`;
  - `[POR DEFINIR]` fotos HEIC: rechazarlas o convertirlas a JPEG;
  - límite del proveedor de IA para imágenes, en la spec de `Ingestion`.
- **Pendientes del ADR-0007:**
  - `[POR DEFINIR]` preparación del servidor como código (candidata: `cloud-init`);
  - `[POR DEFINIR]` nombre del subdominio;
  - verificar Telegram en los planes gratuitos de Healthchecks.io y UptimeRobot (si no, *webhook* a la API de bots de Telegram), la disponibilidad del CX23 y el soporte de Aspire 17 para .NET 10.
- **Al armar el esqueleto:**
  - `IEventPublisher`, configuración de Wolverine y prueba de integración del flujo publicar → consumir con Testcontainers;
  - pruebas de arquitectura de las reglas 6, 7 y 8;
  - proyecto `AdminRentas.Ingestion.Adapters` cuando llegue el primer cambio con extracción;
  - en GitHub, proteger `main` (solo por PR, CI en verde, rama al día) y dejar habilitada solo la integración con *merge*.

## Prompt sugerido para el siguiente chat

> Lee `CLAUDE.md` y `docs/handoff.md`. Empezamos a armar el esqueleto: propón las tareas y los puntos de control, y pregúntame antes de empezar.
