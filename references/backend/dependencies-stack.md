# Stack de Dependencias Esenciales del Backend

Esta referencia técnica documenta el catálogo de dependencias que componen el backend (`apps/api`), explicando el propósito arquitectónico de cada librería y las consideraciones de compatibilidad entre versiones.

---

## 1. Servidor, Transporte HTTP y Protección

### `express` (^4.x / ^5.x)
- **Propósito**: Framework de servidor web y enrutamiento HTTP.
- **Uso en el sistema**: Define la aplicación principal, routers por feature y ciclo de middlewares. (En Express 5, las promesas rechazadas en middlewares async se propagan automáticamente, aunque se mantiene `catchedAsync` por consistencia explícita).

### `cors`
- **Propósito**: Gestión de Cross-Origin Resource Sharing.
- **Uso en el sistema**: Permite peticiones seguras exclusivamente desde los orígenes autorizados del frontend (`FRONTEND_URL` / `CORS_ORIGIN`), habilitando `credentials: true` para el intercambio de cookies `httpOnly`.

### `cookie-parser`
- **Propósito**: Procesamiento y parseo de encabezados `Cookie`.
- **Uso en el sistema**: Permite al backend leer cookies seguras `httpOnly` (como el `refreshToken`), protegiendo tokens de sesión contra ataques XSS en el navegador.

### `express-rate-limit`
- **Propósito**: Protección contra ataques de fuerza bruta, abuso de endpoints y denegación de servicio (DoS).
- **Uso en el sistema**: Configurado en `core/presentation/middlewares/rateLimiter.middleware.ts` con políticas diferenciadas (límite general holgado para `/api` y límite estricto con `skipSuccessfulRequests: true` para `/api/auth/login`).

### `pino` + `pino-http` (con `pino-pretty` en desarrollo)
- **Propósito**: Logging estructurado de alto rendimiento en formato JSON, correlación de peticiones con `x-request-id` y prevención de contaminación de consola.
- **Uso en el sistema**: Configurado en `core/presentation/middlewares/httpLogger.middleware.ts`. Registra método, ruta, código de estado, tiempo de respuesta y Request ID, ignorando automáticamente sondas de salud (`/healthz`). En desarrollo formatea con colores vía `pino-pretty`. (Ver [Observabilidad y Health Checks](../observability-and-health-checks.md)).

---

## 2. Configuración y Validación de Entorno

### `dotenv` + Validación (`zod` o `env-var`)
- **Propósito**: Carga y validación fail-fast de variables de entorno al iniciar el proceso.
- **Uso en el sistema**:
  - `dotenv` carga primero el `.env` de la raíz del monorepo (`../../.env`) y luego el `.env` local de `apps/api`.
  - Las variables se validan al arranque con un esquema **Zod** (o `env-var`), abortando inmediatamente con `process.exit(1)` si falta una clave crítica (`DATABASE_URL`, `JWT_SECRET`, `S3_*`).

---

## 3. Seguridad y Criptografía

### `bcrypt`
- **Propósito**: Hashing unidireccional de contraseñas (`bcrypt.hash` y `bcrypt.compare`).

### `jsonwebtoken`
- **Propósito**: Emisión y verificación criptográfica de tokens JWT de corta duración (Access Token) y larga duración (Refresh Token).

---

## 4. Validación de Contratos y Esquemas

### `zod` (^3.x o ^4.x)
- **Propósito**: Validación de esquemas y tipado estático en tiempo de ejecución.
- **Uso en el sistema**: Valida `body`, `query` y `params` en el middleware `validateRequest(schema)` consumiendo los contratos compartidos de `@<scope>/shared`.
- **Nota de versión**: En Zod v4 (`^4.x`) se prefiere el uso de primitivos directos (`z.email()`, `z.uuid()`) y la propiedad de errores es `error.issues`, mientras que en Zod v3 (`^3.x`) se usa `z.string().email()`.

---

## 5. Base de Datos y Persistencia Relacional (Prisma ORM)

### `@prisma/client`, `@prisma/adapter-pg` y `pg`
- **Propósito**: ORM fuertemente tipado y pool de conexiones para PostgreSQL.
- **Adaptabilidad según versión de Prisma**:
  - **Prisma 7+ (`^7.x`)**:
    - Elimina `url = env("DATABASE_URL")` del bloque `datasource` en `schema.prisma`.
    - Requiere un archivo **`apps/api/prisma.config.ts`** usando `defineConfig` de `'prisma/config'` para inyectar la URL de conexión y configurar la ruta de migraciones/seed.
    - Utiliza obligatoriamente Driver Adapters (`@prisma/adapter-pg` con `pg.Pool`) al instanciar `PrismaClient`.
    - Si se define un `output` personalizado en `schema.prisma` (ej. `output = "../src/generated/prisma"`), el script de build debe copiar los artefactos generados a `dist/` (`node scripts/copy-prisma.js`).
  - **Prisma 5 / 6 (`^5.x` / `^6.x`)**:
    - Declara `url = env("DATABASE_URL")` directamente dentro de `prisma/schema.prisma` y genera por defecto en `node_modules/@prisma/client` sin requerir `prisma.config.ts`.

---

## 6. Almacenamiento de Objetos y Archivos

### `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner` y `multer`
- **Propósito**: Recepción de archivos `multipart/form-data` en memoria (`multer.memoryStorage()`) y persistencia desacoplada en buckets compatibles con S3 (MinIO en local, AWS S3 / Cloudflare R2 en producción).

---

## 7. Compilación y Resolución de Alias

### `typescript`, `tsx` y `tsc-alias`
- **`tsx`**: Ejecución directa y modo watch en desarrollo (`tsx watch src/index.ts`).
- **`tsc-alias`**: Reescribe los alias de importación (`@core/*`, `@features/*`, `@config/*`) a rutas relativas con extensión `.js` durante el build de producción (`tsc && tsc-alias`), evitando errores `ERR_MODULE_NOT_FOUND` al ejecutar Node ESM en `dist/`.

---

## 8. Librerías Opcionales de Dominio (Según Requerimientos del Proyecto)

Estas librerías NO son obligatorias para todo monorepo, pero forman parte del stack validado cuando el dominio de negocio lo requiere:
- **`pdf-parse`**: Extracción de texto desde buffers PDF en servicios de infraestructura para autocompletar o validar comprobantes subidos por el usuario.
- **`@arcasdk/core`**: Integración con Web Services de AFIP/ARCA (facturación electrónica `WSFEv1` y consulta de padrón `A5`).
- **`swagger-jsdoc` y `swagger-ui-express`**: Generación de portal OpenAPI interactivo en `/api-docs`.

---

## Resumen del Comando de Instalación Base

```bash
# Dependencias esenciales de producción
pnpm --filter "@<scope>/api" add express cors cookie-parser express-rate-limit morgan dotenv bcrypt jsonwebtoken zod @prisma/client @prisma/adapter-pg pg @aws-sdk/client-s3 @aws-sdk/s3-request-presigner multer swagger-jsdoc swagger-ui-express

# Tipos y herramientas de desarrollo
pnpm --filter "@<scope>/api" add -D @types/express @types/cors @types/cookie-parser @types/morgan @types/bcrypt @types/jsonwebtoken @types/pg @types/multer @types/swagger-jsdoc @types/swagger-ui-express @types/node prisma tsc-alias tsx typescript
```
