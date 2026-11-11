# Backend Clean Architecture (Feature-Sliced & Screaming Architecture)

Esta guía define de manera exhaustiva la arquitectura limpia por capas aplicada a módulos de negocio dentro de `apps/api`. Cada feature del sistema debe replicar esta estructura con rigor.

---

## 1. Principio Fundamental: Regla de Dependencia

Las dependencias fluyen estrictamente desde las capas exteriores hacia el núcleo del Dominio:

```
  [ Presentation (HTTP, Controladores, Rutas, Docs) ]
                           ↓
  [ Infrastructure (Prisma, Storage S3, Workers, Services) ]
                           ↓
  [ Application (Casos de Uso) ]
                           ↓
  [ Domain (Entidades, Value Objects, Interfaces) ]
```

- **Domain**: Núcleo puro de negocio. No conoce Express, Prisma, HTTP, librerías de red ni bases de datos.
- **Application**: Casos de uso de negocio que orquestan el dominio a través de contratos (interfaces).
- **Infrastructure**: Implementaciones concretas de base de datos, clientes de terceros, workers asíncronos y utilidades técnicas.
- **Presentation**: Capa de entrega HTTP con Express, controladores, tubería de middlewares y documentación Swagger.

---

## 2. Anatomía Completa de una Feature (`features/<feature>/`)

Cada módulo vertical de negocio se divide en cuatro capas estrictas con las siguientes subcarpetas obligatorias:

```
src/features/<feature>/
├── domain/                      # 1. NÚCLEO PURO DE DOMINIO
│   ├── constants/               # Enums y constantes exclusivas del módulo
│   ├── entities/                # Modelos y entidades de negocio
│   ├── repositories/            # Interfaces / contratos de persistencia (IRepository)
│   └── value-objects/           # Objetos de valor inmutables y reglas de validación
│
├── application/                 # 2. CASOS DE USO
│   └── use-cases/               # Un archivo por acción concreta del usuario (Single Responsibility)
│
├── infrastructure/              # 3. TECNOLOGÍA Y PERSISTENCIA
│   ├── repositories/            # Implementación concreta del repositorio usando Prisma
│   ├── mappers/                 # Traductores de filas de Prisma a Entidades de Dominio
│   ├── services/                # Clientes de APIs externas, SDKs (ej: ARCA, correos, etc.)
│   ├── workers/                 # Procesadores asíncronos en segundo plano / Background workers
│   └── utils/                   # Utilidades técnicas específicas de la infraestructura
│
└── presentation/                # 4. CAPA DE ENTRADA HTTP
    ├── controllers/             # Manejadores que reciben Request y retornan Response
    ├── routes/                  # Definición de rutas Express con middlewares de seguridad
    └── docs/                    # Documentación Swagger / OpenAPI (JSDoc)
```

---

## 3. Desglose Detallado por Capa

### A. Capa de Dominio (`domain/`)
- **`constants/`**: Valores inmutables propios del módulo (ej: estados internos, códigos de error de negocio).
- **`entities/`**: Clases o interfaces que encapsulan los datos esenciales y las invariantes de negocio.
- **`repositories/`**: **Interfaces puras** de TypeScript (ej: `IUserRepository`). Declaran la firma de los métodos (`findById`, `save`, `delete`) sin detalles de SQL o Prisma.
- **`value-objects/`**: Encapsulan atributos con validación propia (ej: `UserRole.ts`, `CUIT.ts`, `MontoMonetario.ts`).

### B. Capa de Aplicación (`application/`)
- **`use-cases/`**: Orquestadores de negocio. Cada caso de uso realiza **una única acción**:
  - `CreateUserUseCase.ts`, `EmitirFacturaUseCase.ts`, `BuscarClientePorDniUseCase.ts`.
  - Recibe por inyección de dependencias en el constructor las interfaces de dominio necesarias.
  - Expone un único método público `execute(...)`.
  - Lanza excepciones controladas (`ClientError`, `NotFoundError`).

### C. Capa de Infraestructura (`infrastructure/`)
- **`repositories/`**: Implementan las interfaces de dominio comunicándose con la base de datos a través de Prisma (ej: `UserRepositoryImpl implements IUserRepository`).
- **`mappers/`**: Convierten registros planos del ORM a Entidades del Dominio (`UserMapper.toDomain(prismaUser)`).
- **`services/`**: Adaptadores y SDKs para servicios de terceros (ej: `arca.service.ts` para AFIP/ARCA, clientes SMTP, etc.).
- **`workers/`**: Procesadores que se ejecutan en segundo plano fuera del ciclo de petición HTTP (ej: `arcaBackgroundWorker.ts` procesando colas con backoff exponencial).
- **`utils/`**: Funciones de bajo nivel como formateo de certificados, generación de firmas o serialización técnica.

### D. Capa de Presentación (`presentation/`)
- **`controllers/`**: Extraen parámetros de `req.body`, `req.query`, `req.params`, llaman a los casos de uso y formatean la respuesta con `response(res, statusCode, message, data)`.
- **`routes/`**: Instancian el router de Express y encadenan la tubería de seguridad:
  1. `authMiddleware`: Valida el JWT y la sesión del usuario.
  2. `requireRole(['ADMINISTRADOR', ...])`: Autorización RBAC.
  3. `validateRequest(schema)`: Valida el payload de entrada contra el esquema Zod compartido.
  4. `catchedAsync(...)`: Envoltorio que captura errores asíncronos y los deriva al `errorHandler`.
- **`docs/` (Documentación Swagger / OpenAPI)**: Archivos `.docs.ts` que describen los endpoints para Swagger UI mediante bloques JSDoc `@swagger`.

---

## 4. Estándar de Documentación Swagger (`presentation/docs/`)

Toda ruta de la API debe estar documentada en su respectivo archivo `presentation/docs/<feature>.docs.ts`. 

### Estructura Canónica de un bloque Swagger:
```typescript
/**
 * @swagger
 * /api/usuarios:
 *   post:
 *     summary: Crea un nuevo usuario en el sistema
 *     tags: [Usuarios]
 *     security:
 *       - bearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required:
 *               - email
 *               - password
 *               - rol
 *             properties:
 *               email:
 *                 type: string
 *                 format: email
 *               password:
 *                 type: string
 *                 minLength: 6
 *               rol:
 *                 type: string
 *                 enum: [ADMINISTRADOR, CONTADOR]
 *     responses:
 *       201:
 *         description: Usuario creado exitosamente
 *       400:
 *         description: Datos de entrada inválidos (error de validación Zod)
 *       401:
 *         description: No autenticado
 *       403:
 *         description: Permiso denegado (requiere rol específico)
 *       409:
 *         description: Conflicto (email ya registrado)
 */
```

### Reglas para Swagger:
1. **Etiquetas (`tags`)**: Agrupan endpoints por módulo de negocio.
2. **Seguridad (`security`)**: Indicar `- bearerAuth: []` en todo endpoint protegido por `authMiddleware`.
3. **Parámetros (`parameters`)**: Especificar tipo, descripción y defaults para parámetros en `path` o `query`.
4. **Respuestas (`responses`)**: Documentar el código de éxito (200, 201) y los posibles códigos de error (400, 401, 403, 404, 409).

---

## 5. Arquitectura del Núcleo Transversal (`src/core/`)

Lo que no pertenece a una feature en particular reside en `src/core/`:
- `core/domain/errors/`: Jerarquía de errores (`AppError`, `ClientError`, `NotFoundError`, `UnauthorizedError`).
- `core/infrastructure/`:
  - `context/asyncContext.ts`: Contexto asíncrono con `AsyncLocalStorage` para propagar el `userId` de la petición actual hacia la base de datos sin acoplar capas.
  - `database/prisma.service.ts`: Instancia global con pool de PostgreSQL y extensión (`$extends`) para **auditoría automática transversal** (`AuditLog`).
  - `storage/s3Storage.service.ts` y `adjunto.service.ts`: Cliente S3/MinIO desacoplado y servicio polimórfico de archivos adjuntos (`entidadTipo`, `entidadId`).
- `core/presentation/`:
  - `middlewares/`: `authMiddleware`, `permission.middleware` (`requireRole`), `validateRequest`, `rateLimiter.middleware` (`generalRateLimiter`, `authRateLimiter`), `auditContext.middleware` y `errorHandler`.
  - `routes/upload.routes.ts` y `controllers/upload.controller.ts`: Endpoints transversales de subida y generación de URLs prefirmadas.
  - `utils/`: `catchedAsync`, `response`, `resError`, `statusCodes`.

---

## 6. Referencias Técnicas y Ejemplos de Código

- [Ejemplos Canónicos por Capa (Golden Examples)](layer-examples.md): Snippets funcionales de código para cada subcarpeta (errors, constants, value-objects, entities, repositories, use-cases, services, workers, controllers, routes, swagger docs).
- [Patrones de Auditoría y Seguridad de Sesión](audit-and-security-patterns.md): Auditoría automática con `AsyncLocalStorage`, rotación de Refresh Tokens resiliente a concurrencia y Rate Limiting.
- [Stack de Dependencias Esenciales del Backend](dependencies-stack.md): Catálogo de librerías base y notas de compatibilidad de versiones (Prisma 6 vs 7, Zod 3 vs 4).
- [Reglas Estrictas de Dependencias por Capa](layer-rules.md): Matriz de qué imports están permitidos y cuáles prohibidos.
- [Plantilla de Casos de Uso y Controladores](use-case-controller-pattern.md): Plantillas de código para use cases, controllers y rutas.


