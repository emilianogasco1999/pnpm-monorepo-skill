# Ejemplos Canónicos por Capa en Frontend (Clean Architecture)

Esta referencia técnica provee código modelo y reproducible para cada capa dentro de `apps/web` (Next.js 16 + React 19), tomando como ejemplo la feature de autenticación y gestión de usuarios.

---

## 1. Capa de Dominio (`domain/`)

### A. Entidad de Dominio (`domain/entities/user.entity.ts`)
```typescript
import { Rol } from '@<scope>/shared';

export interface UserProps {
  id: string;
  email: string;
  nombre?: string | null;
  apellido?: string | null;
  rol: Rol;
  activo: boolean;
}

export class User {
  constructor(private readonly props: UserProps) {}

  get id(): string {
    return this.props.id;
  }

  get email(): string {
    return this.props.email;
  }

  get rol(): Rol {
    return this.props.rol;
  }

  get nombreCompleto(): string {
    if (!this.props.nombre && !this.props.apellido) return this.props.email;
    return `${this.props.nombre ?? ''} ${this.props.apellido ?? ''}`.trim();
  }

  get esAdmin(): boolean {
    return this.props.rol === 'ADMINISTRADOR';
  }

  toJSON(): UserProps {
    return { ...this.props };
  }
}
```

### B. Interfaz de Repositorio (`domain/repositories/auth.repository.interface.ts`)
```typescript
import { LoginInput } from '@<scope>/shared';
import { User } from '../entities/user.entity.js';

export interface AuthSession {
  user: User;
  accessToken: string;
}

export interface IAuthRepository {
  login(credentials: LoginInput): Promise<AuthSession>;
  logout(): Promise<void>;
  getMe(): Promise<User>;
}
```

---

## 2. Capa de Aplicación (`application/`)

### Caso de Uso (`application/use-cases/login.use-case.ts`)
```typescript
import { LoginInput } from '@<scope>/shared';
import { IAuthRepository, AuthSession } from '../../domain/repositories/auth.repository.interface.js';

export class LoginUseCase {
  constructor(private readonly authRepository: IAuthRepository) {}

  async execute(credentials: LoginInput): Promise<AuthSession> {
    // 1. Sanitización o reglas previas de negocio
    const normalizedInput: LoginInput = {
      ...credentials,
      email: credentials.email.trim().toLowerCase(),
    };

    // 2. Delegación a la infraestructura mediante el contrato del repositorio
    const session = await this.authRepository.login(normalizedInput);

    return session;
  }
}
```

---

## 3. Capa de Infraestructura (`infrastructure/`)

### Implementación del Repositorio (`infrastructure/repositories/auth.repository.impl.ts`)
```typescript
import { LoginInput } from '@<scope>/shared';
import { apiClient } from '@/core/api/api.client';
import { IAuthRepository, AuthSession } from '../../domain/repositories/auth.repository.interface';
import { User } from '../../domain/entities/user.entity';

interface LoginApiResponse {
  user: {
    id: string;
    email: string;
    nombre?: string | null;
    apellido?: string | null;
    rol: any;
    activo: boolean;
  };
  accessToken: string;
}

export class AuthRepositoryImpl implements IAuthRepository {
  async login(credentials: LoginInput): Promise<AuthSession> {
    const { data } = await apiClient.post<LoginApiResponse>('/api/auth/login', credentials);

    return {
      user: new User(data.user),
      accessToken: data.accessToken,
    };
  }

  async logout(): Promise<void> {
    await apiClient.post('/api/auth/logout');
  }

  async getMe(): Promise<User> {
    const { data } = await apiClient.get<{ user: any }>('/api/auth/me');
    return new User(data.user);
  }
}
```

---

## 4. Capa de Presentación (`presentation/`)

### A. Validación de Formularios (`presentation/validation/login.validation.ts`)
```typescript
import { loginSchema, type LoginInput } from '@<scope>/shared';

// Reutilización directa del esquema Zod del paquete compartido
export { loginSchema };
export type LoginFormValues = LoginInput;
```

### B. Mutación con TanStack Query (`presentation/mutations/useLoginMutation.ts`)
```typescript
'use client';

import { useMutation } from '@tanstack/react-query';
import { useRouter } from 'next/navigation';
import { toast } from 'sonner';
import { LoginInput } from '@<scope>/shared';
import { LoginUseCase } from '../../application/use-cases/login.use-case';
import { AuthRepositoryImpl } from '../../infrastructure/repositories/auth.repository.impl';
import { useAuthStore } from '@/core/store/auth.store';

export function useLoginMutation() {
  const router = useRouter();
  const setSession = useAuthStore((state) => state.setSession);

  const authRepository = new AuthRepositoryImpl();
  const loginUseCase = new LoginUseCase(authRepository);

  return useMutation({
    mutationFn: (credentials: LoginInput) => loginUseCase.execute(credentials),
    onSuccess: (session) => {
      // Guarda usuario y token en store liviano
      setSession(session.user, session.accessToken);
      toast.success(`¡Bienvenido de vuelta, ${session.user.nombreCompleto}!`);
      router.push('/dashboard');
    },
    onError: (error: any) => {
      const message = error?.response?.data?.message ?? 'Error al iniciar sesión. Verifique sus credenciales.';
      toast.error(message);
    },
  });
}
```

### C. Componente de Formulario (`presentation/components/LoginForm.tsx`)

> [!IMPORTANT]
> **Regla de Reutilización de Componentes Base**: Si el proyecto ya cuenta con componentes de formulario encapsulados en `@/components/Form/` (`InputField`, `SelectField`, `CheckboxField`, `SearchableSelectField`, etc.) y componentes de UI en `@/components/ui/` (`Button`, `FieldGroup`, `Card`), **es obligatorio reutilizarlos**, evitando crear inputs o selects HTML nativos ad-hoc con validaciones manuales.

```tsx
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { loginSchema, LoginFormValues } from '../validation/login.validation';
import { useLoginMutation } from '../mutations/useLoginMutation';
import { InputField } from '@/components/Form/InputField';
import { Button } from '@/components/ui/button';
import { FieldGroup } from '@/components/ui/field';

export function LoginForm() {
  const { mutate: login, isPending } = useLoginMutation();

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginFormValues>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
    },
  });

  const onSubmit = (values: LoginFormValues) => {
    login(values);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4 w-full max-w-sm">
      <FieldGroup>
        <InputField
          label="Email"
          type="email"
          placeholder="usuario@empresa.com"
          disabled={isPending}
          error={errors.email?.message}
          {...register('email')}
        />

        <InputField
          label="Contraseña"
          type="password"
          placeholder="••••••••"
          disabled={isPending}
          error={errors.password?.message}
          {...register('password')}
        />

        <Button
          type="submit"
          disabled={isPending}
          className="w-full mt-2"
        >
          {isPending ? 'Ingresando...' : 'Iniciar Sesión'}
        </Button>
      </FieldGroup>
    </form>
  );
}
```

### D. Vista Completa (`presentation/pages/LoginPage.tsx`)
```tsx
import { LoginForm } from '../components/LoginForm';

export function LoginPage() {
  return (
    <div className="min-h-screen flex items-center justify-center bg-muted/30 px-4">
      <div className="w-full max-w-md p-8 bg-card border rounded-xl shadow-lg space-y-6">
        <div className="text-center space-y-2">
          <h1 className="text-2xl font-bold tracking-tight">Iniciar Sesión</h1>
          <p className="text-sm text-muted-foreground">Ingresá tus credenciales para acceder al sistema</p>
        </div>
        <LoginForm />
      </div>
    </div>
  );
}
```

---

## 5. Capa de Enrutamiento (`src/app/`)

### Shell Delgado de Next.js App Router (`src/app/login/page.tsx`)
```tsx
import { LoginPage } from '@/features/auth/presentation/pages/LoginPage';
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Iniciar Sesión | App',
  description: 'Acceso seguro al sistema',
};

export default function Page() {
  return <LoginPage />;
}
```
