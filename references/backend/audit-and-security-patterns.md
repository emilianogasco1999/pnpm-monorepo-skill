# Patrones de Auditoría Transversal y Seguridad de Sesión en Backend

Esta referencia técnica documenta patrones avanzados de arquitectura e infraestructura para `apps/api` que resuelven problemas reales de producción (trazabilidad automática, condiciones de carrera en rotación de tokens y protección contra fuerza bruta) sin contaminar la capa de Dominio ni los Casos de Uso.

---

## 1. Auditoría Automática Transversal (`AsyncLocalStorage` + ORM Interceptor)

### El Problema
Pasar el `userId` manualmente a cada método de repositorio o llamar a un servicio de auditoría dentro de cada Caso de Uso ensucia la lógica de negocio y es propenso a olvidos.

### La Solución Arquitectónica
Combinar **`AsyncLocalStorage`** nativo de Node.js (`node:async_hooks`) con un interceptor global en la capa de persistencia (por ejemplo, `$extends` en Prisma o subscribers/hooks en otros ORMs).

```
[ Petición HTTP con JWT ]
           ↓
[ auditContext.middleware.ts ] ──► Extrae userId y abre contexto AsyncLocalStorage.run({ userId })
           ↓
[ Controller ──► UseCase ──► Repository ] (No saben que están siendo auditados)
           ↓
[ prisma.service.ts ($extends) ] ──► Lee userId de AsyncLocalStorage
           │                         Captura oldValues (antes) y newValues (después)
           ▼
[ Tabla AuditLog (CREATE / UPDATE / DELETE / REVERT) ]
```

### A. Contexto Asíncrono (`src/core/infrastructure/context/asyncContext.ts`)
```typescript
import { AsyncLocalStorage } from 'node:async_hooks';

export interface AppAsyncContext {
  userId?: string;
}

export const asyncLocalStorage = new AsyncLocalStorage<AppAsyncContext>();

export const getCurrentUserId = (): string | undefined => {
  return asyncLocalStorage.getStore()?.userId;
};
```

### B. Middleware de Inyección de Contexto (`src/core/presentation/middlewares/auditContext.middleware.ts`)
Se registra de forma global en `src/index.ts` antes de las rutas:
```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { asyncLocalStorage } from '@core/infrastructure/context/asyncContext.js';
import { envs } from '@config/environment.js';

export const auditContextMiddleware = (req: Request, _res: Response, next: NextFunction) => {
  let userId: string | undefined;

  const authHeader = req.headers.authorization;
  if (authHeader?.startsWith('Bearer ')) {
    const token = authHeader.split(' ')[1];
    try {
      const decoded = jwt.verify(token, envs.JWT_SECRET) as { id?: string };
      userId = decoded.id;
    } catch {
      // Si el token expiró o es inválido, authMiddleware lo rechazará más adelante en rutas protegidas
    }
  }

  asyncLocalStorage.run({ userId }, () => next());
};
```

### C. Extensión del Cliente ORM (`src/core/infrastructure/database/prisma.service.ts`)
Intercepta mutaciones (`create`, `update`, `delete`, `upsert`) excluyendo tablas técnicas (`AuditLog`, `RefreshToken`), consulta el estado previo (`oldValues`) y registra el cambio con el `userId` obtenido desde `getCurrentUserId()`.

---

## 2. Seguridad de Sesión y Rotación de Refresh Tokens Resiliente a Concurrencia

### El Problema de las Peticiones Concurrentes
En aplicaciones SPA/SSR modernas (Next.js + TanStack Query), una pantalla puede disparar 4 o 5 peticiones simultáneas cuando el Access Token acaba de expirar. Si dos peticiones intentan rotar el mismo Refresh Token con milisegundos de diferencia:
1. La primera petición rota el token en la base de datos e invalida el anterior.
2. La segunda petición llega con el token recién invalidado, disparando una **falsa alarma de robo de token (RFC 6819)** y cerrando la sesión del usuario.

### Capas de Protección Recomendadas

1. **Deduplicación de Promesas en Vuelo (`inFlightRefreshes`)**:
   - Un `Map<string, Promise<RefreshResult>>` indexado por el hash del token. Si llegan dos peticiones idénticas al mismo tiempo al backend, ambas esperan la misma promesa en memoria.
2. **Ventana de Gracia Post-Rotación (`rotatedTokensGraceMap`)**:
   - Un cache temporal en memoria (ej. **30 segundos**) que guarda el resultado de un token recién rotado. Si una petición rezagada llega dentro de los 30s con el token anterior, recibe el nuevo par de tokens sin disparar la alerta de robo.
   - Pasados los 30s, cualquier reutilización de un token ya rotado se considera compromiso de credenciales, se incrementa `tokenVersion` del usuario y se revocan todas sus sesiones.
3. **Revocación Inmediata vía `tokenVersion`**:
   - El modelo `User` incluye un entero `tokenVersion` (incluido también en el payload del JWT).
   - Ante cambio de contraseña, desactivación de usuario o detección de robo de sesión, basta incrementar `tokenVersion` para invalidar instantáneamente todos los JWT emitidos previamente.
4. **Vinculación de Dispositivo (`userAgentHash`)**:
   - Se almacena un hash SHA-256 del `User-Agent` dentro del token para evitar que un token extraído sea reutilizado desde un cliente con firma distinta.

---

## 3. Rate Limiting Diferenciado por Superficie (`express-rate-limit`)

Nunca aplicar el mismo límite de tráfico al login que a los endpoints operativos de consulta:

```typescript
import rateLimit from 'express-rate-limit';

// 1. Límite general para la API operativa (ej: 120 peticiones por minuto por IP)
export const generalRateLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 120,
  standardHeaders: true,
  legacyHeaders: false,
  message: { status: 'error', message: 'Demasiadas peticiones, intente nuevamente en un minuto.' },
});

// 2. Límite estricto para autenticación (ej: 10 intentos fallidos cada 15 minutos)
export const authRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  skipSuccessfulRequests: true, // No penaliza inicios de sesión exitosos
  standardHeaders: true,
  legacyHeaders: false,
  message: { status: 'error', message: 'Demasiados intentos de inicio de sesión. Bloqueado temporalmente.' },
});
```

> [!IMPORTANT]
> Cuando la API corre detrás de un proxy inverso (Nginx, Traefik, Cloudflare o Docker), configurar obligatoriamente `app.set('trust proxy', 1)` en `src/index.ts` para que el rate limiter identifique la IP real del cliente (`X-Forwarded-For`) y no bloquee la IP interna del contenedor proxy.

