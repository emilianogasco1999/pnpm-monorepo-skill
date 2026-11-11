# Orquestación de Scripts y Ciclo de Vida del Workspace

Esta referencia técnica describe cómo orquestar la ejecución, compilación, análisis estático y tareas de base de datos en monorepos gestionados con **pnpm**.

---

## 1. Principio de Ejecución Filtrada y Recursiva

`pnpm` proporciona dos modificadores fundamentales para interactuar con los paquetes del monorepo:

1. **`--filter "<paquete>"`**: Aísla la ejecución a una aplicación o paquete específico sin salir del directorio raíz.
2. **`-r` (`--recursive`)**: Ejecuta un comando en todos los paquetes del workspace que tengan definido ese script en su `package.json`.

---

## 2. Mapa de Scripts Canónicos en la Raíz

El `package.json` raíz actúa como el panel de control operativo:

| Script | Comando Subyacente | Propósito |
| :--- | :--- | :--- |
| `dev` | `docker compose -f .docker/dev/docker-compose.dev.yml up --build` | Levanta el entorno completo contenerizado de desarrollo. |
| `dev:local` | `pnpm --filter "./apps/*" --parallel dev` | Inicia simultáneamente frontend y backend en el host local. |
| `dev:api` | `pnpm --filter "@<scope>/api" dev` | Inicia únicamente el servidor backend (`tsx watch`). |
| `dev:web` | `pnpm --filter "@<scope>/web" dev` | Inicia únicamente el servidor Next.js del frontend (`next dev`). |
| `dev:db` | `docker compose -f .docker/dev/docker-compose.dev.yml up -d` | Levanta PostgreSQL y MinIO en segundo plano sin correr apps en Docker. |
| `dev:backup` | `docker compose -f .docker/dev/docker-compose.dev.yml run --rm backup` | Ejecuta un respaldo manual bajo demanda. |
| `dev:down` | `docker compose -f .docker/dev/docker-compose.dev.yml down` | Detiene los contenedores de desarrollo preservando volúmenes. |
| `prod:up` | `docker compose --env-file .env -f .docker/prod/docker-compose.prod.yml up -d --build` | Despliega y construye el stack completo de producción. |
| `prod:down` | `docker compose --env-file .env -f .docker/prod/docker-compose.prod.yml down` | Detiene los servicios de producción. |
| `build` | `pnpm -r build` | Compila todo el monorepo en el orden correcto de dependencias. |
| `lint` | `pnpm -r lint` | Chequea tipos y reglas estáticas en todos los paquetes (`tsc --noEmit`). |
| `test` | `vitest run` | Ejecuta la suite completa de pruebas unitarias e integración en el workspace. |
| `test:unit` | `vitest run --exclude="**/*.integration.spec.ts" --exclude="**/*.api.spec.ts"` | Ejecuta únicamente pruebas unitarias puras (casos de uso y dominio). |
| `test:integration` | `vitest run --include="**/*.integration.spec.ts" --include="**/*.api.spec.ts"` | Ejecuta pruebas contra base de datos y endpoints con Supertest. |
| `test:e2e` | `pnpm --filter "@<scope>/web" exec playwright test` | Ejecuta las pruebas End-to-End de navegador con Playwright. |

---

## 3. Topología de Compilación en Cascada (Build Order)

Cuando se ejecuta `pnpm -r build`:
1. `pnpm` analiza el árbol de dependencias de `package.json`.
2. Primero compila los paquetes base (`packages/shared` y `packages/config`).
3. Una vez listos los artefactos de `packages/`, compila las aplicaciones (`apps/api` y `apps/web`).
4. **Regla de oro**: Las aplicaciones nunca deben compilarse antes de que sus dependencias compartidas locales hayan generado sus tipos/distribución.
5. **Pipeline de compilación en `apps/api`**:
   - Si se utilizan alias de rutas (`@core/*`, `@features/*`), el script `build` ejecuta `tsc && tsc-alias` para transformar los alias a rutas relativas válidas en Node ESM.
   - Si Prisma genera su cliente en una ruta personalizada dentro de `src/` (ej: `src/generated/prisma`), se añade un paso de copia (`node scripts/copy-prisma.js`) para trasladar los binarios/runtimes generados hacia `dist/generated/prisma`.

---

## 4. Tareas de Base de Datos y Generación de Clientes

En arquitecturas donde el backend utiliza un ORM (como Prisma):
- **Generación de Cliente Tipado**:
  ```bash
  pnpm --filter "@<scope>/api" db:generate
  ```
  Debe ejecutarse tras cualquier cambio en el esquema para que los tipos de TypeScript coincidan con la base de datos.
- **Sincronización con la Base de Datos**:
  ```bash
  pnpm --filter "@<scope>/api" db:push
  ```
  Aplica los cambios directamente a la base de datos de desarrollo.
- **Ejecución de Seeds y Scripts Administrativos**:
  ```bash
  pnpm --filter "@<scope>/api" db:seed
  # o ejecutando un script puntual:
  pnpm --filter "@<scope>/api" exec tsx prisma/<script>.ts
  ```
