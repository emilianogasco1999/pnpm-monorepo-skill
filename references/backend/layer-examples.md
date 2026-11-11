# Ejemplos Canónicos por Capa (Backend Clean Architecture)

Este archivo contiene los ejemplos de código reales y funcionales para cada una de las capas de una feature en `apps/api`. La IA debe utilizar estos snippets como plantilla exacta (*Golden Examples*) al crear nuevo código.

---

## 1. Núcleo Transversal: Errores (`core/domain/errors/appErrors.ts`)

```typescript
export type AppError = Error & {
  statusCode: number;
};

export const createError = (
  message: string,
  statusCode: number,
  name = 'Error',
): AppError => {
  const error = new Error(message) as AppError;
  error.statusCode = statusCode;
  error.name = name;
  return error;
};

export const ClientError = (message = 'Error del cliente', statusCode = 400) =>
  createError(message, statusCode, 'ClientError');

export const NotFoundError = (message = 'Recurso no encontrado') =>
  createError(message, 404, 'NotFoundError');

export const UnauthorizedError = (message = 'No autorizado') =>
  createError(message, 401, 'UnauthorizedError');

export const ValidationError = (message = 'Datos inválidos') =>
  createError(message, 422, 'ValidationError');
```

---

## 2. Dominio: Constantes (`features/<feature>/domain/constants/<feature>.messages.ts`)

```typescript
export const userMessages = {
  createSuccess: {
    statusCode: 201,
    message: 'Usuario creado exitosamente',
  },
  getSuccess: {
    statusCode: 200,
    message: 'Usuarios obtenidos exitosamente',
  },
  getByIdSuccess: {
    statusCode: 200,
    message: 'Usuario obtenido exitosamente',
  },
  updateSuccess: {
    statusCode: 200,
    message: 'Usuario actualizado exitosamente',
  },
  deleteSuccess: {
    statusCode: 200,
    message: 'Usuario eliminado exitosamente',
  },
  errors: {
    notFound: {
      statusCode: 404,
      message: 'Usuario no encontrado',
    },
    alreadyExists: {
      statusCode: 409,
      message: 'El email ya está registrado',
    },
  },
} as const;
```

---

## 3. Dominio: Value Object (`features/<feature>/domain/value-objects/UserRole.ts`)

```typescript
export const USER_ROLES = {
  ADMINISTRADOR: 'ADMINISTRADOR',
  CONTADOR: 'CONTADOR',
} as const;

export type UserRole = (typeof USER_ROLES)[keyof typeof USER_ROLES];
```

---

## 4. Dominio: Entidad (`features/<feature>/domain/entities/User.ts`)

```typescript
import { UserRole } from '../value-objects/UserRole.js';

export interface User {
  id: string;
  personId: string | null;
  email: string;
  rol: UserRole;
  activo: boolean;
  ultimoLogin: Date | null;
  createdAt: Date;
}
```

---

## 5. Dominio: Interfaz de Repositorio (`features/<feature>/domain/repositories/user.repository.ts`)

```typescript
import { User } from '../entities/User.js';
import { CreateUserInput, UpdateUserInput, UserQueryParams, PaginatedResponse } from '@<scope>/shared';

export interface IUserRepository {
  findAll(filters?: UserQueryParams): Promise<PaginatedResponse<User>>;
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(data: CreateUserInput & { passwordHash: string }): Promise<User>;
  update(id: string, data: Partial<UpdateUserInput> & { passwordHash?: string }): Promise<User>;
  delete(id: string): Promise<User>;
}
```

---

## 6. Aplicación: Caso de Uso (`features/<feature>/application/useCases/createUserUseCase.ts`)

```typescript
import bcrypt from 'bcrypt';
import { IUserRepository } from '../../domain/repositories/user.repository.js';
import { CreateUserInput } from '@<scope>/shared';
import { ClientError } from '@core/domain/errors/appErrors.js';
import { User } from '../../domain/entities/User.js';

export const createUserUseCase = async (
  userRepository: IUserRepository,
  data: CreateUserInput,
): Promise<User> => {
  const existing = await userRepository.findByEmail(data.email);
  if (existing) {
    throw ClientError(`El email ${data.email} ya está registrado para un usuario`, 409);
  }

  const passwordHash = await bcrypt.hash(data.password, 10);

  return userRepository.create({
    ...data,
    passwordHash,
  });
};
```

---

## 7. Infraestructura: Implementación de Repositorio (`features/<feature>/infrastructure/repositories/user.repository.impl.ts`)

```typescript
import { prisma } from '@core/infrastructure/database/prisma.service.js';
import { User } from '../../domain/entities/User.js';
import { UserRole } from '../../domain/value-objects/UserRole.js';
import { IUserRepository } from '../../domain/repositories/user.repository.js';
import { CreateUserInput, UpdateUserInput, UserQueryParams, PaginatedResponse } from '@<scope>/shared';

export class UserRepositoryImpl implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    const raw = await prisma.user.findUnique({
      where: { id },
      include: { person: true },
    });
    if (!raw) return null;

    return {
      id: raw.id,
      personId: raw.personId,
      email: raw.email,
      rol: raw.rol as UserRole,
      activo: raw.activo,
      ultimoLogin: raw.ultimoLogin,
      createdAt: raw.createdAt,
    };
  }

  async findByEmail(email: string): Promise<User | null> {
    const raw = await prisma.user.findUnique({ where: { email } });
    if (!raw) return null;

    return {
      id: raw.id,
      personId: raw.personId,
      email: raw.email,
      rol: raw.rol as UserRole,
      activo: raw.activo,
      ultimoLogin: raw.ultimoLogin,
      createdAt: raw.createdAt,
    };
  }

  async create(data: CreateUserInput & { passwordHash: string }): Promise<User> {
    const raw = await prisma.user.create({
      data: {
        email: data.email,
        password: data.passwordHash,
        rol: data.rol as any,
        personId: data.personId,
      },
    });

    return {
      id: raw.id,
      personId: raw.personId,
      email: raw.email,
      rol: raw.rol as UserRole,
      activo: raw.activo,
      ultimoLogin: raw.ultimoLogin,
      createdAt: raw.createdAt,
    };
  }

  async update(id: string, data: Partial<UpdateUserInput> & { passwordHash?: string }): Promise<User> {
    const raw = await prisma.user.update({
      where: { id },
      data: {
        ...(data.rol && { rol: data.rol as any }),
        ...(data.activo !== undefined && { activo: data.activo }),
        ...(data.passwordHash && { password: data.passwordHash }),
      },
    });

    return {
      id: raw.id,
      personId: raw.personId,
      email: raw.email,
      rol: raw.rol as UserRole,
      activo: raw.activo,
      ultimoLogin: raw.ultimoLogin,
      createdAt: raw.createdAt,
    };
  }

  async delete(id: string): Promise<User> {
    const raw = await prisma.user.delete({ where: { id } });
    return {
      id: raw.id,
      personId: raw.personId,
      email: raw.email,
      rol: raw.rol as UserRole,
      activo: raw.activo,
      ultimoLogin: raw.ultimoLogin,
      createdAt: raw.createdAt,
    };
  }

  async findAll(filters?: UserQueryParams): Promise<PaginatedResponse<User>> {
    // Paginación y filtros con prisma.user.count y findMany...
    const page = Number(filters?.page) || 1;
    const limit = Number(filters?.limit) || 10;
    const skip = (page - 1) * limit;

    const [total, users] = await Promise.all([
      prisma.user.count(),
      prisma.user.findMany({ skip, take: limit, orderBy: { createdAt: 'desc' } }),
    ]);

    return {
      data: users.map((u) => ({
        id: u.id,
        personId: u.personId,
        email: u.email,
        rol: u.rol as UserRole,
        activo: u.activo,
        ultimoLogin: u.ultimoLogin,
        createdAt: u.createdAt,
      })),
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    };
  }
}
```

---

## 8. Infraestructura: Background Worker (`features/<feature>/infrastructure/workers/arcaBackgroundWorker.ts`)

```typescript
export class BackgroundWorker {
  private isProcessing = false;
  private timer: NodeJS.Timeout | null = null;

  start(intervalMs = 5000): void {
    console.log('[Worker] Iniciado procesador en segundo plano.');
    this.timer = setInterval(() => {
      this.processQueue().catch((err) => {
        console.error('[Worker] Error procesando cola:', err);
      });
    }, intervalMs);
  }

  stop(): void {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
    }
  }

  private async processQueue(): Promise<void> {
    if (this.isProcessing) return;
    this.isProcessing = true;

    try {
      // Buscar tareas pendientes en la base de datos y ejecutarlas
    } finally {
      this.isProcessing = false;
    }
  }
}
```

---

## 9. Presentación: Controlador (`features/<feature>/presentation/controllers/user.controller.ts`)

```typescript
import { Request, Response } from 'express';
import { UserRepositoryImpl } from '../../infrastructure/repositories/user.repository.impl.js';
import { getUsersUseCase } from '../../application/useCases/getUsersUseCase.js';
import { getUserByIdUseCase } from '../../application/useCases/getUserByIdUseCase.js';
import { createUserUseCase } from '../../application/useCases/createUserUseCase.js';
import { updateUserUseCase } from '../../application/useCases/updateUserUseCase.js';
import { deleteUserUseCase } from '../../application/useCases/deleteUserUseCase.js';
import { response } from '@core/presentation/utils/response.js';
import { userMessages } from '../../domain/constants/user.messages.js';

export class UserController {
  private readonly repository = new UserRepositoryImpl();

  getUsers = async (req: Request, res: Response): Promise<void> => {
    const result = await getUsersUseCase(this.repository, req.query as any);
    response(res, userMessages.getSuccess.statusCode, userMessages.getSuccess.message, result);
  };

  getUserById = async (req: Request, res: Response): Promise<void> => {
    const id = req.params.id as string;
    const user = await getUserByIdUseCase(this.repository, id);
    response(res, userMessages.getByIdSuccess.statusCode, userMessages.getByIdSuccess.message, user);
  };

  createUser = async (req: Request, res: Response): Promise<void> => {
    const user = await createUserUseCase(this.repository, req.body);
    response(res, userMessages.createSuccess.statusCode, userMessages.createSuccess.message, user);
  };

  updateUser = async (req: Request, res: Response): Promise<void> => {
    const id = req.params.id as string;
    const user = await updateUserUseCase(this.repository, id, req.body);
    response(res, userMessages.updateSuccess.statusCode, userMessages.updateSuccess.message, user);
  };

  deleteUser = async (req: Request, res: Response): Promise<void> => {
    const id = req.params.id as string;
    const user = await deleteUserUseCase(this.repository, id);
    response(res, userMessages.deleteSuccess.statusCode, userMessages.deleteSuccess.message, user);
  };
}
```

---

## 10. Presentación: Rutas Express (`features/<feature>/presentation/routes/user.routes.ts`)

```typescript
import { Router } from 'express';
import { createUserSchema, updateUserSchema } from '@<scope>/shared';
import { UserController } from '../controllers/user.controller.js';
import { catchedAsync } from '@core/presentation/utils/catchedAsync.js';
import { authMiddleware } from '@core/presentation/middlewares/auth.middleware.js';
import { requireRole } from '@core/presentation/middlewares/permission.middleware.js';
import { validateRequest } from '@core/presentation/middlewares/validateRequest.middleware.js';

export const UserRoutes = (): Router => {
  const router = Router();
  const controller = new UserController();

  // 1. Barrera de sesión: Todo endpoint requiere JWT
  router.use(authMiddleware);

  // 2. Control de roles
  router.use(requireRole(['ADMINISTRADOR']));

  router.get('/', catchedAsync(controller.getUsers));
  router.get('/:id', catchedAsync(controller.getUserById));
  router.post('/', validateRequest(createUserSchema), catchedAsync(controller.createUser));
  router.put('/:id', validateRequest(updateUserSchema), catchedAsync(controller.updateUser));
  router.delete('/:id', catchedAsync(controller.deleteUser));

  return router;
};
```

---

## 11. Presentación: Documentación Swagger (`features/<feature>/presentation/docs/user.docs.ts`)

```typescript
/**
 * @swagger
 * /api/usuarios:
 *   get:
 *     summary: Obtiene la lista de usuarios del sistema
 *     tags: [Usuarios]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *         description: Número de página
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 10
 *         description: Registros por página
 *     responses:
 *       200:
 *         description: Lista de usuarios obtenida exitosamente
 *       401:
 *         description: No autenticado
 *       403:
 *         description: Requiere rol ADMINISTRADOR
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
 *               password:
 *                 type: string
 *               rol:
 *                 type: string
 *                 enum: [ADMINISTRADOR, CONTADOR]
 *     responses:
 *       201:
 *         description: Usuario creado exitosamente
 *       400:
 *         description: Error de validación Zod
 *       409:
 *         description: Email ya registrado
 */
```
