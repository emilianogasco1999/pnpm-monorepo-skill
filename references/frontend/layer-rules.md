# Matriz de Reglas de Dependencias por Capa en Frontend (Clean Architecture)

Esta guía define con rigor qué importaciones están permitidas y cuáles quedan estrictamente prohibidas en cada capa de `apps/web` para preservar la pureza arquitectónica y evitar el acoplamiento a librerías de UI o clientes de red.

---

## 1. Matriz de Importaciones Permitidas

| Capa | Puede Importar de: | NUNCA Puede Importar de: |
| :--- | :--- | :--- |
| **`domain/`** | - Entidades y tipos locales de la feature<br>- Interfaces de repositorios (`repositories/`)<br>- Constantes puras de `@<scope>/shared` | - `react`, hooks (`useState`, `useEffect`)<br>- `axios`, `fetch` o clientes HTTP<br>- `application/`, `infrastructure/` o `presentation/`<br>- Componentes de UI, Tailwind o Shadcn |
| **`application/`** | - `domain/` (Entidades, Interfaces de Repositorios)<br>- Tipos inferidos de Zod (`@<scope>/shared`)<br>- Errores de aplicación de `core/errors/` | - `react`, JSX o custom hooks<br>- `infrastructure/` (repositorios concretos no deben importarse directo, se reciben por inyección de dependencias)<br>- `presentation/` o componentes de UI<br>- `axios` directamente |
| **`infrastructure/`** | - `domain/` (Entidades e Interfaces que implementa)<br>- `core/api/api.client.ts` o Axios<br>- Mappers y adaptadores de transporte de red | - `react`, JSX o componentes de UI<br>- `presentation/` (páginas, componentes)<br>- Casos de uso de `application/` |
| **`presentation/`** | - `application/` (Casos de uso para instanciar y ejecutar)<br>- `domain/` (Entidades y tipos)<br>- Contratos Zod de `@<scope>/shared`<br>- `@tanstack/react-query`, `zustand`<br>- `react-hook-form`, `@hookform/resolvers/zod`<br>- Componentes UI (`components/ui/`, Lucide, Sonner) | - `core/api/api.client.ts` directamente (hacer peticiones salteando el caso de uso y el repositorio)<br>- Acceso directo a base de datos o APIs sin pasar por la capa de infraestructura |
| **`src/app/` (Router)** | - `features/<feature>/presentation/pages/`<br>- Providers de `core/providers/`<br>- Layouts y componentes estructurales globales | - Casos de uso o repositorios directamente<br>- Formularios extensos o lógica de negocio dentro de `page.tsx` |

---

## 2. Anti-Patrones Prohibidos en el Frontend

1. **Llamadas directas de Axios dentro de Componentes**:
   - ❌ `const res = await axios.post('/api/users', data);` dentro de un botón o `useEffect`.
   - ✅ El componente invoca un hook de mutación (`useCreateUserMutation`), el hook ejecuta el caso de uso (`CreateUserUseCase`), y el caso de uso delega al repositorio (`UserRepositoryImpl`).

2. **Mezclar React o JSX dentro del Caso de Uso o Dominio**:
   - ❌ Usar `useState`, `useEffect` o importar JSX en `domain/` o `application/`.
   - ✅ Los casos de uso son clases o funciones TypeScript puras; su lógica es 100% testeable sin montar el DOM de React.

3. **Almacenar Refresh Tokens en `localStorage` o Zustand Persistido**:
   - ❌ Guardar tokens de larga vida en el almacenamiento local del navegador (vulnerabilidad crítica ante ataques XSS).
   - ✅ Los tokens de refresco viajan exclusivamente en cookies `httpOnly` seguras emitidas por la API.

4. **Duplicar Esquemas de Validación en el Cliente**:
   - ❌ Volver a declarar `{ email: z.string().email() }` en cada componente de formulario.
   - ✅ Importar el esquema canónico desde `@<scope>/shared` y consumirlo mediante `zodResolver(schema)`.

5. **Engordar las Rutas de Next.js (`src/app/`)**:
   - ❌ Crear archivos `page.tsx` de 300 líneas con formularios, estados, modales y lógica de negocio.
   - ✅ `page.tsx` debe ser un archivo de menos de 15 líneas que únicamente renderice la página modular desde `features/<feature>/presentation/pages/`.

6. **Reinventar Componentes de Formulario Base**:
   - ❌ Escribir etiquetas HTML nativas `<input>`, `<select>` o divs manuales de error en formularios de features.
   - ✅ Reutilizar siempre los componentes encapsulados de `@/components/Form/` (`InputField`, `SelectField`, `CheckboxField`, `SearchableSelectField`) y primitivas de `@/components/ui/` (`Button`, `FieldGroup`), garantizando accesibilidad, feedback homogéneo de validación y coherencia visual en todo el sistema.
