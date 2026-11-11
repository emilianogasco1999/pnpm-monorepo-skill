# Fullstack Monorepo Architecture Skill (pnpm Workspaces)

Kit de arquitectura empresarial, directivas operativas y scaffolding automatizado para monorepos estructurados con **pnpm workspaces**, **Clean Architecture** en backend (**Express / Prisma**) y **Next.js 16** (App Router) en frontend, conectados mediante contratos Zod compartidos (**SSOT**).

---

## 🚀 Cómo Instalar esta Skill en un Nuevo Proyecto

Tenés dos métodos para incorporar esta skill en cualquier proyecto nuevo:

### Método 1: Como Submódulo de Git (Recomendado)
Permite mantener la skill sincronizada con este repositorio en GitHub para recibir actualizaciones en el futuro sin romper el repositorio padre.

1. Creá y posicionate en la carpeta de tu nuevo proyecto:
   ```bash
   mkdir mi-nuevo-proyecto
   cd mi-nuevo-proyecto
   git init
   ```

2. Vinculá la skill como un submódulo de Git en `.agents/skills/pnpm-monorepo`:
   ```bash
   git submodule add https://github.com/emilianogasco1999/pnpm-monorepo-skill.git .agents/skills/pnpm-monorepo
   ```

3. Guardá el registro en tu proyecto:
   ```bash
   git add .
   git commit -m "chore: add pnpm-monorepo skill as submodule"
   ```

#### 🔄 Cómo actualizar la skill en el futuro:
Cuando hagas mejoras en este repositorio de GitHub y quieras traerlas a tu proyecto:
```bash
git submodule update --remote .agents/skills/pnpm-monorepo
git add .agents/skills/pnpm-monorepo
git commit -m "chore: update pnpm-monorepo skill to latest"
```

---

### Método 2: Clonación y Desvinculación Directa (Copia Estática)
Si preferís que la skill quede 100% copiada dentro de tu proyecto como código propio sin depender de Git externo:

```bash
# 1. Crear carpeta de destino
mkdir -p .agents/skills

# 2. Clonar el repositorio
git clone https://github.com/emilianogasco1999/pnpm-monorepo-skill.git .agents/skills/pnpm-monorepo

# 3. Eliminar la carpeta .git interna para que no interfiera con tu repositorio
# En Windows (PowerShell):
Remove-Item -Recurse -Force .agents/skills/pnpm-monorepo/.git

# En Linux / Mac:
rm -rf .agents/skills/pnpm-monorepo/.git

# 4. Commitear los archivos en tu nuevo proyecto
git add .
git commit -m "chore: add pnpm-monorepo skill"
```

---

## 🛠️ Cómo Inicializar el Monorepo con tu Agente de IA

Una vez instalada la skill, abrí tu asistente de IA (**Antigravity**, **Cursor**, **Claude Code**, etc.) y consultá la guía paso a paso en [INIT-GUIDE.md](INIT-GUIDE.md).

Allí encontrarás el prompt maestro para que el agente construya la infraestructura raíz, configure Docker, instale Prisma, compile `packages/shared` con `tsup`, levante los endpoints de autenticación y genere la suite de pruebas automáticas con **Vitest**.

