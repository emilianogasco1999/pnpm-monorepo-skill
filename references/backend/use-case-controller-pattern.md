# Plantilla Canónica: Casos de Uso, Controladores y Rutas

Esta referencia provee la estructura de código estándar para implementar una nueva funcionalidad respetando Clean Architecture.

> [!NOTE]
> **Dos Estilos Válidos de Implementación (Clases vs Funciones)**:
> Mientras se respete estrictamente la inversión de dependencias (el Caso de Uso recibe la interfaz `IRepository` y nunca importa Prisma directamente), el proyecto admite dos variantes:
> 1. **Estilo Funcional (`application/useCases/createEntityUseCase.ts`)**: Preferido para la mayoría de los módulos de negocio CRUD por su concisión; recibe el repositorio como primer argumento `createEntityUseCase(repository, dto)` (ver ejemplos en `layer-examples.md`).
> 2. **Estilo Orientado a Objetos (`application/use-cases/create-entity.use-case.ts`)**: Preferido en módulos con estado complejo o múltiples dependencias inyectadas en el constructor (ej: `auth` con rotación de tokens y criptografía), como se muestra a continuación.

---

## 1. El Caso de Uso (`application/use-cases/<name>.use-case.ts`)

Cada caso de uso tiene **una única responsabilidad**, recibe sus dependencias en el constructor y expone un método `execute`:

```typescript
import { IEntityRepository } from '../../domain/repositories/IEntityRepository.js';
import { CreateEntityInput } from '@<scope>/shared';
import { ClientError } from '../../../../core/domain/errors/appErrors.js';
import { Entity } from '../../domain/entities/Entity.js';

export class CreateEntityUseCase {
  constructor(private readonly repository: IEntityRepository) {}

  async execute(dto: CreateEntityInput, actorId: string): Promise<Entity> {
    // 1. Validaciones de regla de negocio
    const existing = await this.repository.findByUniqueKey(dto.uniqueKey);
    if (existing) {
      throw ClientError('Ya existe una entidad registrada con esa clave.', 409);
    }

    // 2. Creación y persistencia
    const entity = await this.repository.create({
      ...dto,
      createdBy: actorId,
    });

    return entity;
  }
}
```

---

## 2. El Controlador (`presentation/controllers/<name>.controller.ts`)

El controlador extrae parámetros del transporte HTTP, invoca el caso de uso y formatea la respuesta con la utilidad estándar `response`:

```typescript
import { Request, Response } from 'express';
import { CreateEntityUseCase } from '../../application/use-cases/create-entity.use-case.js';
import { EntityRepositoryImpl } from '../../infrastructure/repositories/entity.repository.impl.js';
import { response } from '../../../../core/presentation/utils/response.js';

export class EntityController {
  private readonly createEntityUseCase: CreateEntityUseCase;

  constructor() {
    const repository = new EntityRepositoryImpl();
    this.createEntityUseCase = new CreateEntityUseCase(repository);
  }

  create = async (req: Request, res: Response): Promise<void> => {
    const actorId = req.user!.id; // Inyectado previamente por authMiddleware
    const result = await this.createEntityUseCase.execute(req.body, actorId);

    response(
      res,
      201,
      'Entidad creada exitosamente.',
      result,
    );
  };
}
```

---

## 3. La Ruta Express (`presentation/routes/<name>.routes.ts`)

Aplica la cadena de seguridad en orden estricto antes de llegar al controlador:

```typescript
import { Router } from 'express';
import { EntityController } from '../controllers/entity.controller.js';
import { authMiddleware } from '../../../../core/presentation/middlewares/auth.middleware.js';
import { requireRole } from '../../../../core/presentation/middlewares/permission.middleware.js';
import { validateRequest } from '../../../../core/presentation/middlewares/validateRequest.middleware.js';
import { catchedAsync } from '../../../../core/presentation/utils/catchedAsync.js';
import { createEntitySchema } from '@<scope>/shared';

export const EntityRoutes = (): Router => {
  const router = Router();
  const controller = new EntityController();

  // 1. Barrera de sesión: Todo endpoint requiere JWT verificado
  router.use(authMiddleware);

  // 2. Endpoint protegido con control de roles y validación de contrato Zod
  router.post(
    '/',
    requireRole(['ADMINISTRADOR', 'CONTADOR']),
    validateRequest(createEntitySchema),
    catchedAsync(controller.create.bind(controller)),
  );

  return router;
};
```
