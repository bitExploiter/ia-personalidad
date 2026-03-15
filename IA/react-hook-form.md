# React Hook Form + Zod — Convenciones

Versiones mínimas: **React Hook Form 7.x**, **Zod 3.x**

Siempre se usan juntos. RHF maneja el estado del form y el submit.
Zod define el schema y valida. Nunca uno sin el otro.

---

## Schema primero — siempre

El tipo se deriva del schema. Nunca al revés.

```ts
// features/users/schemas.ts
import { z } from 'zod';

export const createUserSchema = z.object({
  name: z.string().min(2, 'Nombre muy corto').max(100),
  email: z.string().email('Email inválido'),
  age: z.coerce.number().int().min(18, 'Debes ser mayor de edad').optional(),
  role: z.enum(['admin', 'editor', 'viewer']),
});

// ✅ Tipo derivado del schema — nunca desincronizados
export type CreateUserInput = z.infer<typeof createUserSchema>;
```

---

## Formulario completo con shadcn/ui

```tsx
// features/users/components/CreateUserForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { createUserSchema, type CreateUserInput } from '../schemas';
import {
  Form, FormControl, FormField, FormItem, FormLabel, FormMessage,
} from '@/components/ui/form';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';

interface CreateUserFormProps {
  onSuccess: (data: CreateUserInput) => void;
}

export function CreateUserForm({ onSuccess }: CreateUserFormProps) {
  const form = useForm<CreateUserInput>({
    resolver: zodResolver(createUserSchema),
    defaultValues: { name: '', email: '', role: 'viewer' },
  });

  async function onSubmit(data: CreateUserInput) {
    // data está validado y tipado — safe to use
    await createUser(data);
    form.reset();
    onSuccess(data);
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Nombre</FormLabel>
              <FormControl>
                <Input placeholder="Ana García" {...field} />
              </FormControl>
              <FormMessage /> {/* muestra el error de Zod automáticamente */}
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input type="email" placeholder="ana@ejemplo.com" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? 'Guardando...' : 'Crear usuario'}
        </Button>
      </form>
    </Form>
  );
}
```

---

## Reglas de Zod

**`z.coerce`** para inputs HTML que llegan como string:
```ts
age: z.coerce.number().int().min(0),
date: z.coerce.date(),
```

**`.refine()`** para validaciones cruzadas entre campos:
```ts
z.object({ password: z.string(), confirm: z.string() })
 .refine((v) => v.password === v.confirm, {
   message: 'Las contraseñas no coinciden',
   path: ['confirm'], // a qué campo se asocia el error
 });
```

**Reusar schemas** con `.extend()`, `.partial()`, `.pick()`, `.omit()`:
```ts
// Crear a partir de otro schema, sin repetir campos
export const updateUserSchema = createUserSchema.partial().extend({
  id: z.string().uuid(),
});
export type UpdateUserInput = z.infer<typeof updateUserSchema>;
```

**Parsear en servicios** — validar datos externos antes de usarlos:
```ts
// En un service o Server Action
const parsed = createUserSchema.safeParse(rawData);
if (!parsed.success) {
  // parsed.error.flatten() da errores por campo
  throw new Error('Datos inválidos');
}
// parsed.data está tipado y limpio
```

---

## Testing de schemas

```ts
// features/users/schemas.test.ts
import { createUserSchema } from './schemas';

describe('createUserSchema', () => {
  it('acepta datos válidos', () => {
    const result = createUserSchema.safeParse({
      name: 'Ana', email: 'ana@test.com', role: 'admin',
    });
    expect(result.success).toBe(true);
  });

  it('rechaza email inválido', () => {
    const result = createUserSchema.safeParse({ name: 'Ana', email: 'no-email', role: 'admin' });
    expect(result.success).toBe(false);
    expect(result.error?.flatten().fieldErrors.email).toBeDefined();
  });
});
```
