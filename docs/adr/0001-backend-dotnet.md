# ADR-0001: Backend en .NET con ASP.NET Core

- **Estado:** Aceptado. Los nombres de proyectos, carpetas y esquemas fueron reemplazados por [ADR-0004](0004-idioma-del-codigo.md).
- **Fecha:** 2026-11-23
- **Principios relacionados:** II (monolito modular), V (dinero y fechas), VI (registros inmutables), VII (pruebas), X (operación)

## Contexto

La constitución pide un monolito modular con **límites estrictos y verificados automáticamente** entre módulos (Principio II), cálculos de dinero exactos (Principio V) y pruebas de integración con base de datos real (Principio VII). El dueño del proyecto domina TypeScript, Python, C#/.NET y Rust.

## Decisión

El backend se construye en **.NET 10 (LTS)** con **ASP.NET Core**, como una sola solución con un proyecto por módulo.

### Estructura de la solución

```
src/
├── Host/
│   └── AdminRentas.Api/                  # Punto de entrada: registra módulos, autenticación, endpoints
├── Modulos/
│   ├── Identidad/
│   │   ├── AdminRentas.Identidad/            # Implementación (todo internal)
│   │   └── AdminRentas.Identidad.Contratos/  # API pública y eventos de integración
│   ├── Propiedades/  (misma estructura)
│   ├── Ingesta/
│   ├── Servicios/
│   ├── Ocupacion/
│   └── Notificaciones/
└── Compartido/
    ├── AdminRentas.Nucleo/               # Núcleo compartido: Dinero, Periodo, IDs, Resultado
    └── AdminRentas.Infraestructura/      # Bus de eventos, trabajos programados, auditoría
tests/
├── AdminRentas.Arquitectura.Tests/       # Reglas de dependencia entre módulos
└── Modulos/
    └── <Modulo>.Tests/                   # Unitarias, integración y aceptación por módulo
```

### Reglas

1. **Visibilidad:** todo lo que hay en el proyecto de implementación de un módulo es `internal`. Lo único `public` es su método de registro (`AddServiciosModule()`) y lo que vive en `.Contratos`.
2. **Referencias entre proyectos:**
   - La implementación de un módulo puede referenciar los proyectos `.Contratos` de otros módulos, `Nucleo` e `Infraestructura`. **Nunca** la implementación de otro módulo.
   - Los proyectos `.Contratos` solo referencian `Nucleo`.
   - `Host` referencia las implementaciones únicamente para registrarlas.
3. **Datos:** un `DbContext` de EF Core por módulo con `HasDefaultSchema("<modulo>")` y su propia tabla de historial de migraciones dentro de su esquema.
4. **Configuración común:** `Directory.Build.props` con `Nullable` activado y advertencias tratadas como errores, y `Directory.Packages.props` para administrar las versiones de paquetes en un solo lugar.
5. **Pruebas:** xUnit, Testcontainers para PostgreSQL (Principio VII.2) y NetArchTest o ArchUnitNET para las pruebas de arquitectura (se elige al armar el esqueleto).
6. **Dinero:** `decimal` en el dominio y `numeric(12,2)` en PostgreSQL. Nunca `double` ni `float`.
7. **Fechas:** `DateOnly` para fechas sin hora (vencimientos, inicio de estancia) y `DateTimeOffset` en UTC para instantes.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **TypeScript con NestJS** | Los módulos de Nest organizan la inyección de dependencias, pero no impiden que un módulo importe el interior de otro. Habría que depender de `dependency-cruiser` en CI. Tampoco hay decimal nativo. Su ventaja, compartir lenguaje con el frontend, se compensa generando el cliente desde OpenAPI. |
| **Python con FastAPI** | No hay forma de hacer privado un módulo; todo depende de `import-linter`. Es la mejor opción para IA, pero la extracción es un adaptador pequeño detrás de un puerto (Principio IX) y se puede hacer igual de bien con llamadas HTTP. |
| **Rust con Axum** | Más tiempo de desarrollo para una aplicación de CRUD, eventos y trabajos programados, sin un beneficio que el proyecto necesite. |

## Consecuencias

**Positivas**

- Los límites entre módulos se aplican **al compilar**, no solo en CI.
- Tipos exactos para dinero y fechas, sin librerías adicionales.
- Hay mucho material de referencia sobre monolitos modulares en .NET.
- Cada módulo se podría extraer como servicio llevándose su proyecto, sus contratos y su esquema.

**Negativas o riesgos**

- El frontend estará en otro lenguaje. **Mitigación:** la API expone OpenAPI y el cliente TypeScript se genera automáticamente.
- MediatR, la opción más conocida para mensajes en proceso, ahora tiene licencia comercial (con edición gratuita para uso pequeño). Se decide la alternativa en ADR-0005.
- Más proyectos en la solución (dos por módulo). Es el costo de tener límites que verifica el compilador.

## Decisiones derivadas

- **ADR-0002:** framework del frontend (PWA) y generación del cliente desde OpenAPI.
- **ADR-0005:** bus de eventos en proceso (Wolverine, despachador propio u otro).
- **ADR-0007:** hosting y entorno local. .NET Aspire es candidato para levantar el entorno local con un solo comando (Principio X.2).
