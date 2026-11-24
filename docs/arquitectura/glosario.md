# Glosario del dominio (español → código)

> **Versión:** 0.1.0 · **Fecha:** 2026-11-23
>
> El código, los commits y los identificadores van en **inglés** ([ADR-0004](../adr/0004-idioma-del-codigo.md)). Los documentos y las conversaciones van en **español**. Este glosario es la traducción oficial: si un concepto no está aquí, se agrega antes de usarlo en el código.

## Módulos

| Español | Código | Esquema en PostgreSQL | Prefijo de IDs en specs |
|---|---|---|---|
| Identidad | `Identity` | `identity` | `IDN` |
| Propiedades | `Properties` | `properties` | `PRP` |
| Ingesta | `Ingestion` | `ingestion` | `ING` |
| Servicios | `Utilities` | `utilities` | `UTL` |
| Ocupación | `Occupancy` | `occupancy` | `OCC` |
| Notificaciones | `Notifications` | `notifications` | `NTF` |
| Liquidaciones *(v2)* | `Settlements` | `settlements` | `STL` |
| Mantenimiento *(v2)* | `Maintenance` | `maintenance` | `MNT` |
| Archivos *(futuro)* | `Files` | `files` | `FIL` |
| Núcleo compartido | `SharedKernel` | — | — |
| Infraestructura compartida | `Infrastructure` | — | — |

El nombre raíz de la solución y los namespaces es **`AdminRentas`**: es el nombre propio del producto, no se traduce.

## Personas y roles

| Español | Código | Nota |
|---|---|---|
| Dueño | `Owner` | |
| Administrador | `Manager` | No `Admin`, para no confundirlo con un administrador del sistema. |
| Inquilino | `Tenant` | |
| Usuario | `User` | |
| Asignación de propiedad | `PropertyAssignment` | Qué propiedades ve un administrador. |

## Propiedades

| Español | Código |
|---|---|
| Propiedad | `Property` |
| Configuración de la propiedad | `PropertySettings` |
| Límite de confirmación | `ConfirmationThreshold` |
| Zona horaria | `TimeZone` |

## Ingesta

| Español | Código |
|---|---|
| Documento | `Document` |
| Extracción | `Extraction` |
| Original temporal | `TemporaryOriginal` |
| Huella (SHA-256) | `ContentHash` |
| Nivel de confianza | `Confidence` |
| Origen (correo, subida, foto) | `DocumentSource` (`Email`, `Upload`, `Photo`) |

## Servicios

| Español | Código | Nota |
|---|---|---|
| Servicio (luz, agua, internet…) | `Utility` | |
| Proveedor del servicio (CFE, JUMAPAM…) | `Provider` | |
| Número de contrato o servicio | `AccountNumber` | |
| Periodicidad | `BillingFrequency` (`Monthly`, `Bimonthly`, `Annual`) | |
| Recibo | `Bill` | |
| Periodo del recibo | `BillingPeriod` | |
| Fecha límite de pago | `DueDate` | |
| Domiciliado | `AutoPay` | |
| Pago | `Payment` | |
| Referencia de comprobante | `PaymentReference` | Clave de rastreo SPEI, folio o nota. |
| Reverso | `Reversal` | |
| Medidor | `Meter` | |
| Lectura de medidor | `MeterReading` | |
| Consumo | `Consumption` | |
| Prorrateo | `Proration` | |
| Método de prorrateo (días, consumo) | `ProrationMethod` (`ByDays`, `ByConsumption`) | |
| Porción del inquilino | `TenantShare` | |

**Estados del recibo**

| Español | Código |
|---|---|
| Por confirmar | `PendingConfirmation` |
| Pendiente de pago | `Unpaid` |
| Domiciliado (esperando cargo) | `AwaitingAutoPay` |
| Pagado | `Paid` |
| Vencido | `Overdue` |
| Descartado | `Discarded` |

## Ocupación

| Español | Código |
|---|---|
| Estancia | `Stay` |
| Tipo de estancia (vacacional, mensual) | `StayType` (`Vacation`, `Monthly`) |
| Inquilino cubre servicios | `TenantPaysUtilities` |
| Check-in / check-out | `CheckIn` / `CheckOut` |
| Calendario | `Calendar` |

## Notificaciones

| Español | Código |
|---|---|
| Aviso | `Notification` |
| Canal (Telegram, correo) | `Channel` (`Telegram`, `Email`) |
| Preferencia de avisos | `NotificationPreference` |
| Plantilla de mensaje | `MessageTemplate` |
| Bitácora de envíos | `DeliveryLog` |
| Enlace de WhatsApp | `WhatsAppLink` |
| Resumen semanal | `WeeklyDigest` |

## Liquidaciones *(v2)*

| Español | Código |
|---|---|
| Liquidación | `Settlement` |
| Movimiento | `LedgerEntry` |
| Saldo | `Balance` |
| Cobro del administrador por evento | `ManagerFee` |
| Gasto | `Expense` |
| Reembolso | `Reimbursement` |
| Transferencia | `Transfer` |
| Renta | `Rent` |

## Mantenimiento *(v2)*

| Español | Código |
|---|---|
| Equipo (minisplit, calentador…) | `Equipment` |
| Plan preventivo | `PreventivePlan` |
| Ticket | `MaintenanceTicket` |
| Cotización | `Quote` |
| Proveedor de mantenimiento | `Vendor` |

## Núcleo compartido

| Español | Código |
|---|---|
| Dinero | `Money` |
| Periodo `[inicio, fin)` | `DateRange` |
| Resultado / error | `Result` / `Error` |
