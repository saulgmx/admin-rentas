# ADR-0005: Bus de eventos en proceso con Wolverine y *outbox* en PostgreSQL

- **Estado:** Aceptado
- **Fecha:** 2026-11-23
- **Principios relacionados:** II (monolito modular), VII (pruebas), IX (integraciones externas), X (operación simple)

> Un ADR aceptado **no se edita**. Si la decisión cambia, se escribe uno nuevo que lo reemplaza y aquí solo se actualiza el estado.

## Contexto

Los módulos se comunican por su API pública o por **eventos de integración publicados en un bus interno del proceso** (Principio II.4). Hay que decidir cómo se implementa ese bus.

Restricciones:

- **Las fallas de integraciones externas no bloquean el flujo principal; se reintentan y, si siguen fallando, se notifican** (Principio IX.3). Ejemplo: `Utilities` confirma un recibo y publica `BillConfirmed`; si el contenedor se reinicia antes de que `Notifications` lo procese, un bus solo en memoria pierde el evento y el dueño nunca recibe el aviso. Por lo tanto se necesita **entrega garantizada**: el evento se guarda en la misma transacción que el cambio que lo origina (*outbox*).
- **Un solo despliegue:** la aplicación en un contenedor y PostgreSQL, sin *broker* externo (Principio X.1).
- **Los consumidores son idempotentes** (mapa de módulos, *Convenciones*).
- **Cada módulo es dueño de su esquema** y tiene su propio `DbContext` (Principio II.3, ADR-0001).
- **Volumen muy bajo** (decenas de eventos al día): el rendimiento no es un criterio.
- MediatR, la opción más conocida, tiene licencia comercial (ADR-0001, *Consecuencias*).

## Decisión

El bus de eventos en proceso se implementa con **[Wolverine](https://wolverine.netlify.app/)** (licencia MIT), usando su *outbox* e *inbox* durables en PostgreSQL. Wolverine es un detalle de `Infrastructure`: los módulos no dependen de él.

### Detalle

1. **Dónde vive:** la configuración de Wolverine está en `AdminRentas.Infrastructure` y el `Host` la activa. Es el único lugar que referencia los paquetes de Wolverine.
2. **Publicación:** los módulos publican eventos de integración mediante una abstracción propia, `IEventPublisher`, definida en `Infrastructure`. Su implementación usa el *outbox* de Wolverine ligado al `DbContext` del módulo, de modo que el cambio de estado y el evento se guardan en **la misma transacción**.
3. **Consumo:** los *handlers* son clases simples dentro de la implementación del módulo consumidor (`internal`), sin tipos de Wolverine en su firma. Reciben el evento definido en el proyecto `.Contracts` del módulo que lo publica.
4. **Descubrimiento explícito:** cada módulo registra sus *handlers* desde su método de registro (`AddUtilitiesModule()`), indicando su propio ensamblado. No se usa el escaneo automático de todos los ensamblados.
5. **Entrega:** los eventos de integración viajan por colas locales **durables** de Wolverine (dentro del proceso, persistidas en PostgreSQL), no de forma síncrona dentro de la transacción del publicador. Una falla en un consumidor no revierte el cambio del publicador.
6. **Almacenamiento:** las tablas de mensajes de Wolverine (*outbox*, *inbox*, mensajes programados y mensajes fallidos) viven en un esquema propio, **`wolverine`**, en la misma base de datos. Es infraestructura compartida, no un módulo: ningún módulo lee ni escribe en él directamente.
7. **Reintentos y fallas:** cada consumidor tiene una política de reintentos con espera creciente. Un mensaje que agota sus reintentos pasa a la tabla de mensajes fallidos y se **alerta por Telegram** (Principio X.5). Número de reintentos y tiempos de espera: **[POR DEFINIR]**.
8. **Idempotencia:** el *inbox* de Wolverine evita procesar dos veces el mismo mensaje en el caso normal, pero **los consumidores siguen siendo idempotentes** por diseño, como exige el mapa de módulos. No se confía solo en el *inbox*.
9. **Pruebas de arquitectura:** una prueba verifica que ningún proyecto de módulo (implementación ni `.Contracts`) use tipos del espacio de nombres `Wolverine`.
10. **Versión:** paquete `WolverineFx`, versión mayor estable vigente al armar el esqueleto (6.x a la fecha, con compilación nativa para `net10.0`), fijada en `Directory.Packages.props`. No se usan versiones preliminares.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **Despachador propio con *outbox* en PostgreSQL** | Control y transparencia totales, pero hay que programar y mantener *outbox*, *inbox*, reintentos, mensajes fallidos y el trabajo de envío (unas 300 a 500 líneas con sus pruebas). Son piezas con errores sutiles (duplicados, mensajes atorados) que Wolverine ya resuelve. Queda como plan de salida si Wolverine deja de convenir. |
| **Despachador propio solo en memoria** | Pierde eventos en cada reinicio o despliegue; incumple el Principio IX.3. |
| **MassTransit** | La versión 9 tiene licencia comercial y la 8 deja de recibir soporte a finales de 2026. |
| **MediatR** | Licencia comercial (con edición gratuita) y sin *outbox*: solo despacha en memoria. |
| **Mediator (martinothamar)** | MIT y con generación de código en compilación, pero solo en memoria, sin *outbox*. |
| **Brighter** | MIT y con *outbox*, pero menos usado en monolitos modulares y más pesado de configurar para este caso. |

## Consecuencias

**Positivas**

- Se cumple el Principio IX.3 sin programar infraestructura de mensajería: entrega garantizada, reintentos y mensajes fallidos vienen incluidos.
- Sin *broker* externo: todo vive en el contenedor y en PostgreSQL (Principio X.1).
- Los módulos solo conocen `IEventPublisher` y clases simples; cambiar Wolverine por otra opción afecta a `Infrastructure` y no al dominio (Principio II.9).
- Los mensajes programados de Wolverine podrían servir para avisos diferidos; su uso se decide por separado.

**Negativas o riesgos**

- **"Magia" por convención y generación de código al arrancar:** más difícil de depurar y de aprender. **Mitigación:** descubrimiento explícito por módulo, *handlers* simples y pruebas de integración con Testcontainers del flujo completo publicar → consumir.
- **Dependencia del ecosistema de JasperFx**, que saca versiones mayores con frecuencia; la versión mayor anterior solo recibe parches críticos durante seis meses. **Mitigación:** versión fijada, Wolverine aislado en `Infrastructure` y despachador propio como plan de salida.
- **Un esquema compartido (`wolverine`)** en lugar de un *outbox* por módulo. Si un módulo se extrae como servicio, se lleva su esquema y necesita su propio almacenamiento de mensajes. Es aceptable porque la extracción no es un objetivo (Principio II.9).
- La entrega es **asíncrona**: los consumidores ven los eventos unos instantes después del cambio. Las pruebas de aceptación deben esperar a que el mensaje se procese.

## Decisiones derivadas

- **Mapa de módulos:** actualizar la fila *Bus de eventos* de *Aspectos transversales* (hoy dice "*Outbox* por módulo si se requiere entrega garantizada").
- **Constitución:** marcar como decidido el bus de eventos interno en *Restricciones tecnológicas*.
- **Esqueleto:** `IEventPublisher`, configuración de Wolverine, prueba de arquitectura y prueba de integración del flujo publicar → consumir.
- **Trabajos programados:** si se usan los mensajes programados de Wolverine o una librería aparte se decide por separado: **[POR DEFINIR]**.
