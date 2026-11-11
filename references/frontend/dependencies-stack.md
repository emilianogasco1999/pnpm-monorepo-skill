# Stack de Dependencias Esenciales del Frontend

Esta referencia técnica documenta el catálogo de librerías y dependencias que componen la aplicación cliente (`apps/web`), detallando su propósito arquitectónico, su ubicación en las capas y las mejores prácticas de uso.

---

## 1. Núcleo, Framework y Compilación

### `next` (16.x)
- **Propósito**: Framework de producción con App Router, renderizado híbrido y optimización de assets.
- **Uso en el sistema**: Gestiona el enrutamiento a través de `src/app/`, layouts anidados, carga dinámica de fuentes y generación de bundles optimizados.

### `react` / `react-dom` (19.x)
- **Propósito**: Motor de renderizado declarativo basado en componentes.
- **Uso en el sistema**: Construcción del árbol de componentes de UI y ciclo de vida de la interfaz.

### `typescript` (^5.x)
- **Propósito**: Tipado estático estricto.
- **Uso en el sistema**: Garantiza seguridad de tipos en todas las capas consumiendo las definiciones compartidas de `@<scope>/shared`.

---

## 2. Sistema de Diseño, Estilos y UI

### `tailwindcss` (v4)
- **Propósito**: Motor de utilidades CSS de alto rendimiento y cero configuración runtime.
- **Uso en el sistema**: Estilizado cohesivo de todos los componentes de interfaz mediante clases utilitarias y variables CSS globales (`src/app/globals.css`).

### `shadcn` / `@base-ui/react` / `@radix-ui/react-slot`
- **Propósito**: Primitivas de UI accesibles (WAI-ARIA), sin estilos impuestos y totalmente componibles.
- **Uso en el sistema**: Base para componentes interactivos complejos (Dialogs, Dropdowns, Tooltips, Tabs, Popovers) en `src/components/ui/`.

### `class-variance-authority`, `clsx`, `tailwind-merge`
- **Propósito**: Composición dinámica y segura de clases utilitarias.
- **Uso en el sistema**: Función helper `cn(...)` en `src/lib/utils.ts` para resolver variantes y conflictos de Tailwind sin colisiones.

### `lucide-react`
- **Propósito**: Set de iconografía SVG consistente y liviano.
- **Uso en el sistema**: Iconos visuales descriptivos para acciones, estados y navegación.

### `sonner`
- **Propósito**: Notificaciones toast modernas, no intrusivas y accesibles.
- **Uso en el sistema**: Feedback inmediato al usuario ante mutaciones exitosas o errores del servidor (`toast.success()`, `toast.error()`).

---

## 3. Manejo de Datos y Estado

### `@tanstack/react-query` (^5.x)
- **Propósito**: Gestión integral de **Server State** (caché asíncrono, sincronización en segundo plano, revalidación y reintentos).
- **Uso en el sistema**:
  - `useQuery` en `presentation/queries/`: Lectura y almacenamiento en caché de listas y detalles.
  - `useMutation` en `presentation/mutations/`: Creación, actualización y eliminación de datos con invalidación automática de queries (`queryClient.invalidateQueries`).

### `zustand` (^5.x)
- **Propósito**: Gestión liviana y predecible de **Client State** sin boilerplate ni context re-renders masivos.
- **Uso en el sistema**: Estado de sesión pública (datos del usuario autenticado, roles), preferencias de UI (modo oscuro, estado del sidebar).
- **Regla Crítica**: **NUNCA almacenar refresh tokens en Zustand persistido (`localStorage`)**. Los tokens de sesión sensibles viajan exclusivamente por cookies `httpOnly`.

---

## 4. Formularios y Validación

### `react-hook-form` (^7.x)
- **Propósito**: Manejo de formularios de alto rendimiento basado en inputs no controlados (*uncontrolled components*).
- **Uso en el sistema**: Control de estado, valores, errores y validación en tiempo real en los componentes de formulario (`presentation/components/`).

### `@hookform/resolvers` y `zod`
- **Propósito**: Conector entre React Hook Form y los esquemas de validación de Zod.
- **Uso en el sistema**: Integración directa con los esquemas canónicos de `@<scope>/shared`:
  ```typescript
  const form = useForm<LoginInput>({
    resolver: zodResolver(loginSchema),
  });
  ```

---

## 5. Comunicación de Red (HTTP)

### `axios` (^1.x)
- **Propósito**: Cliente HTTP robusto con soporte nativo de interceptores.
- **Uso en el sistema**: Centralizado en `src/core/api/api.client.ts`. Configurado obligatoriamente con:
  - `baseURL: process.env.NEXT_PUBLIC_API_URL`
  - `withCredentials: true` (permite el envío y recepción automática de cookies `httpOnly` para el refresh token).
  - Interceptor de respuesta para capturar errores `401 Unauthorized` y disparar la renovación automática del access token.

---

## 6. Visualización, Interactividad y Módulos Opcionales de Dominio

Las siguientes librerías se integran según las necesidades del proyecto (no es obligatorio instalarlas todas si el dominio no las requiere):

### `next-themes`, `motion`, `vaul` y `cmdk`
- **`next-themes`**: Gestión de modo claro/oscuro sin parpadeo de hidratación (SSR).
- **`motion`** (`motion/react`): Animaciones declarativas fluidas y respeto de accesibilidad (`MotionConfig reducedMotion="user"`).
- **`vaul`**: Drawers táctiles nativos para vistas móviles.
- **`cmdk`**: Paletas de comandos y selectores con búsqueda asíncrona (`SearchableSelectField`).

### `recharts` y `@tanstack/react-table`
- **`recharts`**: Gráficos analíticos declarativos para dashboards y reportes.
- **`@tanstack/react-table`** (o primitivas `Table` de Shadcn + `DataTablePagination`): Gestión de tablas paginadas, búsqueda y filtrado.

### `maplibre-gl` (Opcional — Sistemas con Geolocalización / GIS)
- **Propósito**: Renderizado de mapas vectoriales/satelitales interactivos y dibujo de polígonos GeoJSON (campos, lotes, rutas) sin depender de API keys pagas.

### `driver.js` (Opcional — Tours Guiados y Onboarding)
- **Propósito**: Recorridos interactivos paso a paso por módulo para capacitar al usuario final dentro de la propia interfaz.

### `date-fns` + Helpers de Zona Horaria (`src/lib/dates.ts`)
- **Propósito**: Formateo y manipulación de fechas evitando corrimientos de día (*off-by-one day shift*) por desfase UTC vs hora local (ej. fijando `America/Argentina/Buenos_Aires` / `UTC-3` al parsear cadenas `YYYY-MM-DD`).

---

## 7. Paquetes Locales del Monorepo

### `@<scope>/shared` (`workspace:*`)
- **Propósito**: Single Source of Truth (SSOT).
- **Uso en el sistema**: Contratos Zod para validación de formularios, tipos inferidos de TypeScript y constantes de roles y estados.

### `@<scope>/config` (`workspace:*`)
- **Propósito**: Configuraciones compartidas.
- **Uso en el sistema**: `tsconfig.react.json` para reglas estrictas de TypeScript en entornos React/DOM.

---

## 8. Prohibiciones Estrictas de Dependencias en `apps/web`

| Dependencia Prohibida | Razón de la Prohibición | Alternativa Correcta |
| :--- | :--- | :--- |
| `@prisma/client`, `prisma` | Librería exclusiva de servidor y acceso directo a base de datos. | Consumir la API REST a través de `core/api/api.client.ts`. |
| `bcrypt`, `jsonwebtoken` | Utilidades de encriptación de servidor y generación de JWTs. | La autenticación se valida en `apps/api`. |
| `express`, `cookie-parser` | Frameworks y utilidades de backend. | El enrutamiento web lo gestiona Next.js App Router. |
| Librerías con APIs de Node (`fs`, `net`, `child_process`) | Incompatibles con el entorno del navegador cliente. | Delegar procesamiento al backend. |
