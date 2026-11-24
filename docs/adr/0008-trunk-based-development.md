# ADR-0008: Trunk-based development con ramas cortas e interruptores de funciones solo cuando hacen falta

- **Estado:** Aceptado
- **Fecha:** 2026-11-23
- **Principios relacionados:** I (desarrollo guiado por especificaciones), VII (pruebas)

> Un ADR aceptado **no se edita**. Si la decisión cambia, se escribe uno nuevo que lo reemplaza y aquí solo se actualiza el estado.

## Contexto

El flujo de especificaciones pide una rama y un PR por punto de control, pero no dice cuánto vive una rama ni qué pasa con las funciones a medias en `main`. El ADR-0007 decidió desplegar a producción al crear una **etiqueta de versión**, así que cada etiqueta publica todo lo que haya en `main`.

Restricciones:

- Un cambio grande se integra a `main` en **varios puntos de control** (por ejemplo, contratos primero y la interfaz al final), así que `main` puede tener funciones incompletas.
- **El código nuevo no se integra a la rama principal si las pruebas fallan** (Principio VII.4).
- Gran parte del código lo escribe un agente de IA, así que conviene un punto de revisión antes de integrar.
- Un solo desarrollador: no hay conflictos entre equipos, pero las ramas largas igual se desalinean de `main`.

## Decisión

Se usa **trunk-based development con ramas cortas**: cada rama vive **uno o dos días como máximo** y se integra a `main` por PR con la CI en verde. Las funciones a medias se esconden **dejando la interfaz para el final**; solo los cambios que modifican comportamiento visible llevan un **interruptor de funciones**.

### Detalle

#### Ramas e integración

1. **`main` es el tronco** y siempre debe poder desplegarse: compila, las pruebas pasan y lo incompleto no se ve.
2. **Una rama por punto de control,** como ya pide el flujo de especificaciones, con nombre `<change>/<checkpoint>`. **Vive uno o dos días como máximo.** Si un punto de control no cabe en ese tiempo, se divide.
3. **Todo entra por PR, con la CI en verde.** `main` se protege para que no acepte cambios directos ni PRs con la CI en rojo (VII.4).
4. **El PR se integra con *merge*, con commit de integración.** No se usan *squash* ni *rebase* al integrar.
   - Se conserva **un commit por tarea**, como pide el flujo de especificaciones.
   - Cada punto de control queda marcado por su commit de integración. `git log --first-parent main` muestra la lista de puntos de control, y `git log` completo muestra cada tarea.
   - Deshacer un punto de control completo es revertir su commit de integración (`git revert -m 1`).
5. **Rebasear solo lo que no se ha subido.** Los commits locales se pueden rebasear libremente para dejarlos limpios. Una vez subida, **una rama no se rebasea ni se fuerza**, porque tanto el dueño como el agente trabajan sobre ella y una reescritura desincronizaría sus copias.
6. **Antes de integrar, la rama debe estar al día con `main`** (regla de protección de GitHub), para que la CI pruebe la combinación real y no dos cambios que pasan por separado pero se rompen juntos. Para ponerla al día se trae `main` a la rama con un *merge*.
7. **Conflictos en migraciones de EF Core:** si dos ramas agregan migraciones al mismo módulo, no se mezcla la instantánea del modelo a mano. Se borra la migración de la rama y se vuelve a generar después de traer `main`.
8. **Los puntos de control de un mismo cambio van en secuencia:** la rama del siguiente sale de `main` después de integrar el anterior. No se encadenan ramas.
9. Los commits de pruebas en rojo siguen quedándose locales hasta que la tarea las pone en verde (flujo de especificaciones), así que la rama nunca sube con la CI en rojo.

#### Funciones a medias

10. **Por defecto, la interfaz va al final.** Mientras un cambio no termina, la PWA no muestra ningún botón ni pantalla para él. Su API puede existir, porque solo la usan usuarios autenticados.
11. **Interruptor de funciones solo si el cambio modifica comportamiento que ya se ve,** por ejemplo un cálculo de prorrateo nuevo o una regla de confirmación distinta, que no se puede esconder quitando un botón.
   - Se implementan con **`Microsoft.FeatureManagement`**, leyendo la configuración. Encender o apagar un interruptor requiere cambiar la configuración en el servidor y reiniciar la app, pero no redesplegar.
   - En producción, un interruptor nuevo está **apagado** por defecto.
   - Las pruebas cubren **los dos caminos**, encendido y apagado, mientras el interruptor exista.
   - **Se borra al cerrar el cambio:** en su último punto de control, o como máximo en el siguiente cambio. Un interruptor olvidado es deuda técnica.
12. El `design.md` de cada cambio dice cómo se esconde lo incompleto: si basta con dejar la interfaz al final o si hace falta un interruptor, y cuál.

#### Versiones

13. Se despliega creando una etiqueta de versión desde `main` (ADR-0007). Normalmente se etiqueta al cerrar un cambio completo, pero como `main` siempre puede desplegarse, se puede etiquetar en cualquier momento (por ejemplo, para una corrección urgente).
14. **Versionado semántico:** `v0.x.y` mientras la app no tenga su primera versión estable.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **Commits directos a `main`, sin ramas ni PRs** | La CI correría después de que el código ya está en `main`, así que cumplir VII.4 dependería de revisiones locales. Además se pierde el PR como punto de revisión del trabajo del agente. |
| **Ramas de función largas** (una rama por cambio completo) | Se desalinean de `main` y se integran todas de golpe al final. Es lo que trunk-based development busca evitar. |
| ***Rebase* para poner al día y *squash* para integrar** (práctica común de trunk-based development) | Da un `main` lineal y compacto, pero reescribe ramas ya subidas, en las que trabajan tanto el dueño como el agente, y deja un solo commit por punto de control en `main`, en lugar de uno por tarea con su ID de escenario. |
| ***Rebase* para poner al día y para integrar, sin *squash*** | `main` lineal y con un commit por tarea, pero solo es seguro si una sola persona hace push a cada rama de PR. Aquí escriben tanto el dueño como el agente. |
| ***Squash* al integrar** | Junta el PR en un solo commit y rompe la regla de un commit por tarea. |
| ***Rebase* al integrar** (historial lineal) | Obliga a rebasear ramas ya subidas y a forzar el *push*, lo que desincroniza las copias del dueño y del agente. `git log --first-parent` da una vista casi igual de limpia con *merge*. |
| **GitFlow** (ramas `develop`, `release` y `hotfix`) | Demasiadas ramas para un desarrollador y despliegues por etiqueta; las etiquetas ya cumplen la función de las ramas `release`. |
| **Solo dejar la interfaz al final** | No sirve para cambios que modifican comportamiento visible, como un cálculo de dinero. |
| **Interruptores para todo cambio** | Más código y pruebas duplicadas, cuando la mayoría de los cambios se esconden solos dejando la interfaz al final. |

## Consecuencias

**Positivas**

- `main` siempre puede desplegarse, así que una corrección urgente no espera a que termine otro cambio.
- Ramas cortas: pocas oportunidades de que se desalineen de `main`.
- Ningún commit ya subido se reescribe, y cada punto de control se puede revertir completo con un solo commit.
- La CI y el PR revisan todo antes de integrar (VII.4), incluido el trabajo del agente.
- Los interruptores aparecen solo donde aportan algo: cambios visibles y, sobre todo, cálculos de dinero que conviene poder apagar sin redesplegar.

**Negativas o riesgos**

- **El historial de `main` no es lineal:** hay que usar `git log --first-parent` para ver solo los puntos de control.
- **Dividir el trabajo en pasos de uno o dos días** requiere disciplina al planear los puntos de control.
- **Las APIs incompletas existen en producción** aunque no se vean en la interfaz. Solo las pueden usar usuarios autenticados, pero hay que cuidar que no dejen datos inconsistentes si alguien las llama.
- **Los interruptores son deuda** si no se borran a tiempo. **Mitigación:** el `design.md` del cambio dice cuándo se quita, y el control de cierre lo revisa.

## Listo para aceptar

- [x] **Evidencia:** trunk-based development con ramas cortas es una práctica establecida. Las piezas que usa son funciones estándar de GitHub (protección de rama, exigir que la rama esté al día, habilitar solo la integración con *merge*) y una librería oficial de Microsoft (`Microsoft.FeatureManagement`).
- [x] **Alternativas:** se compararon commits directos, ramas largas, GitFlow y tres formas de integrar (*squash*, *rebase*, y *rebase* + *squash*), además de tres formas de esconder lo incompleto.
- [x] **Acuerdo:** el dueño eligió cada punto durante la sesión del 2026-11-23.
- [x] **Documentación:** no hay `[POR DEFINIR]`. Las reglas operativas están también en el flujo de especificaciones.
- [x] **Implementación y revisión:** la protección de `main` se configura al armar el esqueleto, y las reglas aplican desde su primera rama. Se revisa en la retrospectiva del primer hito (Gobernanza 6), sobre todo si las ramas de uno o dos días resultaron realistas y si hizo falta algún interruptor.

## Decisiones derivadas

- **Flujo de especificaciones:** actualizar *Puntos de control, ramas y commits* con la duración de las ramas, la protección de `main` y la forma de esconder lo incompleto, y la plantilla de `design.md` con la sección correspondiente.
- **GitHub:** al armar el esqueleto, proteger `main` (solo por PR, CI en verde, rama al día con `main`) y dejar habilitada solo la integración con *merge*.
