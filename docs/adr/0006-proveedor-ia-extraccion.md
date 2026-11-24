# ADR-0006: Extracción de recibos con Claude a través de `Microsoft.Extensions.AI`

- **Estado:** Aceptado
- **Fecha:** 2026-11-23
- **Principios relacionados:** IV (la IA sugiere y una persona confirma), VIII (seguridad y privacidad), IX (integraciones externas)

> Un ADR aceptado **no se edita**. Si la decisión cambia, se escribe uno nuevo que lo reemplaza y aquí solo se actualiza el estado.

## Contexto

El módulo `Ingestion` recibe recibos (fotos tomadas con el celular y PDFs) y extrae sus datos con IA a través del puerto `IDocumentExtractor` (mapa de módulos). Hay que decidir qué proveedor y qué tecnología implementan ese puerto.

Restricciones:

- **Entrada:** fotos y PDFs de recibos mexicanos (CFE, agua municipal, internet, gas, predial, condominio), con formatos y tablas propios de cada proveedor.
- **Salida:** campos genéricos de recibo (proveedor, número de cuenta, periodo, consumo, monto, fecha límite) más **confianza o advertencias**, que deciden quién confirma (Principio IV.1).
- **Auditoría:** se guardan el proveedor y el modelo usados (Principio IV.4).
- **Independencia del proveedor:** el proveedor queda detrás de una interfaz y cambiarlo no toca el dominio (Principios IV.5 y IX.1), con una implementación falsa para pruebas (IX.2).
- **Privacidad:** el documento contiene domicilio y número de cuenta; nunca se escribe en disco ni en los logs (VIII.5), y las llaves se inyectan como secretos (VIII.4).
- **Volumen muy bajo:** menos de 15 recibos al mes. El costo por recibo no es un criterio decisivo.
- El hosting todavía no está decidido (ADR-0007), así que conviene no amarrarse a una nube.

## Decisión

La extracción se hace con un **modelo de lenguaje con visión de Claude** (Anthropic), llamado a través de la interfaz **`IChatClient` de `Microsoft.Extensions.AI`** con la implementación del SDK oficial de Anthropic para C#. El modelo principal es **Claude Haiku 5.5**, con **Claude Sonnet 5.5** como reintento cuando la extracción sale con advertencias.

### Detalle

1. **Dos capas de independencia:**
   - `IDocumentExtractor` es el puerto **del dominio**: recibe el documento y devuelve los campos extraídos, las advertencias y el modelo usado. No expone tipos de `Microsoft.Extensions.AI` ni de Anthropic.
   - `IChatClient` es la interfaz **del modelo**: el adaptador de `IDocumentExtractor` la usa para hablar con el proveedor. Cambiar de proveedor (por ejemplo, a Gemini) es registrar otra implementación de `IChatClient` y ajustar el *prompt*, sin tocar el dominio ni el puerto.
2. **Dónde vive:** en un proyecto aparte, **`AdminRentas.Ingestion.Adapters`**.
   - El puerto `IDocumentExtractor` se define en `AdminRentas.Ingestion` (público dentro del módulo, pero no en `.Contracts`, porque ningún otro módulo lo usa).
   - `AdminRentas.Ingestion.Adapters` referencia a `AdminRentas.Ingestion` e implementa el puerto con una clase `internal`. Expone solo su método de registro.
   - Solo `Host` referencia `AdminRentas.Ingestion.Adapters` y registra el adaptador.
   - Solo `AdminRentas.Ingestion.Adapters` y `Host` referencian los paquetes `Anthropic` y `Microsoft.Extensions.AI`. Una prueba de arquitectura verifica que ningún otro proyecto (incluidos `AdminRentas.Ingestion` y `AdminRentas.Ingestion.Contracts`) use esos espacios de nombres.
   - Este arreglo es el **patrón para los adaptadores externos de todos los módulos** (Telegram, correo y los que vengan): proyecto `AdminRentas.<Module>.Adapters`, referenciado solo por `Host`.
3. **Cliente:** se obtiene con `AnthropicClient.AsIChatClient(modelo)`, del paquete `Anthropic`. El código del SDK (versión 13.13.0) confirma que:
   - un `DataContent` con tipo `application/pdf` se envía como **bloque de documento nativo**, y uno con tipo `image/*` como bloque de imagen nativo;
   - `ChatResponseFormat.ForJsonSchema(...)` se traduce a la **salida estructurada nativa** de Claude, no a una emulación con *prompts*;
   - la respuesta trae `ModelId` y el uso de tokens, que se guardan para la auditoría (IV.4);
   - si hace falta una opción propia de Anthropic que la interfaz común no cubre, `ChatOptions.RawRepresentationFactory` permite pasarla.
4. **Modelos configurables:** los identificadores de modelo (`claude-haiku-5-5` y `claude-sonnet-5-5` a la fecha) se leen de la configuración, no se escriben en el código. Cambiar de modelo no requiere un despliegue de código.
5. **Salida estructurada:** el modelo responde con un esquema JSON que incluye los campos genéricos de recibo y una lista de **advertencias** (texto ilegible, campo ausente, valor dudoso). Lo que devuelve el modelo se valida siempre en el adaptador, aunque la salida estructurada garantice la forma.
6. **Reintento con modelo mayor:** si la extracción con Haiku sale con advertencias o falla las validaciones propias, se repite una vez con Sonnet. Se guarda el modelo que produjo el resultado final y si hubo reintento.
7. **Validaciones propias:** reglas deterministas en el adaptador, independientes del proveedor: monto mayor que cero, periodo con inicio anterior al fin, fecha límite coherente con el periodo y formatos esperados. Cada regla que falla agrega una advertencia.
8. **Verificación contra el texto del documento, solo en PDFs con texto:** se extrae el texto del PDF con una librería .NET sin binarios nativos (candidata: PdfPig) y se comprueba que el monto, el número de cuenta y la fecha límite extraídos **aparezcan literalmente** en ese texto. Si un valor no aparece, se agrega la advertencia "valor no encontrado en el documento". Las fotos y los PDFs escaneados (sin texto) no se verifican así en esta versión.
9. **Cálculo de la confianza:** se combina a partir de las advertencias del modelo y de las validaciones (puntos 5, 7 y 8). Cómo se traduce a `Confidence` y qué cuenta como "baja confianza" para el Principio IV.1: **[POR DEFINIR]**.
10. **Tipos de archivo:** se aceptan PDF, JPEG, PNG y WebP; el resto se rechaza al recibir el documento (VIII.5). Si las fotos HEIC del iPhone se rechazan o se convierten a JPEG, y si Claude acepta HEIC directamente (no verificado): **[POR DEFINIR]**. Límites de tamaño: **[POR DEFINIR]**.
11. **Privacidad:**
    - el documento viaja **dentro de la petición** (base64); no se usa la API de archivos de Anthropic, que lo guardaría del lado del proveedor;
    - la telemetría y los logs de `Microsoft.Extensions.AI` y del SDK se configuran **sin contenido** de mensajes ni de respuestas;
    - la llave `ANTHROPIC_API_KEY` se inyecta como secreto (VIII.4).
12. **Fallas:** un error del proveedor, una respuesta que no cumple el esquema o una negativa del modelo se tratan como extracción fallida: se publica `ExtractionFailed` y queda la captura manual (IV.6). Los reintentos por fallas transitorias siguen el Principio IX.3.
13. **Implementación falsa:** para pruebas y desarrollo local hay un `IDocumentExtractor` falso que no llama a ningún proveedor (IX.2). El adaptador real se prueba con un `IChatClient` falso que devuelve respuestas preparadas.
14. **Versión:** paquete `Anthropic`, versión mayor estable vigente al armar el esqueleto (13.x a la fecha), fijada en `Directory.Packages.props`. No se usan versiones preliminares.

### Validación pendiente

La elección de Claude sobre Gemini se tomó con información pública (precio, salida estructurada, estabilidad y privacidad), sin medir la precisión con recibos reales. Queda pendiente una **prueba comparativa**: unos 10 recibos reales, en foto y en PDF, procesados con Claude Haiku 5.5 y con Gemini 3.5 Flash-Lite con el mismo esquema, comparando campo por campo. Si la prueba favorece claramente a Gemini, se escribe un ADR nuevo que reemplaza a este; gracias a `IChatClient`, el cambio se limita al registro del cliente y al *prompt*.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **Gemini (Google)** | Opción viable: SDK oficial `Google.GenAI`, PDF e imagen nativos (incluido HEIC). En contra: su modelo económico (3.5 Flash-Lite, $0.30 / $2.50 USD por millón de tokens) cuesta más que Haiku 5.5 ($0.10 / $0.50); la salida estructurada solo soporta parte de JSON Schema e ignora algunas restricciones sin avisar; los precios y nombres de modelo cambian seguido; y su nivel gratuito usa el contenido para mejorar sus productos, lo que obliga a pagar desde el inicio para cumplir el Principio VIII. Queda como candidata en la prueba comparativa. |
| **OpenAI** | Viable, pero no se evaluó a fondo; se prefirió decidir entre Claude y Gemini. |
| **OCR especializado en la nube** (Azure Document Intelligence, AWS Textract) | Da confianza nativa por campo, pero sus modelos prediseñados son para facturas genéricas: campos como periodo de consumo o kWh exigen entrenar un modelo propio. Además amarra a una nube antes de decidir el hosting (ADR-0007). |
| **Modelo local** (por ejemplo, Ollama) | Nada sale de la infraestructura propia, pero los modelos pequeños fallan más con fotos de celular y se necesita hardware con GPU, lo que complica el hosting. |
| **Tesseract como lector principal, con un LLM que interpreta el texto** | El texto plano pierde el acomodo de las tablas del recibo, y el LLM extrae peor que leyendo la imagen. Tesseract no funciona bien con fotos de celular sin preprocesarlas, y requiere binarios nativos. No mejora la privacidad: el texto también viaja al proveedor. |
| **Tesseract para verificar fotos** | Útil para detectar valores inventados por el modelo, pero agrega binarios nativos y preprocesamiento de imagen. Queda como **evolución prevista** si en la práctica aparecen errores en fotos que la confirmación humana no detecta a tiempo. |
| **Adaptador `internal` dentro de `AdminRentas.Ingestion`** | Más simple (un proyecto menos), pero la implementación del módulo tendría que referenciar los paquetes del proveedor. En un proyecto aparte, el dominio ni siquiera puede ver esos paquetes, y el mismo patrón sirve para todos los adaptadores externos. |
| **SDK de Anthropic directo, sin `Microsoft.Extensions.AI`** | Menos capas, pero cambiar de proveedor obligaría a reescribir el adaptador completo. `IChatClient` traduce el PDF y la salida estructurada de forma nativa, así que no se pierde nada. |
| **Microsoft Agent Framework** | Está pensado para agentes (herramientas, conversación de varios turnos, orquestación). La extracción es una sola llamada; el framework agregaría capas sin beneficio. Puede reconsiderarse en otro ADR si aparece un asistente conversacional. |

## Consecuencias

**Positivas**

- Lee recibos mexicanos sin entrenar ningún modelo; el esquema JSON define los campos.
- Costo despreciable: menos de $0.001 USD por recibo con Haiku 5.5 y alrededor de $0.01 con Sonnet 5.5.
- No depende de la nube que se elija para el hosting.
- Cambiar de proveedor afecta solo al registro de `IChatClient` y al *prompt*; el dominio y el puerto no cambian (IV.5).
- La verificación contra el texto del PDF detecta valores inventados con una regla propia que se puede probar.

**Negativas o riesgos**

- **La confianza no es un número calibrado** como el de un OCR especializado: depende de advertencias y validaciones propias. **Mitigación:** la confirmación humana es obligatoria (IV.1), y el cálculo se define y se prueba por separado.
- **Las fotos no se verifican contra el documento** en esta versión. **Mitigación:** la confirmación humana compara contra el original (IV.3), y Tesseract queda como evolución prevista.
- **Dependencia de un proveedor externo:** caídas, cambios de precio o retiro de modelos. **Mitigación:** modelos configurables, `IChatClient` y captura manual siempre disponible (IV.6).
- **El documento sale de la infraestructura propia.** Viaja al proveedor, que no lo usa para entrenar por defecto pero lo retiene durante un tiempo según su política. **Mitigación:** solo se envía en la petición, sin la API de archivos.
- **`Microsoft.Extensions.AI` es una capa más** que traduce entre la interfaz común y el SDK; un error de traducción podría pasar inadvertido. **Mitigación:** pruebas de integración del adaptador con documentos de ejemplo.
- **Un proyecto más por cada módulo que tenga adaptadores externos** (`.Adapters`), además de la implementación y `.Contracts`.
- La elección no se ha validado con recibos reales (ver *Validación pendiente*).

## Decisiones derivadas

- **Constitución:** marcar como decidido el proveedor de IA en *Restricciones tecnológicas*.
- **Mapa de módulos:** reglas de dependencia 7 (solo `Host` referencia los proyectos `.Adapters`) y 8 (solo `AdminRentas.Ingestion.Adapters` y `Host` usan `Anthropic` y `Microsoft.Extensions.AI`), convención de proyectos `.Adapters` y nota en el módulo `Ingestion`.
- **Prueba comparativa** Claude contra Gemini con unos 10 recibos reales.
- **Cálculo de la confianza** y umbral de "baja confianza": **[POR DEFINIR]**, en la spec de `Ingestion`.
- **Fotos HEIC** y **límites de tamaño**: **[POR DEFINIR]**.
- **Tesseract para verificar fotos:** evolución prevista, sujeta a un ADR nuevo.
