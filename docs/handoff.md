# Handoff

> **Última actualización:** 2026-11-23

## Estado actual

Fase de diseño. No hay código todavía.

**Documentos listos:**
- Constitución v0.10.3 (borrador), con las prácticas de gym-app incorporadas: jerarquía de documentos, ADRs inmutables, `[POR DEFINIR]`, IDs estables, pruebas primero, mutación, contratos antes que código y retrospectivas.
- Mapa de módulos v0.4.2, con identificadores en inglés.
- Glosario español → código v0.1.0.
- Flujo de especificaciones v0.2.0: OpenSpec con controles C1 a C6, IDs, puntos de control, vía ligera y continuidad.
- `CLAUDE.md` con las instrucciones para el agente.

**ADRs aceptados:**
- 0001: backend en .NET 10 con ASP.NET Core.
- 0002: frontend PWA con React, Vite y TypeScript.
- 0003: especificaciones con OpenSpec y controles propios.
- 0004: el código va en inglés.
- 0005: bus de eventos en proceso con Wolverine y *outbox* en PostgreSQL.
- 0006: extracción de recibos con Claude a través de `Microsoft.Extensions.AI`.

**Sin commit:** el ADR-0006 y los cambios a la constitución, el mapa de módulos y este handoff están en `main`, pendientes de commit. Por acuerdo de la sesión del 2026-11-23, la fase de diseño se trabaja directo en `main`; la regla de una rama por punto de control (flujo de especificaciones) sigue vigente para los cambios con código. `main` local va 2 commits adelante de `origin/main` (ADR-0005), sin push.

## Decisiones tomadas en la última sesión

- **Proveedor de IA: Claude**, con **Haiku 5.5** como modelo principal y **Sonnet 5.5** como reintento cuando la extracción sale con advertencias. Los identificadores de modelo van en la configuración.
- **Capa: `Microsoft.Extensions.AI` (`IChatClient`)** con el SDK oficial `Anthropic`, para poder cambiar de proveedor después. Se verificó en el código del SDK (13.13.0) que el PDF y las fotos viajan como bloques nativos y que el esquema JSON se traduce a la salida estructurada nativa.
- **Dos capas de independencia:** `IDocumentExtractor`, el puerto del dominio, y `IChatClient`, la interfaz del modelo.
- **Adaptador en un proyecto aparte, `AdminRentas.Ingestion.Adapters`,** referenciado solo por `Host`. Es el **patrón para todos los adaptadores externos** (`AdminRentas.<Module>.Adapters`). Se agregaron las reglas de dependencia 7 y 8 al mapa de módulos.
- **Verificación contra el texto del documento solo en PDFs con texto** (candidata: PdfPig). Tesseract para verificar fotos queda como evolución prevista.
- **Privacidad:** el documento va dentro de la petición, sin la API de archivos del proveedor; logs y telemetría sin contenido.
- **Alternativas descartadas:**
  - Gemini, que queda como candidata en la prueba comparativa;
  - OpenAI;
  - OCR especializado en la nube (Azure, AWS);
  - modelo local;
  - Tesseract como lector principal;
  - SDK de Anthropic directo, sin `Microsoft.Extensions.AI`;
  - adaptador dentro de `AdminRentas.Ingestion`;
  - Microsoft Agent Framework (pensado para agentes; la extracción es una sola llamada).

## Siguiente paso

1. **ADR de hosting y entorno local** (.NET Aspire es candidato). Será el ADR-0007, si se crea en ese orden.
2. Después: armar el esqueleto (solución, `openspec init`, CI) y escribir el primer cambio, `register-bill-manual`.

## Notas para retomar

- Los números de ADR se asignan **al crear** el ADR. La constitución menciona hosting como ADR-0007 de forma tentativa.
- La carpeta vacía `docs/specs/` ya no se usa; hay que quitarla del repositorio.
- Al instalar OpenSpec, revisar en openspec.dev/docs qué permiten su configuración de proyecto y sus *schemas*, para convertir los controles C1 a C6 en plantillas.
- **`[POR DEFINIR]` del ADR-0005:** número de reintentos y tiempos de espera por consumidor; si los trabajos programados usan los mensajes programados de Wolverine o una librería aparte.
- **Pendientes del ADR-0006:**
  - **Prueba comparativa** Claude Haiku 5.5 contra Gemini 3.5 Flash-Lite con unos 10 recibos reales (foto y PDF), campo por campo. Es un script desechable fuera del repositorio y necesita una llave de API de cada proveedor. Si Gemini gana claramente, se escribe un ADR nuevo.
  - `[POR DEFINIR]` **cálculo de la confianza** y qué cuenta como "baja confianza" (IV.1). Se define en la spec de `Ingestion`.
  - `[POR DEFINIR]` **fotos HEIC:** rechazarlas o convertirlas a JPEG. No se verificó si Claude las acepta directamente.
  - `[POR DEFINIR]` **límites de tamaño** de los documentos.
- Al armar el esqueleto:
  - `IEventPublisher`, configuración de Wolverine, pruebas de arquitectura (reglas 6, 7 y 8) y prueba de integración del flujo publicar → consumir con Testcontainers;
  - proyecto `AdminRentas.Ingestion.Adapters` cuando llegue el primer cambio con extracción.

## Prompt sugerido para el siguiente chat

> Lee `CLAUDE.md` y `docs/handoff.md`. Seguimos con el ADR de hosting y entorno local: compárame las opciones (incluido .NET Aspire) y pregúntame cuál elegir.
