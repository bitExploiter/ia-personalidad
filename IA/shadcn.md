# shadcn/ui — Convenciones

---

## Concepto clave

shadcn/ui **no es una dependencia npm** — es código que se copia al proyecto.
`npx shadcn@latest add button` agrega el componente en `src/components/ui/`.
Ese código te pertenece: puedes editarlo, pero hazlo con criterio.

---

## Instalación

```bash
# Inicializar en un proyecto nuevo o existente
npx shadcn@latest init

# Agregar componentes individualmente — solo lo que se usa
npx shadcn@latest add button input form card dialog select sheet table skeleton
```

---

## Reglas de uso

**No tocar `src/components/ui/`** directamente salvo casos excepcionales.
Si necesitas variaciones, crear un wrapper encima:

```tsx
// ✅ Bien: wrapper con lógica propia, primitivo intacto
// components/ui/ConfirmButton.tsx
import { Button } from '@/components/ui/button';

interface ConfirmButtonProps {
  onConfirm: () => void;
  label: string;
}
export function ConfirmButton({ onConfirm, label }: ConfirmButtonProps) {
  return <Button variant="destructive" onClick={onConfirm}>{label}</Button>;
}
```

**Componer, no copiar.** Construir componentes de dominio encima de primitivos:

```tsx
// features/users/components/UserCard.tsx
import { Card, CardHeader, CardContent } from '@/components/ui/card';

export function UserCard({ user }: { user: User }) {
  return (
    <Card>
      <CardHeader>{user.name}</CardHeader>
      <CardContent>{user.email}</CardContent>
    </Card>
  );
}
```

**`cn()` para clases condicionales** — siempre, nunca concatenación manual:

```tsx
import { cn } from '@/lib/utils'; // clsx + tailwind-merge

<div className={cn('base-class', isActive && 'ring-2 ring-primary', className)} />
```

**Temas con variables CSS** — configurar en `globals.css`, no hardcodear colores:

```css
/* globals.css */
:root {
  --primary: 222.2 47.4% 11.2%;
  --destructive: 0 84.2% 60.2%;
  /* ... resto de variables de shadcn */
}
```

**Forms de shadcn + React Hook Form siempre integrados:**
Usar `<Form>`, `<FormField>`, `<FormItem>`, `<FormLabel>`, `<FormControl>`,
`<FormMessage>` — están diseñados para trabajar con el contexto de RHF.
Ver [react-hook-form.md](react-hook-form.md) para el ejemplo completo.

---

## Componentes y cuándo usarlos

| Componente  | Cuándo usarlo                                                        |
|-------------|----------------------------------------------------------------------|
| `Button`    | Toda acción del usuario. Variantes: `default`, `outline`, `ghost`, `destructive` |
| `Input`     | Campos de texto. Siempre dentro de `<FormField>` con RHF             |
| `Select`    | Listas cortas (< 10 opciones). Para búsqueda, usar `Combobox`        |
| `Dialog`    | Confirmaciones, formularios modales, detalles en popup               |
| `Sheet`     | Paneles laterales: filtros, edición inline, menús en mobile          |
| `Toast`     | Feedback de acciones (éxito, error). Vía `useToast`                  |
| `Table`     | Datos tabulares. Sorting/filtering implementado en servidor o componente |
| `Skeleton`  | Loading states. Preferir sobre spinners para contenido con forma definida |
| `Badge`     | Etiquetas de estado, categorías, contadores                          |
| `Separator` | División visual entre secciones. No usar `<hr>` directo              |
