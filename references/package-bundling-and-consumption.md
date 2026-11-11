# Mecánica de Compilación y Consumo de Paquetes (`packages/*`)

Esta referencia técnica define el estándar para la construcción, empaquetado, resolución de tipos y consumo en tiempo de ejecución de paquetes compartidos (`packages/shared`, `packages/config`) dentro de un monorepo gestionado con **pnpm workspaces**.

---

## 1. El Problema Raíz: TypeScript en Desarrollo vs JavaScript en Docker

El error más recurrente en monorepos fullstack ocurre en la frontera entre desarrollo y producción:

- **En desarrollo local**: Entornos como `tsx watch` (backend) y Next.js Webpack/Turbopack (frontend) transpilan TypeScript en memoria en tiempo de ejecución. Si `packages/shared` apunta a sus fuentes `src/index.ts`, todo parece funcionar sin problemas.
- **En producción (Docker)**: Node.js ejecuta JavaScript puro (`node dist/index.js`). Node **no comprende archivos `.ts` de forma nativa**.
  - Si `packages/shared` no generó sus archivos compilados, Node lanza `ERR_UNKNOWN_FILE_EXTENSION` o `MODULE_NOT_FOUND`.
  - Si el código importa con extensiones forzadas (`.js` en código fuente `.ts`), se producen desajustes entre el compilador de TypeScript (`tsc`) y el empaquetador.

> **Regla de Oro**: Ninguna aplicación en producción debe ejecutar código fuente `.ts` de `packages/*`. Todo paquete compartido debe comportarse como una librería profesional con artefactos distribuidos en `dist/` (`.mjs` para ESM y `.d.ts` para tipado).

---

## 2. Comparativa Arquitectónica de Consumo

| Criterio | Enfoque A: Source-Only (`src/index.ts`) | Enfoque B: Compilado con `tsup` (**Estándar Canónico**) |
| :--- | :--- | :--- |
| **Tiempo de Build en Dev** | Inmediato (0 ms, sin paso previo). | Ultrarrápido (~20 ms con esbuild). |
| **Modo Watch en Dev** | Automático vía loaders en memoria. | `tsup --watch` recompila en segundo plano ante cada guardado. |
| **Compatibilidad Node en Prod** | **Frágil**: Requiere loaders en runtime (`tsx`) o compilar paquetes compartidos dentro de la API con `tsc-alias`. | **Robusta**: Node ejecuta JavaScript puro estándar sin dependencias de compilación en producción. |
| **Compatibilidad Next.js** | Requiere `transpilePackages: ['@<scope>/shared']`. | Funciona nativamente con o sin `transpilePackages`. |
| **Multi-Stage Docker** | Requiere copiar carpetas completas de código fuente en la imagen final. | Solo requiere el `dist/` compilado y el `package.json`. |

---

## 3. Configuración Canónica de `packages/shared`

Para lograr compilación ultrarrápida (basada en Go/esbuild) y generación de definiciones de tipos `.d.ts`, estandarizamos con **`tsup`**.

### A. Manifiesto del Paquete: `packages/shared/package.json`

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

### B. Configuración del Empaquetador: `packages/shared/tsup.config.ts`

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

### C. Configuración de TypeScript: `packages/shared/tsconfig.json`

```json
{
  "extends": "@<scope>/config/tsconfig.base.json",
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "composite": true
  },
  "include": ["src/**/*"]
}
```

---

## 4. Consumo en las Aplicaciones

Las aplicaciones consumen `@<scope>/shared` a través de la directiva de workspace de pnpm:

```json
{
  "dependencies": {
    "@<scope>/shared": "workspace:*"
  }
}
```

### En Backend (`apps/api`):
El código fuente en TypeScript importa los contratos como cualquier paquete externo:
```typescript
import { createUserSchema, type CreateUserInput } from "@<scope>/shared";
```
- **En tiempo de desarrollo**: TypeScript en el editor lee los tipos desde `packages/shared/dist/index.d.ts`.
- **En tiempo de compilación (`pnpm build`)**: `tsc` valida los tipos contra `dist/index.d.ts` y genera código JS con imports hacia `@<scope>/shared`.
- **En tiempo de ejecución (`node dist/index.js`)**: Node resuelve el paquete mediante el symlink creado por pnpm en `node_modules/@<scope>/shared` y ejecuta `dist/index.js` (ESM). No hay extensiones `.ts` que provoquen fallos.

### En Frontend (`apps/web`):
Next.js resuelve el paquete a través del campo `exports` de `package.json`.
Opcionalmente, para permitir que Next.js aplique Fast Refresh y tree-shaking profundo durante el desarrollo, se declara en `apps/web/next.config.ts`:

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  transpilePackages: ["@<scope>/shared"],
  output: "standalone",
};

export default nextConfig;
```

---

## 5. Orquestación del Flujo de Desarrollo (Dev Pipeline)

Para que los cambios en esquemas Zod o constantes de `packages/shared` se reflejen en tiempo real en la API y la Web sin necesidad de ejecutar un build manual:

En el `package.json` raíz:
```json
{
  "scripts": {
    "dev:shared": "pnpm --filter \"@<scope>/shared\" dev",
    "dev:local": "pnpm --filter \"./packages/*\" --filter \"./apps/*\" --parallel dev"
  }
}
```
Al ejecutar `pnpm dev:local`, `tsup --watch` se inicia en segundo plano. Cada vez que se modifica un archivo en `packages/shared/src/`, `tsup` regenera `dist/` en ~15 ms, disparando automáticamente el recargado de `apps/api` (`tsx watch`) y `apps/web` (Next.js Fast Refresh).

---

## 6. Construcción Determinista en Docker Multi-Stage

El orden de compilación en Docker es **estrictamente secuencial**. Nunca se debe intentar compilar una aplicación antes de que sus dependencias locales compartidas hayan generado su carpeta `dist/`.

### Extracto Canónico del `apps/api/Dockerfile`:

```dockerfile
# ==========================================
# ETAPA DE CONSTRUCCIÓN (BUILDER)
# ==========================================
FROM base AS builder

# 1. Copiar manifiestos y archivos de configuración
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY packages/ packages/
COPY apps/api/ apps/api/

# 2. Instalar todas las dependencias
RUN pnpm install --frozen-lockfile

# 3. Compilar paquetes compartidos PRIMERO (genera dist/ de shared)
RUN pnpm --filter "@<scope>/shared" build

# 4. Generar cliente del ORM
RUN pnpm --filter "@<scope>/api" db:generate

# 5. Compilar la aplicación backend (tsc + tsc-alias)
RUN pnpm --filter "@<scope>/api" build

# 6. Desplegar dependencias de producción aisladas
RUN pnpm --filter "@<scope>/api" --prod deploy /app/pruned
```

### Por qué este orden previene fallos:
1. `pnpm install` crea los symlinks de `node_modules/@<scope>/shared` apuntando a `packages/shared`.
2. `pnpm --filter "@<scope>/shared" build` crea `packages/shared/dist/index.js` y `index.d.ts`.
3. Cuando `apps/api` ejecuta `tsc`, el compilador encuentra los archivos de declaración `.d.ts` ya existentes.
4. En la imagen de producción final (`runner`), el symlink de pnpm resuelve limpiamente hacia los artefactos JavaScript ya compilados en `packages/shared/dist`.

