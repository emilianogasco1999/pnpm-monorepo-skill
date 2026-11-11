# Docker Multi-Stage Builds en Monorepos con pnpm Workspaces

Esta referencia documenta el estándar de empaquetado y construcción de imágenes Docker multi-stage para aplicaciones dentro de un monorepo gestionado por **pnpm workspaces**.

---

## 1. El Desafío del Monorepo en Docker

En un monorepo pnpm, las aplicaciones (`apps/api`, `apps/web`) dependen de paquetes internos compartidos (`packages/shared`, `packages/config`) mediante el protocolo `workspace:*`.

### Problemas comunes a evitar:
1. **Copiar todo el monorepo en una sola capa (`COPY . .`)**: Invalida el cache de Docker ante cualquier cambio irrelevante (un cambio en frontend recompila el backend innecesariamente).
2. **Contexto de build incorrecto**: Si el `context` de Docker se define dentro de `apps/api/`, Docker no puede acceder a `packages/shared` ni a `pnpm-lock.yaml` en la raíz.
3. **Imágenes infladas con dependencias de desarrollo (`devDependencies`)**: En producción, no deben incluirse TypeScript, linters ni herramientas de compilación.

### Regla Canónica de Contexto:
> **El contexto de construcción (`context`) de cualquier Dockerfile del monorepo DEBE ser siempre la raíz del repositorio (`context: ../..` en compose).**

---

## 2. Dockerfile Canónico para Backend: `apps/api/Dockerfile`

Aplica Clean Architecture en Node.js/Express, compila TypeScript, genera el cliente de Prisma y expone un runtime mínimo bajo un usuario sin privilegios.

```dockerfile
# ==========================================
# STAGE 1: Base con PNPM y Node.js Alpine
# ==========================================
FROM node:20-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable && corepack prepare pnpm@latest --activate
WORKDIR /app

# ==========================================
# STAGE 2: Dependencias y Compilación
# ==========================================
FROM base AS builder
RUN apk add --no-cache libc6-compat openssl python3 make g++

# 1. Copiar manifiestos y estructura del workspace
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY packages/ packages/
COPY apps/api/ apps/api/

# 2. Instalar dependencias con bloqueo determinista
RUN pnpm install --frozen-lockfile

# 3. Generar cliente del ORM (Prisma)
RUN pnpm --filter "@<scope>/api" db:generate

# 4. Compilar en orden: primero shared (SSOT con tsup), luego API
RUN pnpm --filter "@<scope>/shared" build
RUN pnpm --filter "@<scope>/api" build

# ==========================================
# STAGE 3: Runtime Mínimo de Producción
# ==========================================
FROM node:20-alpine AS runner
WORKDIR /app

RUN apk add --no-cache libc6-compat openssl curl

ENV NODE_ENV=production
ENV PORT_API=4000

# Usuario de sistema no root para máxima seguridad
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodejs

# Copiar únicamente los paquetes compilados y dependencias necesarias
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/packages ./packages
COPY --from=builder /app/apps/api/node_modules ./apps/api/node_modules
COPY --from=builder /app/apps/api/dist ./apps/api/dist
COPY --from=builder /app/apps/api/prisma ./apps/api/prisma
COPY --from=builder /app/apps/api/certs ./apps/api/certs
COPY --from=builder /app/apps/api/package.json ./apps/api/package.json
COPY --from=builder /app/package.json ./package.json

WORKDIR /app/apps/api

RUN chown -R nodejs:nodejs /app
USER nodejs

EXPOSE 4000

HEALTHCHECK --interval=20s --timeout=5s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:4000/api/health || exit 1

CMD ["node", "dist/index.js"]
```

---

## 3. Dockerfile Canónico para Frontend: `apps/web/Dockerfile`

Utiliza la funcionalidad **Standalone Output** de Next.js (`output: 'standalone'` en `next.config.ts`), que empaqueta automáticamente el servidor Node.js y rastrea solo las dependencias exactas que la web utiliza, reduciendo la imagen a menos de 150 MB.

```dockerfile
# ==========================================
# STAGE 1: Base con PNPM
# ==========================================
FROM node:20-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable && corepack prepare pnpm@latest --activate
WORKDIR /app

# ==========================================
# STAGE 2: Build de Next.js
# ==========================================
FROM base AS builder
RUN apk add --no-cache libc6-compat

# Manifiestos y código fuente necesarios
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY packages/ packages/
COPY apps/web/ apps/web/

RUN pnpm install --frozen-lockfile

# Compilar paquetes compartidos con tsup antes del build de Next.js
RUN pnpm --filter "@<scope>/shared" build
RUN pnpm --filter "@<scope>/web" build

# ==========================================
# STAGE 3: Runtime Standalone
# ==========================================
FROM node:20-alpine AS runner
WORKDIR /app

RUN apk add --no-cache curl

ENV NODE_ENV=production
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

# Usuario no privilegiado
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# En modo standalone de monorepo, Next.js genera la estructura completa en .next/standalone
COPY --from=builder /app/apps/web/.next/standalone ./
COPY --from=builder /app/apps/web/.next/static ./apps/web/.next/static

RUN chown -R nextjs:nodejs /app
USER nextjs

EXPOSE 3000

HEALTHCHECK --interval=20s --timeout=5s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3000/ || exit 1

CMD ["node", "apps/web/server.js"]
```

### Configuración requerida en `apps/web/next.config.ts`:
```typescript
import type { NextConfig } from 'next';
import path from 'path';

const nextConfig: NextConfig = {
  output: 'standalone',
  // Garantiza que Next.js transpile el paquete compartido del workspace
  transpilePackages: ['@<scope>/shared'],
  // outputFileTracingRoot es vital para que Next.js trace correctamente las dependencias en monorepo
  outputFileTracingRoot: path.join(__dirname, '../../'),
  images: {
    remotePatterns: [
      { protocol: 'http', hostname: 'localhost' },
      { protocol: 'https', hostname: '**' },
    ],
  },
};

export default nextConfig;
```

---

## 4. Archivo Canónico `.dockerignore`

Evita enviar gigabytes de archivos temporales al demonio de Docker durante el `COPY`:

```gitignore
node_modules
.pnpm-store
.next
dist
build
*.tsbuildinfo
.git
.vscode
.idea
*.log
.docker/data/
postgres_data/
backups/
.docker/backup/rclone.conf
solicitud.csr
privada.key
certificado.csr
```

---

## 5. Mejores Prácticas y Checklist de Seguridad

1. **Usuario No-Root (`USER nodejs`)**: Nunca correr contenedores Node en producción bajo `root`.
2. **Healthchecks Activos**: Cada servicio declara un endpoint de chequeo (`/api/health` o `/`) con `curl -f` para permitir orquestación confiable (`condition: service_healthy`).
3. **Aislamiento de Certificados**: Los certificados digitales sensibles (ej. AFIP/ARCA, SSL) no deben hornearse dentro de la imagen; se montan como volúmenes de solo lectura (`:ro`) en el docker-compose.
