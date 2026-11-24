# Constitución del proyecto — Administrador de Rentas

> **Versión:** 0.5.0 (borrador) · **Fecha:** 2026-11-23 · **Estado:** En revisión
>
> Este documento define las reglas que no se negocian en el proyecto. Toda especificación, plan, tarea o cambio de código debe cumplirlas. Si una funcionalidad necesita romper una regla, primero se modifica esta constitución (ver *Gobernanza*).

---

## Propósito

Mantener sincronizados al **dueño** (a distancia) y al **administrador** (en la ciudad de la propiedad) sobre el estado de un departamento en renta mixta (vacacional y mensual): servicios, pagos, ocupación, gastos y mantenimiento.

### Alcance

- Uso personal: un dueño y uno o pocos administradores.
- El modelo de datos admite **varias propiedades** desde el inicio, aunque hoy solo exista una. Es barato hacerlo ahora y caro después.
- No es un producto multi-cliente (*multi-tenant*). No se diseña para eso.

### Actores

| Actor | Descripción | Implicación de diseño |
|---|---|---|
| **Dueño** | Paga servicios, reembolsa gastos y recibe la renta. Perfil técnico. | Vista de resumen, aprobaciones, reportes. |
| **Administrador** | Cobra a los inquilinos, consigue inquilinos, paga gastos y cobra por evento. Perfil tecnológico básico. | Acciones mínimas, guiadas, desde el celular. |

---

## Principios

### I. La especificación va primero

1. Ninguna funcionalidad se implementa sin una especificación aprobada que incluya **criterios de aceptación** en formato *Dado / Cuando / Entonces*.
2. El flujo es: **Especificación → Plan → Tareas → Implementación**. La especificación describe *qué* y *por qué*; el plan describe *cómo*.
3. Las especificaciones están en el repositorio, versionadas junto al código, y se actualizan cuando cambia el comportamiento. Una especificación desactualizada es un defecto.
4. Cada especificación declara **qué módulos toca** y **qué contratos agrega o modifica**.

### II. Monolito modular con límites estrictos

1. La aplicación es **un solo desplegable** compuesto por módulos con límites explícitos.
2. Módulos iniciales: `Identidad`, `Propiedades`, `Ingesta`, `Servicios`, `Ocupación`, `Notificaciones`. Módulos previstos: `Liquidaciones`, `Mantenimiento` y `Archivos`. El detalle de cada módulo está en [`arquitectura/mapa-modulos.md`](arquitectura/mapa-modulos.md).
3. **Cada módulo es dueño de sus datos:** tiene su propio esquema en la base de datos. Ningún módulo lee ni escribe en el esquema de otro, y no hay *joins* ni llaves foráneas entre esquemas. Las referencias entre módulos se guardan como identificadores.
4. **Comunicación entre módulos solo por:**
   - su **API pública** (interfaces o fachadas expuestas explícitamente), o
   - **eventos de integración** publicados en un bus interno del proceso.
5. Los detalles internos de un módulo (entidades, repositorios, tablas) **no se exponen**. Si otro módulo los necesita, se agrega un contrato.
6. El **núcleo compartido** (*shared kernel*) es mínimo y solo contiene tipos de valor sin lógica de negocio: `Dinero`, `Periodo`, identificadores tipados y `Resultado/Error`. Agregar algo al núcleo requiere un ADR.
7. Los límites se **verifican automáticamente** con pruebas de arquitectura en CI. Si se rompe un límite, el build falla.
8. Un módulo debe poder extraerse como servicio independiente en el futuro sin reescribir su dominio. No es un objetivo, pero sí una prueba de que los límites son correctos.

### III. Simplicidad para el administrador

1. Toda acción del administrador se diseña para hacerse **desde el celular, con una mano, en menos de 30 segundos**.
2. Las acciones frecuentes, como *subir recibo*, *registrar cobro*, *registrar gasto* o *reportar problema*, deben estar **a un toque** desde la pantalla principal.
3. Se prefieren botones, fotos y opciones predefinidas sobre formularios con texto libre.
4. Todo en **español de México**, sin jerga técnica ni contable.
5. Si el administrador no puede completar una acción solo, la funcionalidad no está terminada.

### IV. La IA sugiere y una persona confirma

1. Los datos extraídos con IA (OCR o modelos de visión) **siempre** entran en estado *por confirmar*. Ningún dato financiero se vuelve definitivo sin la confirmación de una persona.
   - **Quién confirma depende del monto.** El administrador puede confirmar recibos por debajo de un **límite configurable** (valor inicial: **$1,500 MXN**). Los recibos que igualan o superan el límite los confirma el dueño.
   - Si la IA reporta **baja confianza** o advertencias en la extracción, el recibo lo confirma el dueño **sin importar el monto**.
   - Toda confirmación registra quién la hizo y cuándo, y el dueño puede ver en el historial lo que confirmó el administrador.
2. **No se almacenan archivos de forma permanente en la primera versión.** De cada documento se guardan sus metadatos: nombre, tipo, tamaño, origen y una **huella SHA-256** para detectar recibos duplicados. Guardar archivos de forma permanente será una funcionalidad posterior (módulo `Archivos`).
3. **Retención temporal del original:** el documento se conserva solo mientras su recibo está *por confirmar*, para que quien confirme pueda compararlo con los datos extraídos.
   - Se borra en cuanto el recibo se **confirma o se descarta**, o a los **7 días** si nadie lo confirma (lo que pase primero).
   - Si vence sin confirmarse, el recibo sigue *por confirmar* y se avisa al dueño de que el original ya no está disponible.
   - Se guarda en PostgreSQL, dentro del esquema de `Ingesta`, con un tamaño máximo de **10 MB** por documento. No se incluye en los respaldos.
4. La extracción guarda el **proveedor y modelo de IA usados**, así como su **nivel de confianza** o las advertencias, para poder auditar errores.
5. El proveedor de IA queda detrás de una interfaz (ver Principio IX). Cambiar de proveedor no debe tocar el dominio.
6. Si la IA falla, siempre existe la **captura manual** como alternativa.

### V. Dinero, fechas y periodos sin ambigüedad

1. **Dinero:** se representa como entero en centavos o como decimal exacto, **nunca como punto flotante**. Siempre va acompañado de la moneda (por defecto MXN).
2. **Fechas y horas:** se guardan en UTC y se muestran en la zona horaria del usuario. La zona horaria de la propiedad es una zona horaria configurada y la del dueño una zona horaria configurada. Las fechas sin hora (vencimientos, inicio de estancia) se guardan como **fecha local de la propiedad**.
3. **Periodos:** son intervalos semiabiertos `[inicio, fin)`. Aplica a periodos de facturación, estancias y prorrateos.
4. **Prorrateo:** el método se define **por estancia** al acordar la renta, y se aplica a todos sus recibos.
   - **Por días (predeterminado):** se reparte según los días naturales del periodo del recibo que se traslapan con la estancia.
   - **Por consumo (opcional, si el inquilino lo pide):** se cobra el consumo medido entre la lectura de entrada y la de salida (o el corte del periodo), multiplicado por el **costo promedio por unidad** del recibo (monto total ÷ consumo total del periodo). Solo aplica a servicios con medidor, como luz y agua. Cada lectura se registra con valor, fecha y quién la tomó. Si falta una lectura, el recibo se prorratea por días y se avisa al dueño.
   - **Redondeo:** se usa redondeo bancario a centavos, y la diferencia se asigna al dueño para que la suma siempre cuadre.
5. Toda regla de cálculo de dinero tiene pruebas con casos límite: meses de 28 a 31 días, estancias que cruzan periodos, recibos bimestrales y lecturas de medidor faltantes.

### VI. Trazabilidad y registros financieros inmutables

1. Los movimientos financieros (pagos, cobros, reembolsos, transferencias) **no se editan ni se borran**. Se corrigen con un **movimiento de reverso** y uno nuevo.
2. Cada cambio relevante registra **quién, cuándo y qué**, en una bitácora de auditoría.
3. Los demás registros usan **borrado lógico**.
4. Todo pago o cobro puede tener una **referencia de comprobante** en texto: clave de rastreo SPEI, folio o nota. Adjuntar el archivo del comprobante queda para el módulo `Archivos`.

### VII. Pruebas como parte de la especificación

1. Cada criterio de aceptación de una especificación tiene **al menos una prueba automatizada** que lo verifica.
2. **Pirámide de pruebas por módulo:**
   - pruebas unitarias del dominio, sin infraestructura;
   - pruebas de integración del módulo, con base de datos real en contenedor;
   - pruebas de aceptación por funcionalidad, a través de la API pública.
3. **Pruebas de arquitectura** en CI, que validan los límites entre módulos (Principio II.7).
4. El código nuevo no se integra a la rama principal si las pruebas fallan.

### VIII. Seguridad y privacidad

1. **Datos mínimos de inquilinos:** alias, contacto, fechas y condiciones. No se guardan identificaciones oficiales salvo que una especificación lo justifique, y en ese caso van cifradas.
2. Se cumple con la ley mexicana de protección de datos personales en lo que aplique: aviso de privacidad para inquilinos si se guardan sus datos, y eliminación cuando ya no se necesiten.
3. **Autenticación obligatoria** y **roles** (`dueño`, `administrador`). El administrador solo ve las propiedades que tiene asignadas.
4. Los secretos (llaves de IA, token de Telegram, credenciales SMTP) **nunca** van en el repositorio. Se inyectan como variables de entorno o desde un gestor de secretos.
5. Documentos recibidos para extracción: se valida el tipo y el tamaño, y solo se conservan bajo la retención temporal del Principio IV.3. Nunca se escriben en disco ni en los logs, y solo los pueden ver el dueño y los administradores de la propiedad.

### IX. Integraciones externas detrás de puertos

1. Todo servicio externo se consume a través de una **interfaz (puerto)** definida por el módulo que lo necesita, con un **adaptador** intercambiable. Esto aplica al proveedor de IA, Telegram, correo y cualquier API futura (como el almacenamiento de archivos).
2. Cada adaptador tiene una implementación **falsa o en memoria** para pruebas y desarrollo local.
3. Las fallas de integraciones externas **no bloquean** el flujo principal. Se reintentan y, si siguen fallando, se notifican.
4. Los mensajes de WhatsApp se envían como **enlaces `wa.me` con texto preconstruido** que el dueño manda manualmente. No hay integración con la API de WhatsApp Business mientras esta constitución no cambie.

### X. Operación simple y reproducible

1. **Un solo despliegue:** la aplicación en un contenedor y PostgreSQL.
2. La infraestructura se describe como código, y el entorno local se levanta con un solo comando.
3. Las **migraciones de base de datos** son versionadas, van por módulo y se aplican automáticamente en el despliegue.
4. **Respaldos automáticos diarios** de la base de datos (excepto los documentos en retención temporal), con una **restauración probada** al menos una vez por trimestre.
5. **Observabilidad básica:** logs estructurados, verificación de salud (*health check*) y alerta a Telegram si fallan un trabajo programado o un despliegue.
6. Los trabajos programados (como revisar vencimientos o enviar el resumen semanal) son **idempotentes**: ejecutarlos dos veces no duplica avisos ni registros.

---

## Restricciones tecnológicas

| Tema | Decisión | Estado |
|---|---|---|
| Arquitectura | Monolito modular | ✅ Decidido |
| Metodología | Desarrollo guiado por especificaciones | ✅ Decidido |
| Cliente | PWA *mobile-first* (instalable, cámara, notificaciones) | ✅ Decidido |
| Base de datos | PostgreSQL, un esquema por módulo | ✅ Decidido |
| Almacenamiento de archivos | Ninguno permanente en la primera versión; retención temporal de hasta 7 días en PostgreSQL | ✅ Decidido |
| Notificaciones | Bot de Telegram y correo; WhatsApp solo con enlaces `wa.me` | ✅ Decidido |
| Lenguaje y framework del backend | .NET, TypeScript (NestJS) o Python (FastAPI) | ⏳ Pendiente (ADR-001) |
| Framework del frontend | Por definir | ⏳ Pendiente (ADR-002) |
| Herramienta de especificaciones | OpenSpec o Spec Kit | ⏳ Pendiente (ADR-003) |
| Bus de eventos interno | En el proceso; *outbox* si hace falta garantizar la entrega | ⏳ Pendiente (ADR-004) |
| Proveedor de IA para extracción | Por definir, detrás de un puerto | ⏳ Pendiente (ADR-005) |
| Hosting | VPS o contenedor administrado | ⏳ Pendiente (ADR-006) |

---

## Gobernanza

1. Esta constitución **tiene prioridad** sobre cualquier especificación, plan o convención.
2. Cada cambio a la constitución se hace mediante un **ADR** (*Architecture Decision Record*) en `docs/adr/`, que explica el contexto, la decisión y sus consecuencias.
3. **Versionado semántico** de este documento:
   - **MAYOR:** se elimina o redefine un principio.
   - **MENOR:** se agrega un principio o se amplía uno existente.
   - **PARCHE:** aclaraciones y redacción.
4. Cada plan de implementación incluye una sección **"Revisión de la constitución"** que confirma que cumple cada principio o justifica la excepción.
5. Revisión general de esta constitución al cerrar cada versión mayor de la app.

---

## Glosario

| Término | Significado |
|---|---|
| **Recibo** | Estado de cuenta de un servicio para un periodo (luz, agua, internet, gas, predial, condominio). |
| **Domiciliado** | Recibo que se cobra automáticamente a una cuenta o tarjeta del dueño. |
| **Estancia** | Periodo en que un inquilino ocupa la propiedad, ya sea vacacional o mensual. |
| **Evento** | Acción del administrador que genera un cobro: check-in, limpieza, gestión, etc. |
| **Gasto** | Pago que hace el administrador y que el dueño debe reembolsar. |
| **Liquidación** | Saldo entre dueño y administrador: rentas cobradas, eventos, gastos y transferencias. |
| **Por confirmar** | Estado de un dato extraído con IA que todavía no revisa una persona. |
