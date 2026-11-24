# ADR-0010: Servidor en DigitalOcean con 1 GiB y criterio de salida

- **Estado:** Aceptado
- **Fecha:** 2026-11-24
- **Principios relacionados:** VIII (seguridad y privacidad), X (operación simple y reproducible)

> Un ADR aceptado **no se edita**. Si la decisión cambia, se escribe uno nuevo que lo reemplaza y aquí solo se actualiza el estado.

## Contexto

El [ADR-0007](0007-hosting-y-entorno-local.md) eligió un Hetzner Cloud CX23 en Núremberg, con Helsinki como alternativa y DigitalOcean como plan B. Al revisar la consola de Hetzner el 2026-11-24, el CX23 no estaba disponible, así que se activa el plan B.

El plan B quedó en una sola línea (DigitalOcean Basic de 2 GB en EE. UU., $12 USD al mes), pero varios puntos del ADR-0007 son propios de Hetzner: el tamaño del servidor, el proveedor de OpenTofu, la consola de emergencia, el firewall, el costo y la latencia. Este ADR los reemplaza. **Todo lo demás del ADR-0007 sigue vigente**, y el nombre del subdominio lo decide el [ADR-0009](0009-subdominio-fuera-del-repositorio.md).

Además, DigitalOcean ofrece un plan de 1 GiB por la mitad de precio. Los planes de DigitalOcean se pueden agrandar cambiando solo la CPU y la RAM, sin tocar el disco, y regresar después al plan chico. Eso permite empezar con el plan pequeño y subir si hace falta.

## Decisión

El servidor es un **Droplet Basic de DigitalOcean** (CPU Regular) de **1 GiB de RAM, 1 vCPU y 25 GiB de disco**, por **$6 USD al mes**, en **San Francisco (`sfo3`)**. Para que todo quepa en 1 GiB hay una configuración mínima de memoria y un **criterio de salida**: si la memoria no alcanza, se cambia al plan de 2 GiB sin tocar el disco.

Este ADR reemplaza, del ADR-0007, el resumen de la decisión (proveedor y costo) y los puntos 1, 2, 4 (solo el proveedor del servidor), 14, 15 y 21, además de la consecuencia sobre la latencia.

### Detalle

#### Servidor (reemplaza los puntos 1 y 2 del ADR-0007)

1. **Droplet Basic con CPU Regular:** 1 GiB de RAM, 1 vCPU, 25 GiB de SSD y 1,000 GiB de transferencia, por $6.00 USD al mes más impuestos si aplican (precio del portal de DigitalOcean el 2026-11-24).
2. **Región:** San Francisco (`sfo3`), o Nueva York (`nyc3`) si no hay disponibilidad. DigitalOcean no tiene centros de datos en México ni en Latinoamérica. Los usuarios entran por Cloudflare en México, y Cloudflare llega al servidor por su propia red.
3. **Sin los respaldos de DigitalOcean** (cuestan 20% más): los respaldos van cifrados a R2 (ADR-0007, puntos 26 a 30).
4. **Mudarse a otro proveedor** sigue igual que en el ADR-0007: cambiar el módulo de OpenTofu del servidor, aplicar los mismos *playbooks*, desplegar la misma imagen y restaurar el último respaldo.

#### Infraestructura como código (reemplaza solo el proveedor del punto 4 del ADR-0007)

5. **OpenTofu** declara el Droplet, su firewall y sus alertas con el proveedor de DigitalOcean, en el módulo del servidor. Los recursos de Cloudflare no cambian.
   - `resize_disk = false` desde el principio. Su valor por defecto es `true`: si solo se cambia el tamaño, el disco también crece y el cambio ya no se puede revertir.
   - `monitoring = true`, para instalar el agente de métricas (su valor por defecto es `false`).
   - **Cambiar `user_data` (el `cloud-init`) destruye y vuelve a crear el Droplet.** Por eso el `cloud-init` se mantiene mínimo, OpenTofu ignora sus cambios después de la creación (`ignore_changes`) y todo lo demás se configura con Ansible.
6. El `cloud-init` mínimo, Ansible y el estado cifrado de OpenTofu siguen igual que en el ADR-0007.

#### Memoria en 1 GiB

7. **Swap comprimida en RAM (`zram`)**, de 512 MB con compresión `zstd`, configurada por Ansible. No escribe en el SSD. Con la compresión, guarda en la práctica unos 1 a 1.5 GB de memoria poco usada. Es un colchón para los picos, no memoria extra, y comprimir usa algo de CPU en esos momentos.
   - **No se usa *swap* en disco:** los tutoriales de DigitalOcean desaconsejan habilitarla en sus SSD, porque desgasta un hardware que comparten varios clientes, y recomiendan mejor subir de tamaño.
8. **Límites por contenedor** en el `docker-compose.yml`, con `mem_limit`. Valores iniciales: `api` 384 MB, `postgres` 256 MB y `cloudflared` 64 MB; el contenedor de migración usa el mismo límite que `api`. Se ajustan con mediciones reales.
9. **Prioridad ante falta de memoria:** con `oom_score_adj`, el sistema mata primero el contenedor de migración y nunca a PostgreSQL.
10. **PostgreSQL ajustado** con un archivo de configuración versionado: `shared_buffers=64MB`, `work_mem=2MB`, `maintenance_work_mem=32MB` y `max_connections=20`. El *pool* de conexiones de la aplicación se limita para no rebasar ese máximo.

#### Despliegue (reemplaza el punto 21 del ADR-0007)

11. Para no juntar dos procesos de .NET y `pg_dump` en 1 GiB, el despliegue va **en secuencia**:
    1. la CI se une a la red de Tailscale como **nodo temporal** y entra al servidor por SSH, con un usuario dedicado a desplegar;
    2. descarga la imagen nueva;
    3. **detiene la API**;
    4. saca un **respaldo de la base**;
    5. corre la imagen nueva en **modo "solo migrar"**;
    6. si las migraciones terminan bien, arranca la API nueva; si fallan, **vuelve a arrancar la versión anterior**;
    7. revisa la verificación de salud desde dentro del servidor (ADR-0009, punto 5);
    8. si cualquier paso falla, **avisa por Telegram** (X.5).
12. La interrupción por despliegue pasa de unos segundos a **unos 30 a 60 segundos**. Con dos usuarios es aceptable.

#### Acceso de emergencia (reemplaza el punto 14 del ADR-0007)

13. **Recovery Console de DigitalOcean:** funciona aunque fallen la red o SSH, pero pide contraseña. La Droplet Console no sirve como respaldo, porque entra por SSH a través de la red.
14. **Contraseña solo para `root`:** larga, aleatoria y guardada en el gestor de contraseñas del dueño. Ansible la configura desde un archivo de variables **no versionado**. Solo sirve en la Recovery Console: SSH sigue aceptando únicamente llaves y no deja entrar como `root` (ADR-0007, punto 13). Ansible sigue usando el usuario de administración con `sudo` sin contraseña.
15. **Verificación en dos pasos** obligatoria en la cuenta de DigitalOcean, porque quien entre a ella tiene acceso total al servidor.

#### Firewall (reemplaza el punto 15 del ADR-0007)

16. **Cloud Firewall de DigitalOcean** (sin costo), que niega toda conexión de entrada y bloquea el tráfico antes de que llegue al servidor. `ufw` en el servidor sigue como segunda capa. Tailscale y `cloudflared` solo abren conexiones de salida.

#### Monitoreo de memoria y criterio de salida

17. **Agente de métricas de DigitalOcean** (`do-agent`, sin costo), con alertas por correo cuando la memoria o el disco pasan del 80%.
18. **Aviso cuando el sistema mata un proceso por falta de memoria:** un *timer* revisa el *journal* y avisa por Telegram.
19. **Medición antes de producción:** el `docker-compose.yml` de producción se prueba en local con los mismos límites, y se mide con `docker stats` durante un despliegue completo.
20. **Criterio de salida por memoria:** durante el **primer mes** en producción, si el uso de memoria pasa del **85% de la RAM** en un pico, o si el sistema **mata algún proceso** por falta de memoria, se cambia al plan de **2 GiB** cambiando **solo CPU y RAM** (`resize_disk = false`).
    - Se saca un *snapshot* antes, el servidor se apaga unos minutos y en OpenTofu solo cambia el tamaño.
    - El disco se queda en 25 GiB, así que se puede regresar al plan de 1 GiB.
    - El costo sube a $12 USD al mes, porque mantener el disco no reduce el precio del plan.
21. **Criterio de salida por disco:** en cualquier momento, si el disco pasa del **80%** y la limpieza normal (imágenes viejas y logs) no lo baja, también se agranda el disco al del plan de 2 GiB (50 GiB), con `resize_disk = true` en ese cambio.
    - Cuesta lo mismo ($12 USD al mes), pero **ya no se puede regresar** a 1 GiB, porque un disco no se puede achicar.
    - Si en ese momento el servidor todavía está en 1 GiB, el cambio sube a la vez la memoria y el disco.
22. **Operación por temporadas como opción futura:** si se aplica cualquiera de los dos criterios de salida y el costo sube a $12 USD al mes, se evalúa en un **ADR nuevo** la opción de destruir el Droplet fuera de temporada y volver a crearlo desde los respaldos. Destruirlo es la única forma de dejar de pagar: DigitalOcean cobra igual un Droplet apagado. Ese ADR debe considerar que los recibos, los avisos de vencimiento y el mantenimiento siguen todo el año, y que Telegram descarta los mensajes pendientes de un bot después de unas 24 horas. Hoy no se decide nada.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **Esperar a que haya CX23 en Hetzner** | Retrasa el esqueleto sin fecha conocida. |
| **Droplet de 2 GiB ($12 USD al mes)** | Sin ajustes de memoria y con despliegues de segundos, pero cuesta el doble. Como se puede subir a 2 GiB sin tocar el disco, empezar en 1 GiB no cierra esa puerta. En cambio, un Droplet creado en 2 GiB tiene 50 GiB de disco y ya no se puede bajar a 1 GiB. |
| **Droplet de 1 GiB con todos los ajustes** (también código pregenerado de Wolverine y limpieza de disco agresiva) | Más trabajo antes de saber si hace falta. Se agregan solo si las mediciones lo piden. |
| **Droplet de 512 MiB ($4 USD al mes)** | No alcanza ni para arrancar la API, PostgreSQL y `cloudflared` juntos. |
| **Nueva York (`nyc3`) como región principal** | Es la región más grande, pero la latencia es parecida; queda como respaldo. |
| **Toronto (`tor1`)** | Sin ventaja para usuarios en México. |
| **Atlanta, Richmond o Kansas City** | Solo ofrecen CPUs Premium, que son más caras. |
| **Archivo de *swap* en disco** | Más simple, pero DigitalOcean desaconseja la *swap* en sus SSD. |
| **Sin *swap*** | Es lo que sugiere DigitalOcean, pero sin colchón el primer pico de un despliegue podría matar un contenedor antes de que se aplique el criterio de salida. |
| **Sin contraseña en el servidor**, usando "Reset root password" en una emergencia | Mantiene la regla de "sin contraseñas", pero obliga a reiniciar, lo que puede borrar pistas del problema, y depende de que ese mecanismo funcione cuando algo está roto. |
| **Droplet Console como acceso de emergencia** | Entra por SSH a través de la red, así que falla justo cuando falla la red o SSH. |

## Consecuencias

**Positivas**

- **$6 USD al mes**, la mitad del plan B original.
- **Menor latencia** que desde Europa, porque el servidor queda en Norteamérica.
- Subir de tamaño es un cambio de una línea en OpenTofu y unos minutos sin servicio, y se puede revertir.
- Se mantiene la meta de **cero puertos abiertos**.

**Negativas o riesgos**

- **Memoria justa:** si `zram` se usa mucho, la compresión consume CPU y la aplicación se vuelve lenta. El criterio de salida acota ese riesgo al primer mes.
- **Despliegues con más interrupción** (30 a 60 segundos), y un regreso de versión que depende de volver a arrancar la versión anterior en lugar de que siga funcionando.
- **Más configuración** que mantener: límites, *swap*, ajustes de PostgreSQL, alertas y el *timer* de memoria.
- **Hay una contraseña en el servidor**, aunque solo sirve desde la consola de DigitalOcean.
- **El acceso a la cuenta de DigitalOcean equivale al acceso al servidor.** **Mitigación:** verificación en dos pasos.
- **No queda un plan B con nombre.** Mudarse sigue siendo fácil, y Hetzner se puede reconsiderar en un ADR nuevo si vuelve a haber CX23.

## Listo para aceptar

- [x] **Evidencia:** precios tomados del portal de DigitalOcean el 2026-11-24. En la documentación oficial (revisada el 2026-11-24) se verificó lo siguiente:
  - el cambio de solo CPU y RAM, y que se puede revertir;
  - la disponibilidad del Droplet Basic con CPU Regular por región;
  - el Cloud Firewall, sin costo y manejable desde OpenTofu (`digitalocean_firewall`);
  - en el proveedor de OpenTofu, que `resize_disk` y `monitoring` tienen los valores por defecto indicados en el punto 5, y que cambiar `user_data` recrea el Droplet;
  - la Recovery Console y su requisito de contraseña;
  - las alertas sin costo del agente de métricas;
  - la recomendación de DigitalOcean contra la *swap* en disco (la encontró el dueño en el sitio de DigitalOcean).
- [x] **Alternativas:** se compararon tamaños, regiones, opciones de *swap* y formas de acceso de emergencia.
- [x] **Acuerdo:** el 2026-11-24 el dueño eligió el tamaño, la región y el acceso de emergencia, y eligió `zram` (punto 7) y aprobó los límites de los contenedores (punto 8), los valores de PostgreSQL (punto 10) y el criterio de salida por disco (punto 21). Aprobó este documento el mismo día.
- [x] **Documentación:** no hay `[POR DEFINIR]`.
- [x] **Implementación y revisión:** se implementa al armar el esqueleto (módulo de OpenTofu del servidor, *playbooks* de Ansible, `docker-compose.yml` y flujo de despliegue). Se revisa al terminar el primer mes en producción, con los criterios de salida de los puntos 20 y 21; si se aplica alguno, también se evalúa la opción del punto 22.

## Decisiones derivadas

- **ADR-0007:** su estado indica qué puntos reemplaza este ADR.
- **Constitución:** la fila *Hosting* de *Restricciones tecnológicas* cambia "VPS de Hetzner" por "VPS de DigitalOcean" cuando este ADR se acepte.
- **`.gitignore`:** debe excluir el archivo de variables de Ansible con la contraseña de `root` (junto con lo que pide el ADR-0009).
