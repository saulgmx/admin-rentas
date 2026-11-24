# ADR-0002: Frontend PWA con React, Vite y TypeScript

- **Estado:** Aceptado
- **Fecha:** 2026-11-23
- **Principios relacionados:** III (simplicidad para el administrador), V (dinero y fechas), VIII (seguridad), X (operación)

## Contexto

El cliente es una PWA *mobile-first* (constitución, restricciones tecnológicas). Su usuario principal es el administrador, con habilidades tecnológicas básicas, que la usará desde el celular y a veces con datos móviles. El backend es .NET (ADR-0001), así que el frontend no comparte lenguaje con él.

## Decisión

El frontend es una **SPA en React con Vite y TypeScript**, instalable como PWA, que consume la API mediante un **cliente generado desde OpenAPI**.

### Detalles

1. **PWA:** `vite-plugin-pwa` para el manifiesto y el *service worker*.
   - En la primera versión solo se guarda en caché la estructura de la app (*app shell*), para que abra rápido. **No hay modo sin conexión** para los datos.
   - No se usan notificaciones push del navegador: los avisos van por Telegram y correo (constitución, Principio IX).
2. **Cliente de la API:**
   - El backend publica su documento OpenAPI con `Microsoft.AspNetCore.OpenApi`.
   - Los tipos y el cliente TypeScript se generan con `openapi-typescript` y `openapi-fetch`.
   - El código generado **no se edita a mano** y se regenera en CI. Si el contrato cambia y el frontend no compila, el build falla.
3. **Datos del servidor:** TanStack Query para caché, reintentos y estados de carga.
4. **Captura de fotos:** `<input type="file" accept="image/*" capture="environment">`, que abre directamente la cámara trasera del celular.
   - Las fotos se **reducen y comprimen en el navegador** antes de subirlas, para cuidar los datos móviles del administrador y respetar el límite de 10 MB (Principio IV.3).
5. **Formato regional:** español de México. Dinero con `Intl.NumberFormat('es-MX', { style: 'currency', currency: 'MXN' })` y fechas en la zona horaria de la propiedad , que es un dato de cada propiedad. El frontend **no hace cálculos de dinero**: solo muestra los montos que calcula el backend.
6. **Despliegue:** el build estático lo sirve el mismo `AdminRentas.Api` desde `wwwroot`. Así se mantiene **un solo desplegable** (Principio X.1) y no hay problemas de CORS.
7. **Ubicación en el repositorio:** `src/Web/`.

### Se decide al armar el esqueleto

- Enrutador: React Router o TanStack Router.
- Componentes y estilos: por ejemplo, Tailwind con shadcn/ui.
- Formularios y validación: por ejemplo, React Hook Form con Zod.

## Alternativas consideradas

| Alternativa | Por qué no se eligió |
|---|---|
| **Blazor WebAssembly** | Compartiría los proyectos `.Contratos` en C#, pero la primera carga descarga el runtime de .NET (varios MB). En un celular modesto con datos móviles, la app tardaría en abrir y el administrador volvería a WhatsApp. Generar el cliente desde OpenAPI da casi el mismo beneficio de tipos. |
| **Angular** | Buen soporte PWA, pero es más pesado y con más ceremonia de la que necesita una app de este tamaño. |
| **Next.js** | Su fuerte es el renderizado en servidor y el SEO. Esta app es privada y no los necesita, y su soporte PWA no es de primera clase. Además, agregaría un segundo proceso de servidor al despliegue. |

## Consecuencias

**Positivas**

- Carga inicial ligera y rápida, que es lo que más importa para el administrador.
- Los tipos del frontend siempre coinciden con la API, porque se generan de ella.
- Un solo desplegable: el backend sirve también el frontend.

**Negativas o riesgos**

- Dos lenguajes en el repositorio (C# y TypeScript) y dos cadenas de herramientas (`dotnet` y `npm`).
- En iOS, una PWA instalada tiene algunas limitaciones respecto a Android. Para esta app no afectan, porque no usa notificaciones push ni modo sin conexión.

## Decisiones derivadas

- **ADR-0007:** hosting. El frontend se sirve desde el contenedor del backend.
- Esqueleto del proyecto: enrutador, componentes y formularios.
