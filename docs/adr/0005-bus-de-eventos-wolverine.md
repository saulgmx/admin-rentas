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
7. **Reintentos y fallas:** la política distingue dos tipos de falla, según el tipo de excepción.
   - **Transitorias** (red, tiempo de espera agotado, "demasiadas peticiones" de una API externa, base de datos ocupada): primero **3 reintentos inmediatos** con espera creciente (1 s, 5 s y 15 s) y después **3 reintentos programados** (5 min, 30 min y 2 h), en total unas 2.5 horas. Los reintentos programados no bloquean la cola mientras esperan.
   - **Permanentes** (datos inválidos, errores de programación): **sin reintentos**.
   - Un mensaje que agota sus reintentos, o que falla de forma permanente, pasa a la tabla de mensajes fallidos y se **alerta por Telegram** (Principio X.5).
   - Los números son **valores iniciales** y viven en la configuración. Ajustarlos no requiere un ADR nuevo; cambiar la política, sí.
   - La lista de excepciones de cada tipo se define al armar el esqueleto, con pruebas que verifican la clasificación.
8. **Idempotencia:** el *inbox* de Wolverine evita procesar dos veces el mismo mensaje en el caso normal, pero **los consumidores siguen siendo idempotentes** por diseño, como exige el mapa de módulos. No se confía solo en el *inbox*.
9. **Trabajos programados:** se usan los **trabajos recurrentes de Wolverine** (`opts.Schedules.ScheduleRecurring`, disponibles desde la versión 6.34).
   - Las expresiones *cron* se calculan en la zona horaria de la propiedad, que es un dato de cada propiedad (constitución V.2), con los cambios de horario resueltos por Wolverine (usa Cronos).
   - La siguiente ejecución de cada trabajo se guarda en PostgreSQL y sobrevive a reinicios; cada ejecución lleva un identificador "nombre + hora" que evita duplicados.
   - Los módulos **no** usan tipos de Wolverine para declararlos: declaran cada trabajo (nombre, expresión *cron* y mensaje) mediante una abstracción propia definida en `Infrastructure`, como `IEventPublisher`, e `Infrastructure` los registra en Wolverine. El mensaje lo atiende un *handler* simple del módulo.
   - **Trabajos diseñados por estado, no por evento.** Wolverine **no recupera las ejecuciones perdidas**: si el proceso está detenido a la hora programada, esa ejecución se salta. Por eso cada trabajo busca *todo lo que falta hacer* (por ejemplo, todos los recibos por vencer que aún no tienen aviso), no solo lo de ese día. Así, una ejecución perdida solo se atrasa hasta la siguiente. Junto con la idempotencia (Principio X.6), esto permite correr un trabajo más seguido si conviene.
   - Al terminar, cada trabajo avisa a Healthchecks.io (ADR-0007), que alerta si no avisa a tiempo.
10. **Pruebas de arquitectura:** una prueba verifica que ningún proyecto de módulo (implementación ni `.Contracts`) use tipos del espacio de nombres `Wolverine`.
11. **Versión:** paquete `WolverineFx`, versión mayor estable vigente al armar el esqueleto (6.x a la fecha, con compilación nativa para `net10.0`; al menos 6.34 por los trabajos recurrentes), fijada en `Directory.Packages.props`. No se usan versiones preliminares.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **Despachador propio con *outbox* en PostgreSQL** | Control y transparencia totales, pero hay que programar y mantener *outbox*, *inbox*, reintentos, mensajes fallidos y el trabajo de envío (unas 300 a 500 líneas con sus pruebas). Son piezas con errores sutiles (duplicados, mensajes atorados) que Wolverine ya resuelve. Queda como plan de salida si Wolverine deja de convenir. |
| **Despachador propio solo en memoria** | Pierde eventos en cada reinicio o despliegue; incumple el Principio IX.3. |
| **Trabajos programados con un servicio propio y Cronos** | Habría que programar la durabilidad, la zona horaria y la prevención de duplicados que Wolverine ya trae. |
| **Trabajos programados con Quartz.NET** | Otro programador de tareas con sus propias tablas y configuración, cuando Wolverine ya cubre el caso. |
| **MassTransit** | La versión 9 tiene licencia comercial y la 8 deja de recibir soporte a finales de 2026. |
| **MediatR** | Licencia comercial (con edición gratuita) y sin *outbox*: solo despacha en memoria. |
| **Mediator (martinothamar)** | MIT y con generación de código en compilación, pero solo en memoria, sin *outbox*. |
| **Brighter** | MIT y con *outbox*, pero menos usado en monolitos modulares y más pesado de configurar para este caso. |

## Consecuencias

**Positivas**

- Se cumple el Principio IX.3 sin programar infraestructura de mensajería: entrega garantizada, reintentos y mensajes fallidos vienen incluidos.
- Sin *broker* externo: todo vive en el contenedor y en PostgreSQL (Principio X.1).
- Los módulos solo conocen `IEventPublisher` y clases simples; cambiar Wolverine por otra opción afecta a `Infrastructure` y no al dominio (Principio II.9).
- Los trabajos programados usan la misma infraestructura durable que los eventos, sin otra librería.

**Negativas o riesgos**

- **"Magia" por convención y generación de código al arrancar:** más difícil de depurar y de aprender. **Mitigación:** descubrimiento explícito por módulo, *handlers* simples y pruebas de integración con Testcontainers del flujo completo publicar → consumir.
- **Dependencia del ecosistema de JasperFx**, que saca versiones mayores con frecuencia; la versión mayor anterior solo recibe parches críticos durante seis meses. **Mitigación:** versión fijada, Wolverine aislado en `Infrastructure` y despachador propio como plan de salida.
- **Un esquema compartido (`wolverine`)** en lugar de un *outbox* por módulo. Si un módulo se extrae como servicio, se lleva su esquema y necesita su propio almacenamiento de mensajes. Es aceptable porque la extracción no es un objetivo (Principio II.9).
- **Las ejecuciones perdidas de un trabajo programado no se recuperan.** **Mitigación:** trabajos diseñados por estado e idempotentes, y alerta de Healthchecks.io si un trabajo no avisa a tiempo.
- La entrega es **asíncrona**: los consumidores ven los eventos unos instantes después del cambio. Las pruebas de aceptación deben esperar a que el mensaje se procese.

## Listo para aceptar

- [x] **Evidencia:** la documentación oficial de Wolverine (consultada el 2026-11-23) confirma el *outbox* e *inbox* durables en PostgreSQL, los mensajes programados y los trabajos recurrentes con zona horaria. El paquete 6.x compila para `net10.0`. Falta la prueba en el propio proyecto; la cubre la prueba de integración publicar → consumir del esqueleto.
- [x] **Alternativas:** se compararon despachadores propios (con y sin *outbox*), MassTransit, MediatR, Mediator y Brighter para el bus, y un servicio propio con Cronos y Quartz.NET para los trabajos programados.
- [x] **Acuerdo:** el dueño eligió el bus, los trabajos programados y la política de reintentos.
- [x] **Documentación:** no hay `[POR DEFINIR]`. Los valores de reintento son iniciales y configurables; la lista de excepciones por tipo se define con sus pruebas en el esqueleto.
- [x] **Implementación y revisión:** se implementa al armar el esqueleto. Se revisa si Wolverine sube de versión mayor (su versión anterior solo recibe parches críticos seis meses) y en la retrospectiva del primer hito, sobre todo los valores de reintento.

## Decisiones derivadas

- **Mapa de módulos:** actualizar la fila *Bus de eventos* de *Aspectos transversales* (hoy dice "*Outbox* por módulo si se requiere entrega garantizada").
- **Constitución:** marcar como decidido el bus de eventos interno en *Restricciones tecnológicas*.
- **Esqueleto:** `IEventPublisher`, configuración de Wolverine, prueba de arquitectura y prueba de integración del flujo publicar → consumir.
- **Mapa de módulos:** actualizar la fila *Trabajos programados* de *Aspectos transversales* con los trabajos recurrentes de Wolverine.
- **Glosario:** proponer el nombre de la abstracción para declarar trabajos programados antes de usarla.

## Ediciones por datos sensibles

- **2026-11-24:** se quitó la zona horaria concreta de la propiedad. Ahora es un dato de cada propiedad (constitución V.2). La decisión no cambia.
