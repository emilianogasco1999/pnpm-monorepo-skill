# Estrategia y Arquitectura de Testing en Monorepos Fullstack

Esta referencia técnica define los estándares de testing, la pirámide de pruebas, la topología de archivos y los patrones de implementación obligatorios para monorepos gestionados con **pnpm workspaces**, **Clean Architecture** en backend y **Feature-Driven** en frontend.

---

## 1. La Pirámide de Pruebas y Clean Architecture

El objetivo del testing no es alcanzar una métrica cosmética de cobertura, sino otorgar confianza de despliegue con la máxima velocidad de ejecución y el menor costo de mantenimiento.

```
       / \
      /   \      E2E Tests (10%)
     / E2E \     Playwright: Flujos críticos de usuario (Web + API + DB real)
    /-------\
   /         \    Integration Tests (20%)
  / Integrac. \   Vitest + Supertest / DB: Repositorios Prisma y Endpoints HTTP
 /-------------\
/               \  Unit Tests (70%)
/   Unitarios   \ Vitest: Casos de Uso, Entidades de Dominio, Esquemas Zod
----------------- (0 dependencias externas, ejecución en milisegundos)
```

### Principio de Responsabilidad por Nivel:
- **Unit Tests (Dominio y Aplicación)**: Validan reglas de negocio puras en aislamiento absoluto. No levantan servidores HTTP, no tocan Docker ni consultan bases de datos.
- **Integration Tests (Infraestructura y Presentación)**: Validan que los adaptadores hablen correctamente con dependencias externas (queries SQL de Prisma, transacciones, middlewares de Express, cookies de sesión).
- **E2E Tests (Flujo Completo)**: Validan journeys completos desde el navegador del cliente hasta la persistencia y respuesta visual.

---

## 2. Topología de Pruebas en el Monorepo

Para mantener un proyecto ordenado y evitar dispersión, los tests se organizan dentro de carpetas explícitas `tests/` en cada capa y feature.

```
<monorepo-root>/
├── vitest.workspace.ts                  # Orquestador global de workspaces de Vitest
├── packages/
│   └── shared/
│       └── tests/                       # Pruebas de esquemas Zod y utilidades puras
│           └── schemas/
│               └── user.schema.spec.ts
├── apps/
│   ├── api/
│   │   ├── vitest.config.ts             # Configuración de Vitest para entorno Node
│   │   ├── src/
│   │   │   └── features/
│   │   │       └── auth/
│   │   │           ├── application/
│   │   │           │   ├── login.use-case.ts
│   │   │           │   └── tests/       # Tests unitarios del caso de uso
│   │   │           │       └── login.use-case.spec.ts
│   │   │           ├── infrastructure/
│   │   │           │   ├── auth.repository.impl.ts
│   │   │           │   └── tests/       # Tests de integración contra BD
│   │   │           │       └── auth.repository.integration.spec.ts
│   │   │           └── presentation/
│   │   │               ├── auth.controller.ts
│   │   │               └── tests/       # Tests de endpoints HTTP (Supertest)
│   │   │                   └── auth.api.spec.ts
│   └── web/
│       ├── vitest.config.ts             # Configuración de Vitest con happy-dom / jsdom
│       ├── e2e/                         # Pruebas End-to-End con Playwright
│       │   └── auth.e2e.spec.ts
│       └── src/
│           └── features/
│               └── auth/
│                   └── presentation/
│                       └── tests/       # Tests de componentes y custom hooks
│                           ├── login-form.spec.tsx
│                           └── use-login-mutation.spec.ts
```

---

## 3. Configuración del Workspace de Pruebas (Vitest)

Se utiliza **Vitest** como motor unificado por su compatibilidad nativa con TypeScript, ESM y velocidad basada en Vite/esbuild.

### Archivo Raíz: `vitest.workspace.ts`
```typescript
import { defineWorkspace } from "vitest/config";

export default defineWorkspace([
  "packages/*",
  "apps/api/vitest.config.ts",
  "apps/web/vitest.config.ts",
]);
```

### Configuración Backend: `apps/api/vitest.config.ts`
```typescript
import { defineConfig } from "vitest/config";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths()],
  test: {
    name: "api",
    globals: true,
    environment: "node",
    include: ["src/**/tests/**/*.spec.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "json", "html"],
      include: ["src/**/*.ts"],
      exclude: ["src/**/tests/**", "src/index.ts"],
    },
  },
});
```

### Configuración Frontend: `apps/web/vitest.config.ts`
```typescript
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    name: "web",
    globals: true,
    environment: "happy-dom",
    setupFiles: ["./src/testing/setup.ts"],
    include: ["src/**/tests/**/*.spec.{ts,tsx}"],
  },
});
```

---

## 4. Patrón Canónico 1: Test Unitario de Caso de Uso (InMemory Repository)

El caso de uso sólo depende de la interfaz `IAuthRepository`. Para probarlo, implementamos un repositorio en memoria que simula el almacenamiento en un array simple.

### Implementación del Test: `apps/api/src/features/auth/application/tests/login.use-case.spec.ts`

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { LoginUseCase } from "../login.use-case";
import { IAuthRepository } from "../../domain/auth.repository.interface";
import { User } from "../../domain/user.entity";
import { ClientError } from "@/core/domain/errors/appErrors";

// 1. Double de prueba: Repositorio en Memoria
class InMemoryAuthRepository implements IAuthRepository {
  public users: User[] = [];

  async findByEmail(email: string): Promise<User | null> {
    const user = this.users.find((u) => u.email === email);
    return user ? { ...user } : null;
  }

  async save(user: User): Promise<User> {
    this.users.push(user);
    return user;
  }
}

// 2. Doubles para servicios auxiliares
const fakeHasher = {
  compare: async (plain: string, hashed: string) => plain === hashed,
  hash: async (plain: string) => `hashed_${plain}`,
};

const fakeTokenService = {
  generateAccessToken: (user: User) => `access_token_${user.id}`,
  generateRefreshToken: (user: User) => `refresh_token_${user.id}`,
};

describe("LoginUseCase", () => {
  let authRepository: InMemoryAuthRepository;
  let useCase: LoginUseCase;

  beforeEach(() => {
    authRepository = new InMemoryAuthRepository();
    useCase = new LoginUseCase(authRepository, fakeHasher, fakeTokenService);
  });

  it("debe autenticar exitosamente y retornar tokens cuando las credenciales son correctas", async () => {
    // Arrange: Crear usuario previo en memoria
    authRepository.users.push({
      id: "usr_123",
      email: "docente@alberdi.com",
      passwordHash: "hashed_password123",
      role: "DOCENTE",
      isActive: true,
    });

    // Act: Ejecutar caso de uso
    const result = await useCase.execute({
      email: "docente@alberdi.com",
      password: "password123",
    });

    // Assert: Validar resultado
    expect(result.user.id).toBe("usr_123");
    expect(result.accessToken).toBe("access_token_usr_123");
    expect(result.refreshToken).toBe("refresh_token_usr_123");
  });

  it("debe arrojar ClientError cuando el usuario no existe", async () => {
    // Act & Assert
    await expect(
      useCase.execute({
        email: "inexistente@alberdi.com",
        password: "password123",
      })
    ).rejects.toThrow(ClientError);
  });

  it("debe arrojar ClientError cuando la contraseña es inválida", async () => {
    authRepository.users.push({
      id: "usr_123",
      email: "docente@alberdi.com",
      passwordHash: "hashed_correctPassword",
      role: "DOCENTE",
      isActive: true,
    });

    await expect(
      useCase.execute({
        email: "docente@alberdi.com",
        password: "wrongPassword",
      })
    ).rejects.toThrow(ClientError);
  });
});
```

---

## 5. Patrón Canónico 2: Test de Integración de Persistencia (Infrastructure)

Valida que las consultas reales hacia la base de datos PostgreSQL mediante Prisma funcionen sin violaciones de schema ni fallos de mapeo.

### Archivo: `apps/api/src/features/auth/infrastructure/tests/auth.repository.integration.spec.ts`

```typescript
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";
import { PrismaClient } from "@prisma/client";
import { AuthRepositoryImpl } from "../auth.repository.impl";

describe("AuthRepositoryImpl (PostgreSQL / Prisma)", () => {
  let prisma: PrismaClient;
  let repository: AuthRepositoryImpl;

  beforeAll(async () => {
    prisma = new PrismaClient();
    await prisma.$connect();
    repository = new AuthRepositoryImpl(prisma);
  });

  afterAll(async () => {
    await prisma.$disconnect();
  });

  beforeEach(async () => {
    // Limpieza de datos entre ejecuciones
    await prisma.user.deleteMany();
  });

  it("debe persistir un nuevo usuario y recuperarlo por email", async () => {
    await prisma.user.create({
      data: {
        id: "usr_integration_1",
        email: "admin@alberdi.com",
        passwordHash: "secure_hash",
        role: "ADMINISTRADOR",
      },
    });

    const user = await repository.findByEmail("admin@alberdi.com");

    expect(user).not.toBeNull();
    expect(user?.id).toBe("usr_integration_1");
    expect(user?.role).toBe("ADMINISTRADOR");
  });
});
```

---

## 6. Patrón Canónico 3: Test de API y Endpoints (Presentation / Supertest)

Valida la integración del router Express, middlewares de validación Zod (`validateRequest`), autenticación y códigos HTTP.

### Archivo: `apps/api/src/features/auth/presentation/tests/auth.api.spec.ts`

```typescript
import { describe, it, expect } from "vitest";
import request from "supertest";
import { app } from "@/src/app"; // Instancia Express exportada sin .listen()

describe("POST /api/auth/login", () => {
  it("debe responder 400 Bad Request si el payload no cumple el esquema Zod", async () => {
    const response = await request(app)
      .post("/api/auth/login")
      .send({ email: "correo-invalido" }); // Falta contraseña y email tiene formato erróneo

    expect(response.status).toBe(400);
    expect(response.body.errors).toBeDefined();
  });

  it("debe emitir cookie httpOnly con el refresh token tras autenticación exitosa", async () => {
    const response = await request(app)
      .post("/api/auth/login")
      .send({ email: "valido@alberdi.com", password: "PasswordValido123!" });

    if (response.status === 200) {
      const cookies = response.headers["set-cookie"];
      expect(cookies).toBeDefined();
      expect(cookies[0]).toMatch(/refreshToken=/);
      expect(cookies[0]).toMatch(/HttpOnly/i);
    }
  });
});
```

---

## 7. Patrón Canónico 4: Testing de Frontend con MSW y Testing Library

En `apps/web`, las llamadas HTTP se interceptan con **Mock Service Worker (MSW)** para probar componentes de React sin depender de que el backend esté ejecutándose.

### Archivo: `apps/web/src/features/auth/presentation/tests/login-form.spec.tsx`

```tsx
import { describe, it, expect } from "vitest";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import { LoginForm } from "../components/LoginForm";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const renderWithProviders = (ui: React.ReactElement) => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return render(
    <QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>
  );
};

describe("LoginForm Component", () => {
  it("debe mostrar errores de validación de Zod cuando los campos están vacíos", async () => {
    renderWithProviders(<LoginForm />);

    const submitButton = screen.getByRole("button", { name: /iniciar sesión/i });
    fireEvent.click(submitButton);

    await waitFor(() => {
      expect(screen.getByText(/el email es requerido/i)).toBeInTheDocument();
    });
  });
});
```

---

## 8. Comandos Canónicos de Testing en el Workspace

En el `package.json` raíz del monorepo:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:unit": "vitest run --exclude=\"**/*.integration.spec.ts\" --exclude=\"**/*.api.spec.ts\"",
    "test:integration": "vitest run --include=\"**/*.integration.spec.ts\" --include=\"**/*.api.spec.ts\"",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "pnpm --filter \"@<scope>/web\" exec playwright test"
  }
}
```

### Ejecución Filtrada:
```bash
# Correr tests únicamente de la API
pnpm --filter "@<scope>/api" test

# Correr tests únicamente de packages/shared
pnpm --filter "@<scope>/shared" test
```

