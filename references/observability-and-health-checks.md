# Observabilidad, Logging Estructurado y Health Checks

Esta referencia técnica define los estándares de **observabilidad**, **registro estructurado (JSON)**, **trazabilidad de peticiones mediante Request ID** y **sondas de estado (Health Checks)** para aplicaciones Node.js/Express y contenedores Docker en el monorepo.

---

## 1. Logging Estructurado vs Texto Plano

En entornos de producción modernos (contenedores Docker, Grafana Loki, Datadog, AWS CloudWatch), los registros emitidos en texto plano (`console.log`, `morgan`) son difíciles de parsear, filtrar y correlacionar.

### El Estándar Canónico: `pino` + `pino-http`
- **Formato JSON nativo**: Cada log se serializa como un objeto JSON estructurado con timestamp Unix, nivel de severidad (`info`, `warn`, `error`) y metadatos contextuales.
- **Rendimiento ultra alto**: Pino es hasta 5 veces más rápido que Winston o Bunyan gracias a su arquitectura asíncrona de streams.
- **Formato legible en desarrollo**: En local (`NODE_ENV=development`) se utiliza `pino-pretty` para formatear los logs con colores legibles en la terminal sin alterar el formato JSON en producción.

---

## 2. Trazabilidad con Request ID (`x-request-id`)

Para diagnosticar incidentes cuando múltiples usuarios interactúan simultáneamente, cada petición entrante debe poseer un identificador único que se propague a través de todas las capas (middlewares, casos de uso y repositorios).

### Implementación del Logger Central: `apps/api/src/core/infrastructure/logging/logger.ts`

```typescript
import pino from "pino";
import { env } from "@/config/env";

export const logger = pino({
  level: env.NODE_ENV === "production" ? "info" : "debug",
  transport:
    env.NODE_ENV === "development"
      ? {
          target: "pino-pretty",
          options: {
            colorize: true,
            translateTime: "HH:MM:ss Z",
            ignore: "pid,hostname",
          },
        }
      : undefined,
  base: {
    env: env.NODE_ENV,
  },
});
```

### Middleware de Logging HTTP: `apps/api/src/core/presentation/middlewares/httpLogger.middleware.ts`

```typescript
import pinoHttp from "pino-http";
import { randomUUID } from "node:crypto";
import { logger } from "@/core/infrastructure/logging/logger";

export const httpLogger = pinoHttp({
  logger,
  // Asignar o reutilizar el Request ID proveniente del cliente o proxy inverso
  genReqId: (req, res) => {
    const existingId = req.headers["x-request-id"] as string;
    const reqId = existingId || `req_${randomUUID()}`;
    res.setHeader("x-request-id", reqId);
    return reqId;
  },
  // Redactar información confidencial para prevenir fugas en logs
  redact: ["req.headers.authorization", "req.headers.cookie", "req.body.password"],
  
  // Regla antipolución: ignorar sondas de salud automáticas para no saturar la consola
  autoLogging: {
    ignore: (req) => req.url === "/healthz" || req.url === "/ready",
  },
  
  customLogLevel: (req, res, err) => {
    if (res.statusCode >= 500 || err) return "error";
    if (res.statusCode >= 400) return "warn";
    return "info";
  },
});
```

---

## 3. Sondas de Salud: `/healthz` (Liveness) y `/ready` (Readiness)

Para que orquestadores (Docker Compose, Kubernetes, Coolify, Dokku) gestionen el ciclo de vida de los contenedores de forma autónoma, el backend debe exponer dos sondas diferenciadas.

```
       [ Orquestador / Docker ]
             |
             +---> GET /healthz (Liveness)
             |     ¿Node.js responde? (0.1 ms, memoria/CPU viva)
             |     Si falla 3 veces -> Reinicia el contenedor automáticamente
             |
             +---> GET /ready (Readiness)
                   ¿PostgreSQL y MinIO están conectados? (SELECT 1)
                   Si falla -> Bloquea el tráfico de usuarios hasta que esté listo
```

### Implementación del Router de Salud: `apps/api/src/presentation/routes/health.routes.ts`

```typescript
import { Router, Request, Response } from "express";
import { prisma } from "@/core/infrastructure/database/prisma.service";

export const healthRouter = Router();

/**
 * @openapi
 * /healthz:
 *   get:
 *     summary: Liveness probe (indica si el proceso de Node está vivo)
 */
healthRouter.get("/healthz", (_req: Request, res: Response) => {
  res.status(200).json({
    status: "ok",
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
  });
});

/**
 * @openapi
 * /ready:
 *   get:
 *     summary: Readiness probe (indica si las conexiones a BD y servicios están listas)
 */
healthRouter.get("/ready", async (_req: Request, res: Response) => {
  try {
    // 1. Verificar conectividad con PostgreSQL
    await prisma.$queryRaw`SELECT 1`;

    res.status(200).json({
      status: "ready",
      database: "connected",
      timestamp: new Date().toISOString(),
    });
  } catch (error) {
    res.status(503).json({
      status: "unhealthy",
      database: "disconnected",
      error: error instanceof Error ? error.message : "Database unavailable",
      timestamp: new Date().toISOString(),
    });
  }
});
```

---

## 4. Configuración del Healthcheck en Docker

### A. En el Dockerfile de la API (`apps/api/Dockerfile`):
Se añade la instrucción `HEALTHCHECK` nativa para que Docker monitoree el estado del proceso:

```dockerfile
# Monitoreo continuo de salud del contenedor
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:4000/healthz || exit 1
```

### B. En `docker-compose.yml` (o `.docker/dev/docker-compose.dev.yml`):
Permite que otros servicios esperen a que la base de datos o la API estén listas antes de arrancar:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build:
      context: ../..
      dockerfile: apps/api/Dockerfile
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/healthz"]
      interval: 15s
      timeout: 5s
      retries: 3
```

