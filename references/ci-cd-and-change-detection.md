# CI/CD y Detección Inteligente de Cambios en pnpm Monorepo

Esta referencia técnica define el pipeline de **Integración Continua (CI)** con **GitHub Actions**, el cálculo de impacto de cambios con filtros de `pnpm` y los gates de calidad obligatorios para Pull Requests en el monorepo.

---

## 1. El Problema de CI en Monorepos y la Solución con Grafo de Impacto

En un monorepo que alberga múltiples aplicaciones y paquetes compartidos, ejecutar la suite completa de tests, análisis estático y compilación ante cualquier cambio menor es ineficiente y costoso en minutos de CI.

### El Grafo de Impacto de pnpm (`...[origin/main]`):
El modificador de filtro `...[<rama>]` analiza el historial de Git y el árbol de dependencias internas para seleccionar **únicamente los paquetes modificados y sus dependientes directos o indirectos**:

```bash
# Ejecutar tests únicamente en los paquetes que cambiaron respecto a main
pnpm --filter "...[origin/main]" test:unit

# Compilar únicamente lo afectado por el Pull Request
pnpm --filter "...[origin/main]" build
```

### Tabla de Comportamiento:
| Archivo Modificado en el PR | Paquetes Seleccionados por pnpm | Qué se Ejecuta en CI |
| :--- | :--- | :--- |
| `apps/web/src/components/Button.tsx` | `@<scope>/web` | Lint, test y build **solo de la Web** (~35s). |
| `apps/api/src/features/auth/use-case.ts` | `@<scope>/api` | Lint, test y build **solo de la API** (~40s). |
| `packages/shared/src/schemas/user.ts` | `@<scope>/shared`, `@<scope>/api`, `@<scope>/web` | Como ambos dependen de `shared`, se valida el monorepo completo. |
| `README.md` o documentación | Ninguno | CI finaliza con éxito en segundos sin correr builds. |

---

## 2. Pipeline Canónico de GitHub Actions: `.github/workflows/ci.yml`

Este workflow valida automáticamente cada Pull Request hacia `dev` o `main`, y cada push a las ramas principales.

```yaml
name: CI Monorepo Quality Gate

on:
  pull_request:
    branches: [main, dev]
  push:
    branches: [main, dev]

# Cancela ejecuciones en curso si se sube un nuevo commit al mismo PR
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  validate:
    name: Lint, Test & Build
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      # 1. Checkout del código con historial completo para que pnpm calcule el diff
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # 2. Habilitar Corepack e instalar PNPM
      - name: Install pnpm
        uses: pnpm/action-setup@v4
        with:
          run_install: false

      # 3. Configurar Node.js con Cache nativo de pnpm
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"

      # 4. Instalación determinista de dependencias
      - name: Install Dependencies
        run: pnpm install --frozen-lockfile

      # 5. Generar cliente del ORM (Prisma)
      - name: Generate Prisma Client
        run: pnpm --filter "@<scope>/api" db:generate

      # 6. Compilar paquetes compartidos PRIMERO (requisito para types)
      - name: Build Shared Packages
        run: pnpm --filter "@<scope>/shared" build

      # 7. Verificación de Linter y Tipos en paquetes afectados
      - name: Lint & Typecheck (Affected)
        run: |
          if [ "${{ github.event_name }}" = "pull_request" ]; then
            pnpm --filter "...[origin/${{ github.base_ref }}]" lint
          else
            pnpm -r lint
          fi

      # 8. Suite de Pruebas Unitarias e Integración (Afectados)
      - name: Run Tests (Affected)
        run: |
          if [ "${{ github.event_name }}" = "pull_request" ]; then
            pnpm --filter "...[origin/${{ github.base_ref }}]" test:unit
          else
            pnpm -r test:unit
          fi

      # 9. Compilación de Aplicaciones Afectadas
      - name: Build Applications (Affected)
        run: |
          if [ "${{ github.event_name }}" = "pull_request" ]; then
            pnpm --filter "...[origin/${{ github.base_ref }}]" build
          else
            pnpm -r build
          fi

  # Job opcional de verificación de Dockerfiles (dry-run build)
  docker-smoke-test:
    name: Docker Build Smoke Test
    needs: validate
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Test API Docker Build
        uses: docker/build-push-action@v5
        with:
          context: .
          file: apps/api/Dockerfile
          push: false
          tags: test-api:local
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 3. Reglas de Protección de Ramas en GitHub

Para que el pipeline de CI garantice estabilidad real:

1. **Branch Protection Rules (`main` y `dev`)**:
   - Activar **"Require status checks to pass before merging"**.
   - Marcar el check `validate (Lint, Test & Build)` como obligatorio (*Required*).
   - Activar **"Require branches to be up to date before merging"** para asegurar que el PR se valide contra el estado actual de la rama destino.
2. **Prohibición de Commits Directos**:
   - Nadie (incluyendo administradores) debe poder hacer `git push` directo a `main` o `dev`. Todo cambio entra por Pull Request.
   - Todo PR debe respetar la convención de títulos en Conventional Commits (`feat:`, `fix:`, `refactor:`, `chore:`).

