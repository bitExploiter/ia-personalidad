# React 19 — Convenciones

---

## Estructura de proyecto

```
src/
  components/          → componentes reutilizables (UI pura)
    ui/                → primitivos generados por shadcn/ui (no editar a mano)
    layout/            → estructura de página (Header, Sidebar, etc.)
  features/            → módulos por dominio de negocio
    users/
      components/      → componentes específicos del feature
      hooks/           → hooks del feature
      services/        → llamadas API del feature
      store.ts         → slice de Zustand del feature (si aplica)
      schemas.ts       → schemas Zod del feature
      types.ts         → tipos derivados de Zod + tipos propios
      index.ts         → barrel export
  hooks/               → hooks globales reutilizables
  lib/                 → utilidades, helpers, configuración de clientes
  store/               → stores Zustand globales
  types/               → tipos globales y DTOs compartidos
  pages/ o routes/     → componentes de página (según router)
```

---

## Reglas base

- **Functional components siempre.** No class components.
- **No default exports** (salvo pages si el router lo requiere). Named exports
  + barrel files (`index.ts`) para imports limpios.
- **Props tipadas explícitamente**:
  ```tsx
  interface UserCardProps {
    user: User;
    onSelect: (id: string) => void;
  }
  export function UserCard({ user, onSelect }: UserCardProps) { ... }
  ```
- **Estado local primero**: `useState`/`useReducer` → Context (prop drilling
  3+ niveles) → Zustand (estado global de cliente).
- **Fetching**: servicios propios en `features/<dominio>/services/` con `fetch`
  nativo o cliente HTTP. Llamados desde hooks o Server Actions.
- **Estilos**: Tailwind CSS + `cn()`. Ver [tailwind.md](tailwind.md).

---

## Hooks nuevos en React 19

**`use()` — leer Promises y Context directamente:**
```tsx
import { use } from 'react';

// Context (funciona dentro de condicionales, a diferencia de useContext)
const theme = use(ThemeContext);

// Promise — suspende automáticamente hasta que resuelve
function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}
```

**`useActionState` — acciones con estado de loading integrado:**
```tsx
import { useActionState } from 'react';

async function submitAction(prevState: State, formData: FormData) {
  const result = await saveUser(formData);
  return result;
}

function UserForm() {
  const [state, dispatch, isPending] = useActionState(submitAction, null);
  return (
    <form action={dispatch}>
      <input name="name" />
      <button disabled={isPending}>Guardar</button>
    </form>
  );
}
```

**`useOptimistic` — actualizaciones optimistas:**
```tsx
import { useOptimistic } from 'react';

function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state, newTodo: Todo) => [...state, newTodo],
  );

  async function handleAdd(formData: FormData) {
    const newTodo = { id: crypto.randomUUID(), text: formData.get('text') as string };
    addOptimistic(newTodo);          // muestra inmediatamente en UI
    await saveTodoToServer(newTodo); // persiste en background
  }

  return <form action={handleAdd}>...</form>;
}
```

**`useFormStatus` — estado del form accesible desde componentes hijos:**
```tsx
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Guardando...' : 'Guardar'}</button>;
}
```

**`ref` como prop — `forwardRef` ya no es necesario:**
```tsx
// React 19: ref es una prop normal
function Input({ ref, ...props }: React.ComponentProps<'input'>) {
  return <input ref={ref} {...props} />;
}
<Input ref={myRef} />
```

**Server Actions:**
```tsx
// actions/user.ts
'use server';

export async function createUser(formData: FormData) {
  const parsed = userSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { error: parsed.error.flatten() };
  await db.user.create({ data: parsed.data });
  revalidatePath('/users');
}
```

---

## Cuándo usar qué patrón de formulario

| Situación                                          | Usar                          |
|----------------------------------------------------|-------------------------------|
| Form simple, Server Action, sin validación compleja | `useActionState`              |
| Form complejo, validación en tiempo real, UX rica  | React Hook Form + Zod         |

---

## Testing

- **Vitest** como runner (compatible con Vite).
- **React Testing Library** — testear comportamiento, no implementación.
- **MSW (Mock Service Worker)** para mockear llamadas HTTP en tests.
- Naming: `NombreComponente.test.tsx`, `useHook.test.ts`.
