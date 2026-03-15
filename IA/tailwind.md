# Tailwind CSS — Convenciones

---

## Configuración base

- **Tailwind v4** como versión preferida si el proyecto es nuevo. En
  proyectos existentes con v3, respetar la versión instalada.
- **Config en `tailwind.config.ts`** (TypeScript, no JS). Definir el design
  system del proyecto aquí: colores, fuentes, breakpoints, spacing custom.
- **`@layer`** para organizar estilos custom: `base` para resets y
  tipografía global, `components` para clases reutilizables, `utilities`
  para utilidades propias.

---

## Convenciones de uso

- **Utility-first estricto.** Escribir CSS custom solo cuando Tailwind no
  tiene la utilidad o cuando un patrón se repite 3+ veces. En ese caso,
  extraer a `@apply` en un `@layer components`.
- **Orden de clases consistente**: layout → sizing → spacing → typography →
  colors → effects → responsive → states. Usar el plugin
  `prettier-plugin-tailwindcss` para automatizar el orden.
- **Responsive con mobile-first.** Empezar con el estilo base (mobile) y
  agregar breakpoints hacia arriba: `text-sm md:text-base lg:text-lg`.
  Nunca diseñar desktop-first.
- **Colores semánticos sobre literales.** En `tailwind.config.ts` definir
  tokens semánticos (`primary`, `surface`, `muted`, `destructive`) que
  mapeen a colores concretos. En el markup usar `bg-primary` en vez de
  `bg-blue-600`.
  ```ts
  // tailwind.config.ts
  colors: {
    primary: { DEFAULT: '#2563eb', dark: '#1d4ed8', light: '#60a5fa' },
    surface: { DEFAULT: '#ffffff', muted: '#f8fafc' },
    destructive: { DEFAULT: '#dc2626', dark: '#b91c1c' },
  }
  ```
- **Dark mode con `class` strategy**, no `media`. Esto permite toggle manual
  y respeta preferencia del sistema como fallback.
  ```ts
  // tailwind.config.ts
  darkMode: 'class',
  ```
- **Nunca `!important`** (`!` prefix en Tailwind). Si necesitas override, el
  problema es de especificidad — reestructurar el componente.
- **`cn()` helper para clases condicionales.** Usar `clsx` + `tailwind-merge`
  empaquetados en una función utilitaria:
  ```ts
  // lib/utils.ts
  import { clsx, type ClassValue } from "clsx";
  import { twMerge } from "tailwind-merge";
  export function cn(...inputs: ClassValue[]) {
    return twMerge(clsx(inputs));
  }
  ```
- **Max width en contenedores.** Siempre usar `max-w-screen-xl mx-auto` o
  similar para evitar que el contenido se estire en pantallas ultra-anchas.

---

## Componentes con shadcn/ui

- **shadcn/ui** como librería de componentes: se copian al proyecto, no son
  dependencia externa. Esto permite modificarlos sin restricciones.
- **No instalar component libraries completas** (Material UI, Chakra, Ant
  Design) junto con Tailwind. Elegir un sistema y mantenerlo.
- **Componentes UI en `src/components/ui/`**. Mantener la convención de
  shadcn: un archivo por componente, nombrado en `kebab-case`
  (`button.tsx`, `dialog.tsx`, `data-table.tsx`).

---

## Animaciones y transiciones

- **`transition-*` de Tailwind** para animaciones simples (hover, focus).
- **`tailwindcss-animate`** (plugin de shadcn) para animaciones de
  entrada/salida en modals, dropdowns, etc.
- **Framer Motion** solo si se necesitan animaciones complejas con estado.
  No importar una librería de 30kb para un fade-in.

---

## Performance

- **Purge en producción** es automático en Tailwind v3+. No agregar
  safelists salvo que haya clases generadas dinámicamente (y justificarlo).
- **No generar clases dinámicamente** con template literals:
  ```tsx
  // MAL — Tailwind no puede purgar esto
  className={`bg-${color}-500`}
  // BIEN — clases completas que Tailwind puede encontrar
  className={color === 'blue' ? 'bg-blue-500' : 'bg-red-500'}
  ```
- **Archivos grandes de componentes**: si un componente tiene 20+ clases
  de Tailwind en un solo elemento, considerar extraer a variantes con `cva`
  (class-variance-authority):
  ```ts
  import { cva } from "class-variance-authority";
  const buttonVariants = cva("inline-flex items-center rounded-md font-medium", {
    variants: {
      variant: {
        default: "bg-primary text-white hover:bg-primary-dark",
        outline: "border border-primary text-primary hover:bg-primary/10",
        ghost: "hover:bg-surface-muted",
      },
      size: {
        sm: "h-8 px-3 text-sm",
        md: "h-10 px-4 text-base",
        lg: "h-12 px-6 text-lg",
      },
    },
    defaultVariants: { variant: "default", size: "md" },
  });
  ```
