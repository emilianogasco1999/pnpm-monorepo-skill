# Matriz de Reglas de Dependencias por Capa (Clean Architecture)

Esta guía define qué importaciones están permitidas y cuáles quedan estrictamente prohibidas en cada capa para preservar la pureza arquitectónica y el desacoplamiento.

---

## 1. Matriz de Importaciones Permitidas

| Capa | Puede Importar de: | NUNCA Puede Importar de: |
| :--- | :--- | :--- |
| **`domain/`** | - Objetos de valor locales (`value-objects/`)<br>- Interfaces de su propio dominio<br>- Constantes puras de `@<scope>/shared` | - `express` (`Request`, `Response`)<br>- `@prisma/client` o cualquier ORM<br>- `application/`, `infrastructure/` o `presentation/`<br>- SDKs de red, bases de datos o storage |
| **`application/`** | - `domain/` (Entidades, Interfaces de Repositorios)<br>- DTOs y tipos inferidos de Zod (`@<scope>/shared`)<br>- Errores de aplicación (`core/domain/errors`) | - `express` (`req`, `res`, `next`)<br>- `infrastructure/` (repositorios concretos de Prisma)<br>- `@prisma/client` directamente<br>- `presentation/` |
| **`infrastructure/`**| - `domain/` (Entidades e Interfaces que implementa)<br>- `@prisma/client` / pool de PostgreSQL<br>- Clientes de AWS / S3 / SDKs externos<br>- Adaptadores de base de datos (`@prisma/adapter-pg`) | - `express` (`Request`, `Response`)<br>- `presentation/` (controladores o rutas)<br>- Casos de uso de `application/` |
| **`presentation/`**  | - `application/` (Casos de uso para instanciar y ejecutar)<br>- `domain/` (únicamente tipos o value objects)<br>- `express` (`Request`, `Response`, `Router`)<br>- Middlewares de `core/presentation` | - `@prisma/client` para ejecutar queries directas<br>- `infrastructure/` para saltear casos de uso<br>- Lógica de negocio dentro de funciones de controlador |

---

## 2. Anti-Patrones Prohibidos

1. **Consulta directa a Prisma en el Controlador**: Todo acceso a datos debe atravesar el caso de uso y el repositorio.
2. **Exponer objetos de transporte (`req`, `res`) al Caso de Uso**: Los casos de uso deben recibir DTOs tipados e independientes de HTTP.
3. **Acoplar Entidades de Dominio al esquema del ORM**: Las entidades son modelos de negocio independientes; la capa de infraestructura utiliza `mappers/` para traducir entre modelos de Prisma y Entidades de Dominio.
