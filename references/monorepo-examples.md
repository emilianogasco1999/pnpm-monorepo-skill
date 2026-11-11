# Ejemplos Canónicos del Monorepo Raíz y Paquetes Compartidos

Este archivo provee los ejemplos de código exactos para los archivos de configuración raíz y los paquetes compartidos (`packages/shared` y `packages/config`).

---

## 1. Definición del Workspace (`pnpm-workspace.yaml`)

```yaml
packages:
  - 'apps/*'
  - 'packages/*'

# En pnpm v10+, las dependencias con scripts postinstall/binarios nativos
# requieren aprobación explícita para compilarse durante pnpm install:
allowBuilds:
  '@prisma/client': true
  '@prisma/engines': true
  bcrypt: true
  esbuild: true
  prisma: true
  sharp: true
  unrs-resolver: true
  '@scarf/scarf': false

strictDepBuilds: false
```

---

## 2. Orquestador Raíz (`package.json`)

```json
{
  "name": "monorepo-root",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "docker compose -f .docker/dev/docker-compose.dev.yml up --build",
    "dev:db": "docker compose -f .docker/dev/docker-compose.dev.yml up -d",
    "dev:api": "pnpm --filter \"@<scope>/api\" dev",
    "dev:web": "pnpm --filter \"@<scope>/web\" dev",
    "dev:local": "pnpm --filter \"./apps/*\" --parallel dev",
    "dev:backup": "docker compose -f .docker/dev/docker-compose.dev.yml run --rm backup",
    "dev:down": "docker compose -f .docker/dev/docker-compose.dev.yml down",
    "prod:up": "docker compose --env-file .env -f .docker/prod/docker-compose.prod.yml up -d --build",
    "prod:down": "docker compose --env-file .env -f .docker/prod/docker-compose.prod.yml down",
    "build": "pnpm -r build",
    "lint": "pnpm -r lint"
  },
  "devDependencies": {
    "typescript": "^5.4.0"
  },
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=9.0.0"
  }
}
```

---

## 3. Docker Compose Raíz (`docker-compose.yml`) y `.env.example`

Puede configurarse mediante `include` apuntando a `.docker/dev/docker-compose.dev.yml`:

```yaml
# Opción A: Delegación modular a .docker/dev/
include:
  - path: .docker/dev/docker-compose.dev.yml
```

### Plantilla Canónica de `.env.example` en la Raíz:
```dotenv
# ==========================================
# CONFIGURACIÓN GENERAL DEL SISTEMA
# ==========================================
NODE_ENV=development
PORT_API=4000
PORT_WEB=3000
# URL interna utilizada por Next.js para proxy de /api (SSR y rewrites hacia Express)
# En dev local: http://127.0.0.1:4000 | En Docker prod: http://api:4000
INTERNAL_API_URL=http://127.0.0.1:4000
COMPOSE_FILE=.docker/dev/docker-compose.dev.yml

# ==========================================
# BASE DE DATOS (POSTGRESQL)
# ==========================================
POSTGRES_USER=app_user
POSTGRES_PASSWORD=app_password_secret
POSTGRES_DB=app_db
POSTGRES_PORT=5432

# Cadena de conexión para la aplicación
DATABASE_URL=postgresql://app_user:app_password_secret@localhost:5432/app_db?schema=public

# ==========================================
# PGADMIN (ADMINISTRADOR DE BASE DE DATOS)
# ==========================================
PGADMIN_EMAIL=admin@dominio.com
PGADMIN_PASSWORD=admin123
PGADMIN_PORT=5050

# ==========================================
# SEGURIDAD Y TOKEN
# ==========================================
JWT_SECRET=super_secret_jwt_key_2026
JWT_REFRESH_SECRET=super_secret_jwt_refresh_key_2026
ACCESS_TOKEN_EXPIRES=15m
REFRESH_TOKEN_EXPIRES=7d
CORS_ORIGIN=http://localhost:3000

# ==========================================
# ALMACENAMIENTO DE ARCHIVOS (MINIO / S3)
# ==========================================
MINIO_ROOT_USER=app_minio_admin
MINIO_ROOT_PASSWORD=app_minio_secret
MINIO_PORT=9000
MINIO_CONSOLE_PORT=9001
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=app_minio_admin
S3_SECRET_KEY=app_minio_secret
S3_BUCKET=app-archivos
S3_REGION=us-east-1
```

---

## 4. Manifiesto y Empaquetado de Contratos (`packages/shared/`)

### A. Manifiesto Canónico (`packages/shared/package.json`)
Declara el mapa `exports` para que Node (Docker) y TypeScript (IDE) resuelvan sin fricción:

```json
{
  "name": "@<scope>/shared",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "default": "./dist/index.js"
    }
  },
  "files": [
    "dist"
  ],
  "scripts": {
    "build": "tsup",
    "dev": "tsup --watch",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@<scope>/config": "workspace:*",
    "rimraf": "^5.0.7",
    "tsup": "^8.0.2",
    "typescript": "^5.4.5"
  }
}
```

### B. Configuración de `tsup` (`packages/shared/tsup.config.ts`)

```typescript
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm", "cjs"],
  dts: true,
  clean: true,
  sourcemap: true,
  splitting: false,
  treeshake: true,
  minify: process.env.NODE_ENV === "production",
});
```

---

## 5. Contratos Compartidos: Esquemas Zod (`packages/shared/src/schemas/user.schema.ts`)

> [!NOTE]
> **Compatibilidad Zod v3 vs Zod v4**:
> - En **Zod v4** (`^4.x`), los validadores de formato son funciones de primer nivel: `z.email({ message: '...' })`, `z.uuid({ message: '...' })`.
> - En **Zod v3** (`^3.x`), se encadenan sobre `z.string()`: `z.string().email('...')`, `z.string().uuid('...')`.
> El patrón arquitectónico (SSOT + `z.infer`) es idéntico en ambas versiones.

```typescript
import { z } from 'zod';

export const USER_ROLES = ['ADMIN', 'OPERADOR', 'CONTADOR', 'GERENTE'] as const;
export const userRoleSchema = z.enum(USER_ROLES);
export type UserRole = z.infer<typeof userRoleSchema>;

export const createUserSchema = z.object({
  // En Zod v4: z.email('Formato de email inválido')
  // En Zod v3: z.string().email('Formato de email inválido')
  email: z.email({ message: 'Formato de email inválido' }),
  password: z
    .string({ message: 'La contraseña es requerida' })
    .min(6, 'La contraseña debe tener al menos 6 caracteres'),
  rol: userRoleSchema,
  personId: z.uuid({ message: 'ID de persona inválido' }).optional().nullable(),
});

export type CreateUserInput = z.infer<typeof createUserSchema>;

export const updateUserSchema = createUserSchema.partial().extend({
  activo: z.boolean().optional(),
});

export type UpdateUserInput = z.infer<typeof updateUserSchema>;
```

---

## 5. Contratos Compartidos: Constantes (`packages/shared/src/constants/roles.ts`)

```typescript
export const ROLES = {
  ADMIN: 'ADMIN',
  OPERADOR: 'OPERADOR',
  CONTADOR: 'CONTADOR',
  GERENTE: 'GERENTE',
} as const;

export type Rol = (typeof ROLES)[keyof typeof ROLES];
```

---

## 6. Exportación de Contratos (`packages/shared/src/index.ts`)

```typescript
export * from './constants/index.js';
export * from './types/index.js';
export * from './schemas/index.js';
```

---

## 7. Configuración Compartida de TypeScript (`packages/config/`)

### A. Base Estricta (`packages/config/tsconfig.base.json`)
Contiene únicamente flags de rigor de tipos e interoperabilidad comunes a cualquier entorno:
```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "sourceMap": true
  }
}
```

### B. Perfil Backend / Node (`packages/config/tsconfig.node.json`)
```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "declaration": true,
    "declarationMap": true
  }
}
```

### C. Perfil Frontend / React (`packages/config/tsconfig.react.json`)
```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "noEmit": true,
    "incremental": true,
    "plugins": [{ "name": "next" }]
  }
}
```

### D. Herencia y Path Aliases en `apps/api/tsconfig.json`
Para utilizar alias limpios (`@core/*`, `@features/*`, `@config/*`) sin romper la salida ESM en `dist/`, se combina `paths` con `tsc-alias` en el script `build` (`"build": "tsc && tsc-alias"`):
```json
{
  "extends": "@<scope>/config/tsconfig.node.json",
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "rootDir": "./src",
    "outDir": "./dist",
    "baseUrl": "./",
    "allowSyntheticDefaultImports": true,
    "paths": {
      "@core/*": ["src/core/*"],
      "@features/*": ["src/features/*"],
      "@domain/*": ["src/core/domain/*"],
      "@application/*": ["src/core/application/*"],
      "@infrastructure/*": ["src/core/infrastructure/*"],
      "@presentation/*": ["src/core/presentation/*"],
      "@config/*": ["src/config/*"]
    }
  },
  "include": ["src/**/*"]
}
```

---

## 7. Plantilla Canónica de Reglas de Agente (`AGENTS.md`)

Todo monorepo creado con este estándar debe incluir en su raíz un archivo `AGENTS.md` para garantizar la gobernanza del código y el formato de entrega a QA/Linear:

```markdown
# Code Review Rules - Monorepo

## Clean Architecture & Monorepo
- Apps: `apps/api` (Express backend) y `apps/web` (Next.js frontend).
- Shared contracts: all DTOs and Zod schemas must live in `packages/shared`.
- In `apps/api`, maintain layer separation: domain -> application -> infrastructure -> presentation.
- Response messages and HTTP status codes in `apps/api` must be centralized in `domain/constants/{feature}.messages.ts`.
- Never use direct database models in public responses; always map to DTOs and omit sensitive fields (e.g. `passwordHash`).

## TypeScript & Code Quality
- Strict TypeScript: no `any` types unless strictly necessary for dynamic database drivers.
- Use `const` and `let`, never `var`.
- Prefer named exports.

## Git & Commits
- Follow Conventional Commits: `feat`, `fix`, `refactor`, `docs`, `chore`, `test`, `build`.
- No direct commits to `main`, `qa`, or `dev`.
- Branches must follow `feature/*`, `bugfix/*`, or `hotfix/*`.

## Cierre de Tareas y Comentarios de Linear (Definition of Done)
Siempre que se dé por finalizada una tarea, feature o bugfix, es **obligatorio** generar el reporte para el ticket de Linear con la siguiente estructura completa:

\`\`\`markdown
### 📌 Resumen de la Tarea
- **Descripción**: [Qué se implementó o corrigió de forma concisa]
- **Rama / PR**: `feature/...` o `bugfix/...` | [PR #XX si fue abierto]
- **Requiere Migración / Env**: [No | Sí: detallar migraciones de Prisma o nuevas variables requeridas en .env]

---

### 📂 Archivos Modificados
- `apps/api/src/...`
- `apps/web/src/...`
- `packages/shared/src/...`

---

### 🔌 Endpoints & Contratos (si aplica al backend/API)
> [!IMPORTANT]
> - Validar las rutas contra el archivo de montaje real en Express (\`src/index.ts\`) para garantizar que el prefijo base sea el exacto implementado en el backend (ej. \`/api/...\` sin asumir prefijos como \`/api/v1\`).
> - Si la tarea involucra **múltiples endpoints**, es obligatorio desglosar **CADA UNO de ellos en su propia subsección** con su método, ruta, payload, response y errores específicos.

#### 1. \`[VERBO] /api/...\`
- **Descripción**: [Propósito del endpoint]
- **Payload (Request)**:
  \`\`\`json
  {}
  \`\`\`
- **Response Exitoso (HTTP 200/201)**:
  \`\`\`json
  {}
  \`\`\`
- **Casos de Error Mapeados**:
  - \`400 Bad Request\`: [Descripción]
  - \`401 Unauthorized / 403 Forbidden\`: [Descripción]
  - \`404 Not Found\`: [Descripción]

---

### 🧪 Guía de Pruebas (QA / Review)

#### 1. Happy Path (Verificación de funcionamiento exitoso)
1. [Paso 1: Levantar servicios / estado inicial]
2. [Paso 2: Ejecución de la acción / petición curl / interacción UI]
3. [Paso 3: Resultado esperado y validación de persistencia/respuesta]

#### 2. Unhappy Paths (Verificación de errores esperados)
- **Caso A ([Ej. Contraseña inválida / Registro duplicado])**: [Acción disparadora] -> Debe responder [Status HTTP] con mensaje de error específico.
- **Caso B ([Ej. Faltan campos requeridos en Zod])**: [Acción disparadora] -> Debe responder 400 Bad Request con detalle de validación.
\`\`\`
```

