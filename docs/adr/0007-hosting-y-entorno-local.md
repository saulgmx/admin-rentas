# ADR-0007: Hosting en un VPS con Docker Compose y Cloudflare Tunnel, y entorno local con Aspire

- **Estado:** Aceptado
- **Fecha:** 2026-11-23
- **Principios relacionados:** VIII (seguridad y privacidad), IX (integraciones externas), X (operación simple y reproducible)

> Un ADR aceptado **no se edita**. Si la decisión cambia, se escribe uno nuevo que lo reemplaza y aquí solo se actualiza el estado.

## Contexto

Hay que decidir dónde corre la aplicación en producción, cómo se despliega, cómo se protege y respalda, y cómo se levanta el entorno local.

Restricciones:

- **Un solo despliegue:** un contenedor con la API, que también sirve la PWA (ADR-0002), y PostgreSQL (Principio X.1).
- **Proceso siempre encendido:** Wolverine procesa colas durables en segundo plano (ADR-0005), hay trabajos programados diarios y semanales, y `IMailbox` revisa el correo. Un servicio que apaga el contenedor cuando no hay tráfico detendría todo eso.
- **HTTPS con dominio:** la PWA lo necesita para instalarse y usar el *service worker*.
- **Infraestructura como código** y entorno local con un solo comando (X.2).
- **Migraciones** versionadas, por módulo y automáticas en el despliegue (X.3).
- **Respaldos diarios** (sin los documentos en retención temporal) y una **restauración probada** al menos cada trimestre (X.4).
- **Verificación de salud** y **alerta por Telegram** si falla un trabajo programado o un despliegue (X.5).
- **Carga mínima:** dos usuarios en México y decenas de eventos al día. La latencia no es un criterio fuerte.
- **Presupuesto bajo:** es un proyecto personal pequeño.
- El dueño ya tiene el dominio `saulgmx.com` con DNS en Cloudflare (registros en nube gris).

## Decisión

La aplicación corre en **un VPS pequeño de Hetzner en Europa**, con **Docker Compose**. Los usuarios entran por **Cloudflare Tunnel**, sin puertos abiertos, y la administración es por **Tailscale**. Se despliega con **GitHub Actions** al crear una **etiqueta de versión**. Los respaldos van **cifrados a Cloudflare R2**, y el monitoreo usa **UptimeRobot** y **Healthchecks.io**. En local, el entorno se levanta con **Aspire**.

Costo aproximado: **unos €6 al mes** (más impuestos si aplican); todo lo demás cabe en planes gratuitos.

### Detalle

#### Servidor

1. **Hetzner Cloud CX23:** 2 vCPU, 4 GB de RAM, 40 GB NVMe y 20 TB de tráfico, por €5.99 al mes sin impuestos (precio publicado el 2026-11-23). Ubicación: Núremberg (`nbg1`), o Helsinki (`hel1`) si no hay disponibilidad.
2. **Plan B:** DigitalOcean Basic de 2 GB en EE. UU. ($12 USD al mes), si Hetzner deja de convenir. Mudarse es cambiar el módulo de OpenTofu del servidor, aplicar los mismos *playbooks* de Ansible, desplegar la misma imagen y restaurar el último respaldo. El DNS no cambia, porque el subdominio apunta al túnel y no a la IP del servidor.
3. **Sistema operativo:** Ubuntu Server LTS vigente.
4. **Infraestructura como código** (X.2), en tres capas versionadas en el repositorio:
   - **OpenTofu** declara los recursos en Hetzner (servidor y firewall) y en Cloudflare (túnel, su configuración de rutas y los buckets de R2). Lo específico de cada proveedor vive en su propio módulo, así que cambiar de proveedor solo toca el módulo del servidor.
   - **`cloud-init` mínimo:** en el primer arranque crea el usuario de administración con su llave SSH y une el servidor a Tailscale con una llave de un solo uso.
   - **Ansible**, por Tailscale, configura el resto: endurecimiento de SSH, `ufw`, actualizaciones automáticas, Docker, los archivos de secretos, el Compose de producción y los *timers* de respaldo. Los *playbooks* se pueden volver a aplicar sin recrear el servidor y no dependen del proveedor.
   - **Estado de OpenTofu:** en un bucket de R2 **privado y separado** del de respaldos, con el *backend* S3 de OpenTofu (`use_lockfile` para el bloqueo, que R2 soporta con escrituras condicionales). El estado y los planes van **cifrados por OpenTofu** (AES-GCM, con una frase de paso guardada en el gestor de contraseñas del dueño), porque contienen secretos como los del túnel.
   - OpenTofu y Ansible **se ejecutan desde el equipo del dueño**, no desde la CI. La infraestructura cambia poco, y así la CI no necesita credenciales de Hetzner ni de Cloudflare.
   - Versiones: OpenTofu estable vigente al armar el esqueleto (1.13.x a la fecha) y proveedor de Cloudflare 5.x, fijados en el repositorio.

#### Servicios en Docker Compose

5. Un archivo `docker-compose.yml` de producción, **escrito a mano** y versionado en el repositorio, con estos servicios:
   - **`api`**: la aplicación .NET, que también sirve la PWA;
   - **`postgres`**: PostgreSQL con un volumen persistente;
   - **`cloudflared`**: el conector del túnel.
6. **Ningún servicio publica puertos en el servidor.** Se comunican por una red interna de Docker. PostgreSQL nunca es accesible desde fuera.
7. Los contenedores corren **sin `root`** y con sistema de archivos de **solo lectura** donde se pueda.
8. **Secretos** (llave de Anthropic, token de Telegram, token del túnel, credenciales de R2 y de correo): en archivos con permisos restringidos en el servidor, fuera del repositorio (VIII.4).
9. La CI valida el archivo con `docker compose config` en cada cambio.

#### Acceso de los usuarios

10. **Cloudflare Tunnel** en **`<subdominio>.saulgmx.com`**. El conector abre una conexión **de salida** hacia Cloudflare, así que el servidor no necesita ningún puerto abierto.
    - El registro del subdominio queda en nube naranja; los demás registros del dominio no cambian.
    - Cloudflare da el HTTPS con su certificado comodín, así que el subdominio no aparece en los registros públicos de transparencia de certificados.
    - El plan gratuito permite subidas de hasta 100 MB por petición, de sobra para el límite de 10 MB por documento (IV.3).
11. **Telegram y correo sin puertos de entrada:** el bot consulta sus mensajes desde el servidor (*long polling*) y `IMailbox` lee el correo con conexiones de salida.

#### Acceso de administración

12. **Tailscale** (red privada): SSH solo escucha dentro de esa red. El puerto 22 no está abierto a internet.
    - La red se crea con una **cuenta personal de dominio público** (Gmail o GitHub), no con una de dominio propio como `saulgmx.com`. Tailscale clasifica así una red como personal y la inscribe en su plan Personal gratuito; una de dominio propio la trata como de negocio.
    - Solo entran el dueño y la CI. El administrador no usa Tailscale.
13. **Solo llaves SSH,** sin contraseñas y sin entrar como `root`.
14. **Consola web de Hetzner** como acceso de emergencia si Tailscale falla.
15. **Firewall de Hetzner** que niega toda conexión de entrada, y `ufw` en el servidor como segunda capa.
16. **Actualizaciones de seguridad automáticas** del sistema operativo (`unattended-upgrades`).

#### Entorno local

17. **Aspire**: un proyecto *AppHost* levanta PostgreSQL y la API con un comando (`aspire run`), con el panel de logs, trazas y métricas (X.2). Versión mayor estable vigente al armar el esqueleto (17.x a la fecha, que exige el SDK de .NET 10 para el *AppHost* e incluye la integración de PostgreSQL).
18. Aspire **solo se usa en local**. El `docker-compose.yml` de producción no se genera desde Aspire.

#### Despliegue

19. **GitHub Actions** construye la imagen y la publica en **GitHub Container Registry**, etiquetada con la versión y el *hash* del commit.
20. **Disparadores:**
    - crear una **etiqueta de versión** (`v*`) despliega esa versión;
    - un **botón manual** (`workflow_dispatch`) permite volver a desplegar una etiqueta existente, incluida una anterior para regresar de versión.
21. **Pasos del despliegue:**
    1. la CI se une a la red de Tailscale como **nodo temporal** y entra al servidor por SSH, con un usuario dedicado a desplegar;
    2. descarga la imagen nueva;
    3. saca un **respaldo de la base justo antes de migrar**;
    4. corre la imagen en **modo "solo migrar"**;
    5. si las migraciones terminan bien, arranca la API nueva; si fallan, el despliegue se detiene y **la versión anterior sigue funcionando**;
    6. revisa la verificación de salud;
    7. si cualquier paso falla, **avisa por Telegram** (X.5).
22. Durante el cambio de versión puede haber unos segundos sin servicio. Con dos usuarios es aceptable.
23. Cómo se integra el código a `main` y cómo se esconden las funciones a medias: ADR-0008.

#### Migraciones

24. La aplicación tiene un **modo de arranque "solo migrar"**: aplica las migraciones pendientes de cada `DbContext` de módulo y prepara las tablas de Wolverine, y después termina. Cada migración de EF Core corre en una transacción, y en PostgreSQL los cambios de estructura también se revierten si fallan.
25. **Migraciones compatibles con la versión anterior** (patrón "expandir y contraer"): para borrar o renombrar algo, primero se agrega lo nuevo; lo viejo se quita en una versión posterior. Así, regresar a la etiqueta anterior no rompe la aplicación.

#### Respaldos

26. **Respaldo diario** con `pg_dump`, comprimido. **No incluye el contenido de los documentos en retención temporal** (X.4).
27. **Cifrado antes de subirlo** con `age`. El servidor solo tiene la llave pública; la llave privada se guarda fuera del servidor, en el gestor de contraseñas del dueño. Así, ni el almacenamiento ni quien entre al servidor pueden leer los respaldos.
28. **Destino: Cloudflare R2.** El plan gratuito incluye 10 GB al mes y no cobra por descargar.
29. **Retención:** 7 diarios, 4 semanales y 6 mensuales. Los respaldos previos a cada migración se conservan junto con los diarios.
30. **Prueba de restauración automática (X.4):** un flujo programado de GitHub Actions, **cada semana**, descarga el último respaldo, lo descifra y lo restaura en un PostgreSQL temporal. Después corre revisiones básicas (existen los esquemas de los módulos y las tablas tienen datos) y avisa por Telegram si algo falla. La llave privada para esta prueba es un secreto de GitHub.

#### Monitoreo y logs

31. **UptimeRobot** (plan gratuito, cada 5 minutos) consulta la verificación de salud **por el camino público**: Cloudflare, túnel, API y base de datos. Así detecta también una caída del túnel. Su plan gratuito no incluye Telegram ni *webhooks* (según su página, empiezan en planes de pago), así que alerta por **correo**; la constitución exige Telegram solo para trabajos y despliegues.
32. **Healthchecks.io** (plan gratuito, hasta 20 trabajos) funciona como "interruptor de hombre muerto" para el respaldo diario y los trabajos programados: cada uno avisa al terminar, y si no avisa a tiempo salta la alerta por Telegram (X.5). Healthchecks.io tiene integración con Telegram, y su página de precios no limita las integraciones en el plan gratuito. Si al configurarlo resultara de pago, se usa un *webhook* que llama directamente a la API de bots de Telegram.
33. **Logs** estructurados, guardados por Docker con rotación de tamaño, sin servicio externo. Se consultan por SSH.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **`cloud-init` y un script con `hcloud`** | Lo mínimo, pero los recursos de Cloudflare quedarían manuales y cualquier cambio de configuración obligaría a recrear el servidor. Para un dueño con experiencia en Terraform y Ansible, el costo extra de la opción elegida es mínimo. |
| **OpenTofu y `cloud-init`, sin Ansible** | La configuración del servidor solo se aplicaría en el primer arranque. |
| **Terraform en lugar de OpenTofu** | Funcionaría igual, pero su licencia es BSL desde 2023 y no cifra el estado por sí mismo; la protección dependería solo de que el bucket sea privado. |
| **Supabase Pro para PostgreSQL** | Desde $25 USD al mes, de 3 a 5 veces el costo del servidor completo. Sus respaldos no se pueden descargar y restaurarlos reemplaza la base, así que de todos modos haría falta `pg_dump` propio para probar restauraciones. No usaríamos sus extras (Auth, API REST automática), y su API REST podría exponer tablas fuera de la app. Wolverine exigiría conexión directa o en modo sesión, nunca en modo transacción. Su plan gratuito no tiene respaldos y se pausa por inactividad. |
| **Plataforma administrada** (Railway, Render, Fly.io) | Menos mantenimiento, pero de $10 a $30 USD al mes con PostgreSQL administrado, para una carga mínima. |
| **Nube grande** (Azure, AWS, GCP) | Requiere una réplica siempre activa y PostgreSQL administrado: $30–60 USD al mes o más, con más complejidad (IAM, redes). |
| **Hetzner en EE. UU.** (CPX11) | Más cerca de México, pero €17.99 al mes por 2 GB: el triple de precio por la mitad de RAM. |
| **Hetzner CAX11 (ARM)** | Mismos recursos por un poco más, y obliga a construir imágenes ARM en la CI. |
| **DigitalOcean** | Más cerca, pero $12 USD al mes por 1 vCPU y 2 GB. Queda como plan B. |
| **Oracle Cloud Always Free** | $0, pero Oracle puede reclamar instancias gratuitas con poco uso. |
| **Equipo en casa con Cloudflare Tunnel** | $0, pero la app se cae con cada corte de luz o de internet. |
| **Caddy directo** (puertos 80 y 443 abiertos) | Nadie en medio ve el tráfico, pero la IP queda expuesta a escaneos y el subdominio aparece en los registros públicos de certificados. |
| **Cloudflare como proxy + Caddy** | Oculta la IP, pero requiere abrir el 443 a las IPs de Cloudflare y mantener esa lista. El túnel logra lo mismo sin puertos. |
| **App solo por Tailscale** | Lo más seguro, pero el administrador tendría que instalar y mantener Tailscale en su celular, lo que va contra su perfil tecnológico básico. |
| **SSH por Cloudflare Tunnel con Access** | Si el túnel falla, se pierde el acceso justo cuando hace falta repararlo. |
| **SSH abierto solo a una IP** | La IP de casa cambia, y rompe la meta de cero puertos abiertos. |
| **Aspire genera el Compose de producción** | Una sola fuente, pero la función podría seguir experimental en una herramienta que cambia de versión mayor seguido, y los ajustes de seguridad quedarían menos visibles que en un archivo escrito a mano. |
| **Docker Compose para todo, sin Aspire** | Pierde el panel de observabilidad local, y levantar el entorno local requiere dos comandos. |
| **El servidor jala las imágenes nuevas** | Difícil saber si el despliegue funcionó y avisar cuando falla. |
| **Kamal** | Despliegues sin interrupción que no hacen falta con dos usuarios, a cambio de otra herramienta y su propio proxy. |
| **Panel autoalojado** (Coolify, Dokploy) | Consume RAM y agrega otro servicio que proteger. |
| **Desplegar en cada integración a `main`** | Llevaría a producción funciones a medias entre puntos de control. |
| **La API migra al arrancar** | Si una migración falla, la app queda caída. |
| **Ejecutables de migración de EF Core** (*bundles*) | Varios archivos más que construir, copiar y correr, uno por módulo. |
| **Backblaze B2 para respaldos** | Igual de buena, y además independiente de Cloudflare; se prefirió R2 porque la cuenta ya existe. |
| **Hetzner Storage Box para respaldos** | Cuesta lo que R2 da gratis, y queda con el mismo proveedor que el servidor. |
| **Solo Healthchecks.io** | El aviso sale desde dentro del servidor y no detecta una caída del túnel. |
| **Solo UptimeRobot** | No detecta un respaldo que dejó de correr mientras la app sigue arriba. |

## Consecuencias

**Positivas**

- Unos €6 al mes en total.
- **Cero puertos abiertos** a internet: ni web, ni SSH, ni base de datos.
- Mudarse a otro proveedor es fácil: un contenedor y un respaldo de la base.
- Despliegues trazables: cada versión es una etiqueta, regresar es volver a desplegar la anterior, y una migración fallida no tumba la app.
- La restauración se prueba cada semana sin depender de que alguien se acuerde.

**Negativas o riesgos**

- **Cloudflare descifra el tráfico** en su red y ve los recibos que se suben (domicilio y número de cuenta). Es un tercero más con acceso a datos personales, como el proveedor de IA.
- **Concentración en Cloudflare:** DNS, túnel y respaldos. Si se pierde la cuenta, se pierden las tres cosas, aunque el servidor y su base siguen intactos. **Mitigación:** Backblaze B2 queda como alternativa directa para respaldos.
- **Tailscale Personal es "solo para uso no comercial".** Su FAQ define el uso personal por la cuenta con que se crea la red (dominio público) y pone como ejemplo conectarse a un servidor en la nube. Por eso el riesgo es bajo: la red es de una sola persona y la crea con su cuenta personal. Si Tailscale cambia ese criterio, la alternativa es pagar un plan o cambiar de herramienta en un ADR nuevo.
- **El mantenimiento del servidor es nuestro:** actualizaciones del sistema, espacio en disco y seguridad. **Mitigación:** actualizaciones automáticas, cero puertos abiertos y preparación del servidor como código.
- **Un solo servidor:** si falla, la app se cae hasta restaurarla en otro. Es aceptable para dos usuarios, con respaldos externos y probados.
- **Latencia desde México a Europa** (unos 150–180 ms de ida y vuelta). No importa para este uso.
- **Dos descripciones de la infraestructura:** el *AppHost* de Aspire y el Compose de producción pueden desalinearse. **Mitigación:** el Compose es pequeño y la CI lo valida.
- Unos segundos sin servicio en cada despliegue.
- **Si se pierde la frase de paso del estado de OpenTofu,** el estado es irrecuperable. **Mitigación:** guardarla en el gestor de contraseñas; en el peor caso, los recursos se pueden importar de nuevo a un estado nuevo.

## Decisiones derivadas

- **Constitución:** marcar como decidido el hosting en *Restricciones tecnológicas*.
- **ADR-0008:** trunk-based development con ramas cortas, e interruptores de funciones solo para cambios que modifican comportamiento visible.
- **Esqueleto:** *AppHost* de Aspire, `docker-compose.yml` de producción, `Dockerfile`, modo "solo migrar", verificación de salud, flujos de CI, de despliegue y de prueba de restauración.
- **Infraestructura:** módulos de OpenTofu, `cloud-init` y *playbooks* de Ansible, con los pasos para crear el servidor documentados en el repositorio.

## Listo para aceptar

- [x] **Evidencia:** precios y límites verificados el 2026-11-23 en las páginas oficiales (Hetzner, DigitalOcean, Cloudflare Zero Trust y R2, Tailscale, UptimeRobot, Healthchecks.io, Aspire 17 y OpenTofu, incluidos el cifrado del estado y el bloqueo con R2). El proveedor de Cloudflare gestiona túneles y buckets. Pendiente solo de confirmar en la consola: disponibilidad del CX23 en Núremberg; si no hay, Helsinki, y si tampoco, el plan B.
- [x] **Alternativas:** se compararon proveedores, formas de exponer la app, de administrar el servidor, de describir la infraestructura, de desplegar, de migrar, de respaldar y de monitorear.
- [x] **Acuerdo:** el dueño eligió cada punto en la sesión del 2026-11-23.
- [x] **Documentación:** no hay `[POR DEFINIR]`.
- [x] **Implementación y revisión:** se implementa al armar el esqueleto. Se revisa en la retrospectiva del primer hito (costos reales, uso de memoria y si los planes gratuitos siguen alcanzando), o antes si un proveedor cambia precios o condiciones.
