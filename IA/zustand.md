# Zustand — Convenciones

Versión mínima: **5.x**

---

## Reglas de uso

- **Un store por dominio.** No un store global con todo mezclado.
- El store solo guarda **estado del cliente** (UI state, preferencias, carrito,
  sesión). El estado del servidor se maneja con `fetch` / servicios propios.
- Las **acciones van dentro del store**, no en los componentes.
- Usar **slices** cuando el store crece. No un objeto gigante.
- **Persistir** con `zustand/middleware` (`persist`) solo lo que debe sobrevivir
  al refresco de página.

---

## Estructura de un store

```ts
// features/cart/store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { CartItem } from './types';

interface CartState {
  // estado
  items: CartItem[];
  // acciones
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
}

export const useCartStore = create<CartState>()(
  persist(
    (set) => ({
      items: [],

      addItem: (item) =>
        set((state) => {
          const exists = state.items.find((i) => i.id === item.id);
          if (exists) {
            return {
              items: state.items.map((i) =>
                i.id === item.id ? { ...i, qty: i.qty + 1 } : i,
              ),
            };
          }
          return { items: [...state.items, item] };
        }),

      removeItem: (id) =>
        set((state) => ({ items: state.items.filter((i) => i.id !== id) })),

      clearCart: () => set({ items: [] }),
    }),
    { name: 'cart-storage' }, // clave en localStorage
  ),
);
```

---

## Consumo en componentes

```tsx
// Seleccionar solo lo que el componente necesita — evita re-renders innecesarios
const items = useCartStore((state) => state.items);
const addItem = useCartStore((state) => state.addItem);

// Selector múltiple con objetos/arrays: usar useShallow
import { useShallow } from 'zustand/react/shallow';

const { items, addItem } = useCartStore(
  useShallow((state) => ({ items: state.items, addItem: state.addItem })),
);
```

---

## Slices — cuando el store crece

```ts
// store/slices/uiSlice.ts
import type { StateCreator } from 'zustand';

export interface UISlice {
  sidebarOpen: boolean;
  toggleSidebar: () => void;
}

export const createUISlice: StateCreator<UISlice> = (set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
});

// store/index.ts — combinar slices
import { create } from 'zustand';
import { createUISlice, type UISlice } from './slices/uiSlice';
import { createUserSlice, type UserSlice } from './slices/userSlice';

export const useAppStore = create<UISlice & UserSlice>()((...args) => ({
  ...createUISlice(...args),
  ...createUserSlice(...args),
}));
```

---

## Anti-patrones

```ts
// ❌ Mal: manipular estado fuera del store
function handleAdd(item: CartItem) {
  const state = useCartStore.getState();
  // lógica aquí en el componente...
}

// ✅ Bien: la acción encapsula la lógica, el componente solo la invoca
const addItem = useCartStore((state) => state.addItem);
addItem(newItem);

// ❌ Mal: suscribirse al store completo
const everything = useCartStore();

// ✅ Bien: selector específico
const items = useCartStore((state) => state.items);
```
