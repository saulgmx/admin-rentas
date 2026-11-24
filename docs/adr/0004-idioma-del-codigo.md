# ADR-0004: El código va en inglés

- **Estado:** Aceptado
- **Fecha:** 2026-11-23
- **Principios relacionados:** I (la especificación va primero), II (monolito modular)
- **Reemplaza parcialmente:** los nombres de proyectos, carpetas y esquemas de [ADR-0001](0001-backend-dotnet.md). La decisión técnica de ADR-0001 sigue vigente.

## Contexto

El mapa de módulos y el ADR-0001 usaban identificadores en español (`Servicios`, `ConfirmarRecibo`, `AdminRentas.Ocupacion`). Hay que decidir el idioma del código antes de escribirlo.

## Decisión

1. **Código, identificadores, commits, ramas y PRs en inglés.**
2. **Documentos y conversación en español:** constitución, ADRs, specs, diseño y tareas.
3. **El glosario es la traducción oficial** entre el español de los documentos y el inglés del código: [`docs/arquitectura/glosario.md`](../arquitectura/glosario.md). Un concepto nuevo se agrega al glosario antes de usarse en el código.
4. **Los IDs de reglas y escenarios son neutrales** (por ejemplo `UTL-R01-S1`), con prefijo por módulo según el glosario, para poder buscarlos con `grep` en documentos y pruebas.
5. El nombre raíz **`AdminRentas`** es el nombre propio del producto y no se traduce.
6. Las carpetas de specs vivas usan el nombre del módulo en inglés (`openspec/specs/utilities/`), aunque su contenido esté en español.

### Estructura de la solución (reemplaza la de ADR-0001)

```
src/
├── Host/
│   └── AdminRentas.Api/
├── Modules/
│   ├── Identity/
│   │   ├── AdminRentas.Identity/
│   │   └── AdminRentas.Identity.Contracts/
│   ├── Properties/
│   ├── Ingestion/
│   ├── Utilities/
│   ├── Occupancy/
│   └── Notifications/
├── Shared/
│   ├── AdminRentas.SharedKernel/
│   └── AdminRentas.Infrastructure/
└── Web/                                  # Frontend (ADR-0002)
tests/
├── AdminRentas.ArchitectureTests/
└── Modules/
    └── AdminRentas.<Module>.Tests/
```

Las reglas de visibilidad y referencias de ADR-0001 aplican igual, con los nombres en inglés (`.Contracts` en lugar de `.Contratos`). El registro de cada módulo queda como `AddUtilitiesModule()`.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **Dominio en español y lo técnico en inglés** (`Recibo`, `ReciboRepository`) | Specs y código usarían las mismas palabras, pero mezcla idiomas dentro de cada identificador y no es consistente con proyectos anteriores. |
| **Todo en español** | Choca con las convenciones del ecosistema, las librerías y los mensajes de las herramientas. |

## Consecuencias

**Positivas**

- Consistente con el ecosistema de .NET y TypeScript, y con gym-app.
- Los IDs neutrales permiten rastrear con `grep` una regla desde la spec hasta su prueba, aunque estén en idiomas distintos.

**Negativas o riesgos**

- Las specs dicen "recibo" y el código `Bill`. **Mitigación:** el glosario es obligatorio, y el `design.md` de cada cambio usa los nombres del código cuando habla de contratos y eventos.
