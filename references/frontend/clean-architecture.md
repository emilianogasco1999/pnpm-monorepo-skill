# Frontend Clean Architecture (Feature-Sliced & Screaming Architecture)

Esta guía define de manera exhaustiva la arquitectura limpia por capas aplicada a módulos de interfaz dentro de `apps/web` (Next.js 16 + React 19). Cada feature del frontend debe replicar esta estructura con rigor y coherencia.

---

## 1. Principio Fundamental: Regla de Dependencia en el Cliente

Las dependencias fluyen estrictamente desde las capas exteriores hacia el núcleo del Dominio:

```
  [ Next.js App Router (src/app/) ]
                ↓
  [ Presentation (Pages, Components, Hooks, UI Store, Forms) ]
                ↓
  [ Application (Casos de Uso del Cliente) ]
                ↓
  [ Domain (Entidades, Value Objects, Interfaces de Repositorios) ]
                ↑
  [ Infrastructure (Implementación con Axios / core/api, Adapters, Mappers) ]
```

- **Domain**: Núcleo puro de negocio del cliente. No conoce React, JSX, hooks, Axios ni componentes de UI. Define entidades e interfaces de repositorios (`IRepository`).
- **Application**: Casos de uso (`use-cases`) que orquestan reglas del negocio en el navegador (ej: ejecutar login, limpiar sesión, procesar filtros de negocio).
- **Infrastructure**: Implementaciones concretas de acceso a datos (`RepositoryImpl`) consumiendo la API backend mediante `core/api/api.client.ts`. Mapea payloads HTTP a Entidades de Dominio.
- **Presentation**: Capa de experiencia de usuario. Contiene componentes de UI (Container / Presentational), páginas orquestadas, validaciones de formularios con Zod (`packages/shared`), queries y mutaciones con TanStack Query y estado local/global con Zustand.
- **Next.js App Router (`src/app/`)**: Capa de enrutamiento delgada. Solo contiene layouts, providers y páginas shell que delegan inmediatamente el render a la capa de presentación (`features/<feature>/presentation/pages`).

---

## 2. Topología Completa de la Aplicación (`apps/web/`)

La estructura de la aplicación web organiza los archivos de configuración, el empaquetado en contenedor y el código fuente estructurado en `src/`:

```
apps/web/
├── Dockerfile                  # Empaquetado multietapa para despliegue en producción
├── components.json             # Configuración de CLI y registry de Shadcn UI
├── next.config.ts              # Configuración y optimizaciones de Next.js
├── package.json                # Dependencias, scripts y vinculaciones del workspace
├── postcss.config.mjs          # Integración de TailwindCSS con PostCSS
├── tsconfig.json               # Configuración del compilador de TypeScript
└── src/                        # Código fuente modular de la aplicación
    ├── app/                    # Next.js 16 App Router: layouts, rutas, metadatos y shells
    ├── assets/                 # Recursos multimedia estáticos (logos, SVGs, imágenes, iconos)
    ├── components/             # Componentes compartidos del Design System (Form/, Table/, Theme/, ui/, app-sidebar)
    ├── core/                   # Núcleo transversal de la aplicación (API Client, providers, stores, errors)
    ├── features/               # Módulos verticales de negocio estructurados con Clean Architecture
    ├── hooks/                  # Custom hooks transversales de React (useMobile, useDebounce, etc.)
    └── lib/                    # Utilidades técnicas y helpers de librerías externas (cn() para Tailwind)
```

### Desglose de Carpetas de `src/`:
- **`app/`**: Capa de enrutamiento puro de Next.js (App Router). Contiene `layout.tsx`, `page.tsx`, `not-found.tsx` y rutas públicas o protegidas. **Nunca contiene lógica de negocio ni formularios pesados**; únicamente renderiza los shells y páginas desde `features/<feature>/presentation/pages/`.
- **`assets/`**: Archivos multimedia locales que no se sirven como estáticos públicos de CDN (logos corporativos, ilustraciones, iconos vectoriales).
- **`components/`**: Componentes visuales transversales del Design System reutilizables en cualquier parte del sistema:
  - `Form/`: Controles encapsulados con etiqueta, accesibilidad y mensajes de error (`InputField`, `SelectField`, etc.).
  - `Table/`: Primitivas reutilizables de tablas y paginadores.
  - `Theme/`: Controles de modo claro/oscuro.
  - `ui/`: Primitivas base instaladas por Shadcn UI (`Button`, `Card`, `Dialog`, etc.).
  - Componentes de layout global como `app-sidebar.tsx`, `nav-main.tsx`, `nav-user.tsx`, `ConfirmDialog.tsx`.
- **`core/`**: El corazón transversal de la aplicación (servicios compartidos por todas las features): cliente HTTP centralizado (`api/api.client.ts`), interceptores de sesión, stores globales de Zustand, configuración de providers y clases de error.
- **`features/`**: Módulos verticales de negocio desacoplados. Cada feature representa un dominio funcional (auth, usuarios, clientes, facturación) y se divide internamente en 4 capas de Clean Architecture.
- **`hooks/`**: Custom hooks de utilidad pura de React independientes de cualquier regla de negocio (ej: detección de pantallas móviles `use-mobile.tsx`, observadores de scroll, atajos de teclado).
- **`lib/`**: Helpers y configuraciones de bajo nivel de librerías de terceros (ej: `utils.ts` con la función `cn(...)` para combinar clases de Tailwind).

---

## 3. Anatomía Completa de una Feature (`src/features/<feature>/`)

Cada módulo vertical de negocio del frontend se organiza en cuatro capas estrictas:

```
src/features/<feature>/
├── domain/                      # 1. NÚCLEO PURO DE DOMINIO DEL CLIENTE
│   ├── entities/                # Modelos y entidades con identidad propia
│   ├── repositories/            # Interfaces / contratos de persistencia (IAuthRepository)
│   └── types/                   # Tipos TypeScript específicos del dominio de la feature
│
├── application/                 # 2. CASOS DE USO (ORQUESTADORES)
│   └── use-cases/               # Un archivo por acción concreta (LoginUseCase, GetProfileUseCase)
│
├── infrastructure/              # 3. TECNOLOGÍA Y COMUNICACIÓN HTTP
│   ├── repositories/            # Implementación concreta del repositorio usando Axios (AuthRepositoryImpl)
│   ├── adapters/                # Adaptadores de payloads de red y normalizadores
│   └── mappers/                 # Traductores de DTOs HTTP a Entidades de Dominio
│
└── presentation/                # 4. CAPA VISUAL Y DE INTERACCIÓN
    ├── components/              # Componentes de UI modulares (Container & Presentational)
    ├── pages/                   # Vistas completas de la feature (ej: LoginPage.tsx, UsersDashboardPage.tsx)
    ├── mutations/               # Custom hooks de mutación con TanStack Query (useLoginMutation.ts)
    ├── queries/                 # Custom hooks de consulta y cache con TanStack Query (useUsersQuery.ts)
    ├── validation/              # Esquemas de validación de formularios derivados de @<scope>/shared
    └── store/                   # Estado de UI específico de la feature (Zustand / local state)
```

---

## 4. Desglose Detallado por Capa

### A. Capa de Dominio (`domain/`)
- **Regla de Oro**: TypeScript 100% puro. **Prohibido importar React, hooks (`useState`, `useEffect`), librerías de UI o Axios**.
- **Entidades (`entities/`)**: Clases o interfaces que modelan los datos de negocio con sus invariantes.
- **Interfaces de Repositorios (`repositories/`)**: Contratos que la capa de infraestructura debe cumplir:
  ```typescript
  export interface IAuthRepository {
    login(credentials: LoginInput): Promise<AuthUser>;
    logout(): Promise<void>;
    getMe(): Promise<AuthUser>;
  }
  ```

### B. Capa de Aplicación (`application/`)
- **Responsabilidad**: Encapsular la lógica de qué sucede cuando el usuario realiza una acción de negocio.
- **Casos de Uso (`use-cases/`)**: Clases enfocadas en una sola tarea (`execute()`). Reciben sus dependencias (repositorios) por constructor:
  ```typescript
  export class LoginUseCase {
    constructor(private readonly authRepository: IAuthRepository) {}

    async execute(credentials: LoginInput): Promise<AuthUser> {
      const user = await this.authRepository.login(credentials);
      return user;
    }
  }
  ```

### C. Capa de Infraestructura (`infrastructure/`)
- **Responsabilidad**: Conectar el frontend con el mundo exterior (APIs REST, WebSockets, Storage).
- **Implementación de Repositorios (`repositories/`)**: Consume el cliente Axios centralizado (`core/api/api.client.ts`), maneja las respuestas HTTP y mapea los datos a las entidades de dominio:
  ```typescript
  export class AuthRepositoryImpl implements IAuthRepository {
    async login(credentials: LoginInput): Promise<AuthUser> {
      const { data } = await apiClient.post<LoginApiResponse>('/api/auth/login', credentials);
      return AuthMapper.toDomain(data);
    }
  }
  ```

### D. Capa de Presentación (`presentation/`)
- **Responsabilidad**: Renderizar la UI y reaccionar a los eventos del usuario.
- **Páginas (`pages/`)**: Ensamblan componentes, consumen los hooks de mutación/consulta y manejan el flujo visual.
- **Validación (`validation/`)**: Reutiliza los esquemas Zod canónicos de `packages/shared` para conectar formularios con `@hookform/resolvers/zod`.
- **Hooks de Datos (`queries/`, `mutations/`)**: TanStack Query envuelve la llamada al caso de uso, administrando `isPending`, `error`, invalidación de caché y toasts.

---

## 5. Núcleo Transversal (`src/core/`)

Para servicios globales y configuración compartida entre todas las features:

```
src/core/
├── api/                         # Cliente HTTP centralizado (Axios) con withCredentials, cola de refresh e interceptores
├── config/                      # Variables de entorno (env.ts), rutas de app (routes.ts), rutas API (api-routes.ts) y menú RBAC
├── domain/                      # Entidades y constantes transversales (ej: Roles, Estados)
├── errors/                      # Clases de error de UI (AppError, NotFoundError)
├── providers/                   # Providers globales (QueryClientProvider, ThemeProvider + MotionConfig, Toaster)
├── services/                    # Servicios transversales de cliente (ej: upload.service.ts para S3)
├── store/                       # Stores globales de Zustand (breadcrumbs dinámicos, preferencias de UI)
└── utils/                       # Utilidades puras compartidas (formateo de CUIT, normalizadores de paginación)
```

### Patrón Crítico en `core/api/api.client.ts`: Cola de Concurrencia para Silent Refresh
Cuando expira el Access Token y la pantalla dispara múltiples peticiones en paralelo que responden `401 Unauthorized`, el interceptor de Axios **no debe lanzar múltiples llamadas a `/auth/refresh`**. En su lugar:
1. Mantiene una bandera `isRefreshing` y un arreglo `failedQueue` de promesas pendientes.
2. La primera petición que recibe `401` ejecuta `executeSilentRefresh()` (promesa singleton).
3. Las demás peticiones `401` concurrentes se encolan en `failedQueue` y, apenas se resuelve el nuevo token, reintentan su petición original automáticamente con el nuevo header `Authorization: Bearer <newToken>`.

---

## 6. Componentes Compartidos y Primitivas de Formulario (`src/components/`)

Para no reinventar la rueda y mantener una interfaz visualmente armónica y accesible:
- **`src/components/Form/`**: Componentes de formulario de alto nivel con label, mensajes de error y accesibilidad integrados (`InputField`, `SelectField`, `SwitchField`, `SearchableSelectField`, `FileUploadDropzone`, `MultiFileUploadDropzone`, `FormModal`). **Es obligatorio reutilizarlos en los formularios de las features**.
- **`src/components/ui/`**: Primitivas base del design system construidas con Shadcn / Base UI (`Button`, `FieldGroup`, `Card`, `Dialog`, `Drawer`, etc.).
- **`src/components/Table/`**: Envoltorios de tablas paginadas y buscadores reutilizables (`DataTablePagination`, `DataTableSearch`).
- **Componentes Transversales de Resiliencia y Archivos**:
  - `PdfViewerModal.tsx`: Previsualizador de PDFs e imágenes mediante URLs prefirmadas de S3, con fallback automático a tarjeta de descarga en pantallas móviles (`< 768px`).
  - `ServerStatusChecker.tsx`: Monitor de disponibilidad que verifica periódicamente `/api/health` y muestra una pantalla de reconexión si el backend cae.

---

## 7. El Rol Delgado de Next.js App Router (`src/app/`)

El directorio `src/app/` actúa **exclusivamente como despachador de rutas**:

1. **`app/layout.tsx`**: Declara fuentes, metadatos SEO y envuelve la aplicación con los providers de `core/providers`.
2. **`app/**/page.tsx`**: Shells livianos. **No deben contener formularios extensos ni lógica de negocio**. Simplemente importan y renderizan la página correspondiente desde `presentation/pages/`:

```tsx
// src/app/login/page.tsx
import { LoginPage } from '@/features/auth/presentation/pages/LoginPage';

export default function Page() {
  return <LoginPage />;
}
```
