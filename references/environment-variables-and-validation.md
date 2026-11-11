# Validación de Variables de Entorno en Runtime (Patrón Fail-Fast)

Esta referencia técnica establece los estándares para la declaración, tipado estricto y validación en tiempo de ejecución de variables de entorno en proyectos monorepo fullstack (**Node/Express** y **Next.js**).

---

## 1. El Principio Fail-Fast

En aplicaciones Node.js convencionales, el objeto global `process.env` tipa todas sus claves como `string | undefined`. Esto genera dos problemas arquitectónicos graves:

1. **Fallas Silenciosas Tardías**: Si falta una variable crítica (ej. `JWT_SECRET` o `DATABASE_URL`), el proceso arranca normalmente. El error 500 se produce horas después cuando un usuario intenta ejecutar una acción que la requiere.
2. **Fuga de Secretos en Frontend**: En Next.js, utilizar accidentalmente una variable de servidor (`DATABASE_URL`, credenciales de API privadas) en un componente cliente puede filtrar credenciales en el bundle público de JavaScript del navegador.

> **Principio Fail-Fast**: Si falta una sola variable de entorno obligatoria o su formato es inválido (ej. un puerto que no es numérico o una URL mal formada), **el proceso debe abortar en el milisegundo cero (`process.exit(1)`)**, impidiendo que la aplicación inicie y reportando en consola exactamente qué variable falló.

---

## 2. Implementación Canónica en Backend (`apps/api`)

En `apps/api`, la validación se centraliza en `src/config/env.ts` utilizando **Zod**. Ningún archivo de la aplicación debe leer `process.env` directamente; todos deben importar el objeto `env` validado y fuertemente tipado.

### Archivo: `apps/api/src/config/env.ts`

```typescript
import { z } from "zod";
import dotenv from "dotenv";

// Cargar variables del archivo .env si estamos en desarrollo local
dotenv.config();

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  PORT_API: z.coerce.number().default(4000),
  DATABASE_URL: z.string({ message: "DATABASE_URL es obligatoria" }).url(),
  
  // Seguridad y JWT
  JWT_SECRET: z.string().min(16, "JWT_SECRET debe tener al menos 16 caracteres"),
  JWT_REFRESH_SECRET: z.string().min(16, "JWT_REFRESH_SECRET debe tener al menos 16 caracteres"),
  ACCESS_TOKEN_EXPIRES: z.string().default("15m"),
  REFRESH_TOKEN_EXPIRES: z.string().default("7d"),
  
  // CORS y cliente
  CORS_ORIGIN: z.string().default("http://localhost:3000"),
  
  // Almacenamiento S3 / MinIO
  S3_ENDPOINT: z.string().url(),
  S3_ACCESS_KEY: z.string().min(1),
  S3_SECRET_KEY: z.string().min(1),
  S3_BUCKET: z.string().min(1),
  S3_REGION: z.string().default("us-east-1"),
});

const parseEnv = () => {
  const result = envSchema.safeParse(process.env);

  if (!result.success) {
    console.error("❌ ERROR CRÍTICO: Variables de entorno inválidas o faltantes en apps/api:");
    console.error(JSON.stringify(result.error.format(), null, 2));
    process.exit(1);
  }

  return result.data;
};

// Exportación única inmutable y tipada
export const env = parseEnv();
export type Env = z.infer<typeof envSchema>;
```

### Uso en el arranque del servidor (`apps/api/src/index.ts`):
```typescript
import { env } from "@/config/env"; // Al importar env, se valida automáticamente antes de montar Express
import express from "express";

const app = express();

app.listen(env.PORT_API, () => {
  console.log(`🚀 API ejecutándose en el puerto ${env.PORT_API} [${env.NODE_ENV}]`);
});
```

---

## 3. Implementación Canónica en Frontend (`apps/web`)

En Next.js se utiliza la librería estándar del ecosistema **`@t3-oss/env-nextjs`** junto con Zod. Esta librería proporciona:
- **Separación estricta Servidor vs Cliente**: Las variables sin el prefijo `NEXT_PUBLIC_` quedan completamente bloqueadas en componentes del cliente.
- **Validación en Build**: Si falta una variable durante `next build`, la compilación falla inmediatamente en el pipeline de CI.

### Archivo: `apps/web/src/core/config/env.ts`

```typescript
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  server: {
    NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
    // URL interna directa para llamadas SSR hacia la API (sin salir a internet)
    INTERNAL_API_URL: z.string().url().default("http://127.0.0.1:4000"),
  },
  client: {
    // Variables públicas que se exponen al navegador (deben llevar prefijo NEXT_PUBLIC_)
    NEXT_PUBLIC_API_URL: z.string().url().default("http://localhost:4000"),
    NEXT_PUBLIC_APP_NAME: z.string().default("Mi Alberdi"),
  },
  runtimeEnv: {
    NODE_ENV: process.env.NODE_ENV,
    INTERNAL_API_URL: process.env.INTERNAL_API_URL,
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
    NEXT_PUBLIC_APP_NAME: process.env.NEXT_PUBLIC_APP_NAME,
  },
  emptyStringAsUndefined: true,
});
```

### Regla de Consumo en Frontend:
- Para llamadas desde el navegador (Axios/TanStack Query):
  ```typescript
  import { env } from "@/core/config/env";
  const apiClient = axios.create({ baseURL: env.NEXT_PUBLIC_API_URL });
  ```
- Para llamadas en Server Components o Route Handlers:
  ```typescript
  import { env } from "@/core/config/env";
  const res = await fetch(`${env.INTERNAL_API_URL}/api/users`);
  ```

---

## 4. Regla de Sincronización con `.env.example`

El archivo `.env.example` en la raíz del monorepo es el **contrato documental** para los desarrolladores y agentes de IA.

1. **Paridad 1:1**: Toda variable definida en los esquemas Zod de `apps/api/src/config/env.ts` o `apps/web/src/core/config/env.ts` **DEBE** existir con un valor de ejemplo funcional en `.env.example`.
2. **Prohibición de Secretos Reales**: Nunca commitear claves privadas reales, tokens de producción ni contraseñas auténticas en `.env.example`. Utilizar siempre placeholders o valores válidos para el entorno Docker local (`app_user`, `admin123`, etc.).
3. **Exclusión Estricta en Git**: Los archivos `.env`, `.env.local` y `.env.production` están prohibidos en el repositorio de Git y deben estar listados en `.gitignore`.

