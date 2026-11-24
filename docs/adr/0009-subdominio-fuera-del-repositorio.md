# ADR-0009: Subdominio de producción fuera del repositorio

- **Estado:** Aceptado
- **Fecha:** 2026-11-24
- **Principios relacionados:** VIII (seguridad y privacidad), X (operación simple y reproducible)

> Un ADR aceptado **no se edita**. Si la decisión cambia, se escribe uno nuevo que lo reemplaza y aquí solo se actualiza el estado.

## Contexto

El repositorio es **público**. El [ADR-0007](0007-hosting-y-entorno-local.md) (punto 10) fijó el nombre del subdominio de la aplicación en el documento y, en el mismo punto, usó el certificado comodín de Cloudflare para que el subdominio **no aparezca** en los registros públicos de transparencia de certificados. Al escribir el nombre en un documento público, esa medida se pierde: cualquiera que lea el repositorio conoce la dirección de la aplicación.

Ocultar el nombre no sustituye la autenticación (VIII.3), pero es una capa más: reduce los escaneos y los intentos automáticos contra la aplicación. Como todavía no hay nada desplegado, cambiar el nombre no cuesta nada. El repositorio también mencionaba la ciudad de la propiedad y la del dueño, que se quitan con el mismo procedimiento.

## Decisión

El subdominio de producción se elige con un nombre **difícil de adivinar** y **no se escribe en el repositorio**: ni en documentos, ni en código, ni en configuración versionada, ni en los logs de la CI. El nombre anterior se abandona, porque ya estuvo publicado. Este ADR reemplaza **solo el nombre** del punto 10 del ADR-0007; el resto de ese punto sigue vigente.

### Detalle

1. **Elección del nombre:** la hace el dueño, fuera del repositorio y fuera de las conversaciones con agentes. El nombre no usa palabras relacionadas con el proyecto (como `rentas`, `renta`, `depa`, `admin`, `app` o el nombre de la ciudad) ni nombres comunes de las listas de enumeración de subdominios. Una palabra o combinación sin relación con el proyecto, o una cadena aleatoria corta, cumple.
2. **Única fuente:** el gestor de contraseñas del dueño, junto con los demás secretos de la infraestructura.
3. **OpenTofu:** el nombre entra como variable desde un archivo `*.tfvars` **no versionado**. El repositorio solo incluye un ejemplo con un valor ficticio. El nombre también queda en el estado, que ya va cifrado (ADR-0007, punto 4).
4. **Ansible:** si la aplicación necesita conocer su URL pública (por ejemplo, para poner enlaces en los mensajes de Telegram), el valor viaja en un archivo de variables **no versionado** y se escribe en el archivo de secretos del servidor, igual que los demás secretos (VIII.4).
5. **Verificación de salud después de desplegar** (ADR-0007, punto 21.6): la CI la revisa **desde dentro del servidor**, por SSH, así que no necesita conocer el subdominio. El camino público lo cubre UptimeRobot (ADR-0007, punto 31), que se configura a mano en su consola.
6. **Documentos:** cuando haga falta mencionar el subdominio, se escribe `<subdominio>.saulgmx.com`.
7. **El nombre anterior no se usa:** no tiene registro DNS ni redirección, para que no sirva de pista hacia el nombre nuevo.
8. **La historia de git se reescribe una sola vez** con `git filter-repo`, para quitar el nombre anterior y las ciudades de todos los commits, y se publica con un *force push* a `main`. Es una **excepción única** a la regla del [ADR-0008](0008-trunk-based-development.md) de no rebasear ramas publicadas. Se permite porque el repositorio tiene un día, no tiene *forks* y solo lo usa el dueño. Los otros clones se vuelven a clonar. Los commits anteriores pueden seguir accesibles en GitHub por su *hash* hasta que GitHub los limpie; si hace falta, se pide a soporte de GitHub que los borre.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **Mantener público el nombre anterior** | Sin costo, pero anula a propósito la medida del certificado comodín, y el nombre es de los primeros que prueba una enumeración de subdominios. |
| **Guardar el nombre como secreto de GitHub** para que la CI revise la salud por el camino público | Funciona, porque GitHub oculta los secretos en los logs, pero agrega un secreto más a la CI cuando la revisión desde el servidor basta y UptimeRobot ya cubre el camino público. |
| **No reescribir la historia** y solo quitar los datos del estado actual | Los datos dejan de verse a simple vista, pero siguen en la historia para quien la busque. |
| **Repositorio privado** | Ocultaría todo, pero en una cuenta personal se pierden funciones gratuitas de los repositorios públicos (como la protección de *push* contra secretos) y se limitan los minutos de GitHub Actions. |
| **Cloudflare Access delante de la aplicación** | Es un control real y no solo ocultamiento, pero obliga a iniciar sesión dos veces, y las sesiones de Access complican la PWA y su *service worker*. Va en contra del perfil tecnológico básico del administrador. |

## Consecuencias

**Positivas**

- Se recupera la protección del certificado comodín: el subdominio no aparece ni en los registros de certificados ni en el repositorio.
- Menos tráfico automatizado contra la aplicación.
- La historia publicada ya no contiene el subdominio ni las ciudades.
- La CI no necesita conocer el subdominio.

**Negativas o riesgos**

- **Es ocultamiento, no un control.** El nombre va a circular en enlaces de Telegram y de correo, en el historial del navegador y en Cloudflare. La protección real sigue siendo la autenticación (VIII.3).
- **Un dato más que no vive en el repositorio:** si se pierde el gestor de contraseñas, el nombre se recupera desde el panel de Cloudflare.
- Quien arme la infraestructura en otro equipo necesita crear los archivos no versionados a partir de los ejemplos.
- **Reescribir la historia cambia los *hash* de todos los commits:** los clones existentes quedan inservibles y se vuelven a clonar.

## Listo para aceptar

- [x] **Evidencia:** el ADR-0007 ya verificó que el certificado comodín de Cloudflare cubre el subdominio sin publicarlo en los registros de certificados. Las variables de OpenTofu desde `*.tfvars` y los archivos de variables de Ansible son funciones estándar.
- [x] **Alternativas:** se compararon cinco opciones, con el motivo de cada descarte.
- [x] **Acuerdo:** el 2026-11-24 el dueño eligió cambiar el subdominio, reescribir la historia y quitar las ciudades, y aprobó este documento.
- [x] **Documentación:** no hay `[POR DEFINIR]`. El nombre real queda fuera del repositorio a propósito y no es un pendiente.
- [x] **Implementación y revisión:** se implementa al armar el esqueleto (módulos de OpenTofu, *playbooks* de Ansible y flujo de despliegue). Se revisa si se adopta Cloudflare Access o si el repositorio deja de ser público.

## Decisiones derivadas

- **ADR-0007:** su estado indica que el nombre del subdominio fue reemplazado por este ADR.
- **`.gitignore`:** debe excluir los archivos `*.tfvars` y los de variables de Ansible con valores reales antes de crear la infraestructura.
- **Handoff:** sin el nombre anterior.
- **Constitución y `CLAUDE.md`:** un ADR aceptado se puede editar solo para quitar datos sensibles, y los documentos no nombran la ubicación de la propiedad ni de las personas; la zona horaria es un dato de cada propiedad y de cada usuario.
