# Patrón Canónico: Casos de Uso, Hooks (Controladores de UI) y Componentes

Esta referencia técnica explica la interacción entre Casos de Uso, Custom Hooks de TanStack Query y Componentes de React bajo Clean Architecture en el frontend (`apps/web`).

---

## 1. El Flujo de Datos Unidireccional en el Cliente

En el backend, el flujo es `Route -> Controller -> UseCase -> Repository`.
En el frontend de Next.js / React, el equivalente arquitectónico es:

```
[ Usuario interactúa con la UI ]
                 ↓
[ Componente Presentacional (Formulario / Botón) ]
                 ↓
[ Custom Hook de Mutación / Query (Controlador de UI) ]
                 ↓
[ Caso de Uso (Application / execute()) ] ───(Reglas de Negocio)───► [ Entidad de Dominio ]
                 ↓
[ Contrato IRepository (Domain) ]
                 ▲
                 │ (Implementado por)
[ Repositorio Concreto (Infrastructure) ]
                 ↓
[ Cliente HTTP Axios (core/api) ]
                 ↓
[ API Backend ]
```

---

## 2. El Caso de Uso en Frontend (`application/use-cases/<name>.use-case.ts`)

El caso de uso es **TypeScript puro**. No conoce React, ni JSX, ni hooks (`useState`, `useQuery`).
- **Responsabilidad Única**: Orquestar una acción de negocio del usuario.
- **Inyección de Dependencias**: Recibe las interfaces de repositorio en su constructor.
- **Testeabilidad**: Se puede testear con tests unitarios instantáneos sin montar navegadores ni React DOM.

> [!TIP]
> **Regla Pragmática (Cuándo crear `application/use-cases/` vs Llamada Directa al Repositorio)**:
> - **Con Caso de Uso (`application/use-cases/`)**: Obligatorio cuando existe lógica de negocio en el cliente, sanitización compleja, coordinación de múltiples repositorios o efectos de sesión (ej: `auth`, `usuarios`, `personas`).
> - **Simplificación Pragmática en CRUDs Puros**: Si una feature solo realiza lectura/escritura pasamanos hacia la API (donde toda la regla de negocio ya vive en `apps/api`), el hook de TanStack Query en `presentation/queries/` o `presentation/mutations/` puede invocar directamente el método de `repositoryImpl` (`infrastructure/repositories/`), evitando crear clases `UseCase` vacías que solo hagan `return this.repo.getAll()`. NUNCA se permite, sin embargo, llamar a `apiClient` (Axios) directamente desde un componente o hook salteando el repositorio.

```typescript
import { IClientRepository } from '../../domain/repositories/client.repository.interface';
import { CreateClientInput } from '@<scope>/shared';
import { Client } from '../../domain/entities/client.entity';

export class CreateClientUseCase {
  constructor(private readonly clientRepository: IClientRepository) {}

  async execute(dto: CreateClientInput): Promise<Client> {
    // 1. Validaciones o normalizaciones de reglas de cliente
    if (!dto.razonSocial.trim()) {
      throw new Error('La razón social no puede estar vacía.');
    }

    // 2. Persistencia delegada a la infraestructura
    const client = await this.clientRepository.create(dto);
    return client;
  }
}
```

---

## 3. El Hook de Mutación / Query como Controlador de UI (`presentation/mutations/` o `queries/`)

El hook actúa como el **Controlador / Adaptador de Entrega**:
- Extrae parámetros de la UI.
- Instancia el repositorio concreto y el caso de uso.
- Administra el ciclo de vida asíncrono (`isPending`, `isError`, `isSuccess`).
- Coordina los efectos colaterales de UI (feedback visual con `sonner`, redirección con `next/navigation`, invalidación de caché en TanStack Query y actualización de stores globales).

```typescript
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';
import { toast } from 'sonner';
import { CreateClientInput } from '@<scope>/shared';
import { CreateClientUseCase } from '../../application/use-cases/create-client.use-case';
import { ClientRepositoryImpl } from '../../infrastructure/repositories/client.repository.impl';

export function useCreateClientMutation(onSuccessCallback?: () => void) {
  const queryClient = useQueryClient();

  const repository = new ClientRepositoryImpl();
  const useCase = new CreateClientUseCase(repository);

  return useMutation({
    mutationFn: (input: CreateClientInput) => useCase.execute(input),
    onSuccess: (newClient) => {
      // 1. Invalida cache para refrescar listas automáticamente
      queryClient.invalidateQueries({ queryKey: ['clients'] });
      // 2. Notificación al usuario
      toast.success(`Cliente "${newClient.razonSocial}" creado con éxito.`);
      // 3. Callback opcional (cerrar modal, resetear formulario)
      onSuccessCallback?.();
    },
    onError: (error: any) => {
      const message = error?.response?.data?.message ?? 'Error al registrar el cliente.';
      toast.error(message);
    },
  });
}
```

---

## 4. Patrón Container / Presentational en Componentes

Para mantener el código visual limpio y testeable, separamos responsabilidades en dos tipos de componentes:

### A. Componentes Presentacionales (Dumb / UI Pura)
- No conocen mutaciones, ni casos de uso, ni llamadas a red.
- Reciben datos vía `props` y emiten eventos (`onSubmit`, `onClick`).
- Enfocados en accesibilidad, diseño visual y validación local de campos con `react-hook-form`.

### B. Componentes Container / Páginas (Smart / Orquestadores)
- Ubicados en `presentation/pages/` o componentes wrappers.
- Invocan los custom hooks (`useCreateClientMutation`, `useClientsQuery`).
- Manejan estados de carga (skeletons, spinners), modales y errores globales.
- Pasan los datos y callbacks a los componentes presentacionales.

---

## 5. Resumen de Beneficios de esta Separación

1. **Desacoplamiento Tecnológico**: Si mañana se migra de Axios a otra solución o cambia la API, solo se modifica `infrastructure/repositories/`. La lógica de la UI y los casos de uso quedan intactos.
2. **Reutilización**: Un mismo caso de uso puede ejecutarse desde una página web, desde un worker en segundo plano o desde un script sin cambiar una sola línea de código.
3. **Consistencia de Validación**: La UI valida con Zod (`@<scope>/shared`) antes de disparar la petición HTTP, garantizando feedback instantáneo al usuario sin cargas innecesarias de red.
