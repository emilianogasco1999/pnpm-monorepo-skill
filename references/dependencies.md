# Política y Gestión de Dependencias en pnpm Monorepo

Esta referencia técnica establece los estándares obligatorios para la adición, versionado y consumo de dependencias en proyectos organizados con **pnpm workspaces**.

---

## 1. Niveles de Aislamiento y Reglas por Carpeta

Cada capa del monorepo tiene un propósito estricto. **Queda terminantemente prohibido cruzar dependencias entre capas incompatibles.**

| Ubicación | Tipo de Dependencias Permitidas | Ejemplos Permitidos | Prohibiciones Estrictas |
| :--- | :--- | :--- | :--- |
| **Raíz (`/`)** | Herramientas globales de desarrollo y orquestación del workspace (`-w`). | `typescript`, `prettier`, `@types/node` | **NUNCA** instalar dependencias de runtime (`express`, `react`, `axios`, etc.). |
| **`packages/shared`** | Librerías isomórficas puras de validación y utilidades agnósticas al entorno. | `zod` | **NUNCA** dependencias de Node (`fs`, `crypto`, `pg`) ni dependencias del navegador (`react`, `dom`). |
| **`packages/config`** | Configuraciones y plugins de análisis estático / compilación. | `@types/node`, configs de ESLint/Prettier | Sin dependencias de negocio o runtime. |
| **`apps/api`** | Dependencias exclusivas de backend y persistencia. | `express`, `@prisma/client`, `bcrypt`, `jsonwebtoken`, `@aws-sdk/*` | **NUNCA** dependencias de UI/React. |
| **`apps/web`** | Dependencias exclusivas de frontend y experiencia de usuario. | `next`, `react`, `tailwindcss`, `@tanstack/react-query`, `zustand`, `lucide-react`, `sonner` | **NUNCA** dependencias de backend (`prisma`, `express`, `pg`). |

---

## 2. Consumo Interno: El Protocolo `workspace:*`

Las aplicaciones dentro de `apps/` deben consumir los paquetes locales de `packages/` usando el protocolo dinámico de pnpm:

En el `package.json` de cada app:
```json
{
  "dependencies": {
    "@<scope>/shared": "workspace:*"
  },
  "devDependencies": {
    "@<scope>/config": "workspace:*"
  }
}
```

### ¿Por qué `workspace:*`?
- **Resolución local inmediata**: pnpm crea symlinks automáticos hacia el código fuente de los paquetes locales, sin necesidad de publicarlos en un registro de npm.
- **Detección de cambios instantánea**: Las modificaciones en `packages/shared` o `packages/config` se reflejan en tiempo real en las aplicaciones dependientes.

---

## 3. Prevención de Dependencias Fantasma (*Phantom Dependencies*)

A diferencia de npm y Yarn v1 (que "aplanaban" o hacían *hoisting* de todo en una carpeta `node_modules` gigante), **pnpm utiliza una estructura aislada basada en enlaces duros y enlaces simbólicos (`.pnpm/`)**.

### Implicancia de Arquitectura:
- Si una aplicación necesita usar `zod` directamente en su código, pero no lo tiene declarado en su propio `package.json` (o a través de `@<scope>/shared`), **el import fallará en tiempo de compilación o ejecución**.
- **Regla**: Todo paquete que sea importado explícitamente en el código de una aplicación debe estar declarado en su respectivo `package.json`. No se asume que una dependencia "está instalada porque otra la trajo".

---

## 4. Requisitos Isomórficos para `packages/shared`

`packages/shared` debe compilar tanto para el motor de Node.js (V8 en el servidor) como para el navegador web.

- **No usar APIs de Node**: No importar `node:fs`, `node:path`, `node:crypto`, `node:process` dentro de `packages/shared`.
- **No usar APIs del navegador**: No acceder a `window`, `document`, `localStorage` dentro de `packages/shared`.
- **Zod como contrato universal**: Todo esquema de validación debe ser puro (estructuras, tipos primitivos, validaciones de regex, mensajes de error).
- **Compilación obligatoria con `tsup`**: Para garantizar interoperabilidad entre Node en Docker y Next.js sin errores de extensiones `.ts`, `packages/shared` debe compilar artefactos (`dist/index.js` y `dist/index.d.ts`) usando `tsup` y exponerlos mediante el mapa `exports` de `package.json` (ver [Mecánica de Compilación y Consumo](package-bundling-and-consumption.md)).

---

## 5. Comandos Canónicos de Gestión de Dependencias

Siempre ejecutar los comandos de instalación utilizando el filtro `--filter` para no contaminar la raíz:

### Para una aplicación específica:
```bash
# Agregar dependencia de producción a la API
pnpm --filter "@<scope>/api" add bcrypt

# Agregar dependencia de desarrollo a la Web
pnpm --filter "@<scope>/web" add -D @types/react

# Vincular un paquete compartido local
pnpm --filter "@<scope>/api" add "@<scope>/shared@workspace:*"
```

### Para la raíz del monorepo:
```bash
# Agregar una herramienta de desarrollo global (requiere flag -w)
pnpm add -D -w prettier
```

### Mantenimiento y actualización:
```bash
# Instalar todas las dependencias del monorepo
pnpm install

# Actualizar dependencias interactivamente en todo el workspace
pnpm update -r --interactive
```
