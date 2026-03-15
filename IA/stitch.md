# Google Stitch — Convenciones de diseño UI

---

## Qué es Stitch y su rol en el flujo

[Google Stitch](https://stitch.withgoogle.com/) genera diseños UI y código
exportable (HTML/CSS, Tailwind, JSX) a partir de prompts en lenguaje natural.

En este proyecto, Stitch es una **herramienta de diseño**, no de código final.
El código exportado siempre pasa por un proceso de adaptación al stack del
proyecto (shadcn/ui + Tailwind 4 + React 19 + TypeScript). Nunca se usa
directamente en producción.

---

## Flujo CDD (Contract-Driven Development)

El diseño de interfaces está informado por los contratos ya definidos, no por
suposiciones. Primero datos, luego diseño, luego código:

```
Schema DB (postgresql-postgis.md)
  → Contratos API (api.md)
    → Prompt Stitch (informado por schema + API + roles)
      → Stitch genera diseño + código
        → Adaptar a stack (shadcn + Tailwind 4 + React 19 + TS)
          → Componente final en features/<dominio>/components/
```

| Fase | Input | Output | Convención |
|------|-------|--------|------------|
| 1. Modelar datos | Requisitos de negocio | Schema SQL | [postgresql-postgis.md](postgresql-postgis.md) |
| 2. Definir contratos | Schema SQL | Endpoints + DTOs | [api.md](api.md) |
| 3. Generar prompt | Schema + API + roles | Prompt estructurado | Este archivo |
| 4. Diseño en Stitch | Prompt | Diseño visual + código | stitch.withgoogle.com |
| 5. Adaptar código | Export de Stitch | Componentes React | [react.md](react.md), [shadcn.md](shadcn.md), [tailwind.md](tailwind.md) |

---

## Template de prompt para Stitch

Estructura recomendada para generar prompts consistentes. Adaptar según la
vista — no todas las secciones aplican siempre.

```
## Contexto
[Nombre de la app / módulo]. [Descripción breve del dominio].

## Entidad principal
- Campos: [lista de campos con tipos humanos, no SQL]
- Estados: [enum/CHECK values si aplican]
- Relaciones: [FKs relevantes para la UI]

## Endpoints que alimentan esta vista
- GET /api/v1/[recurso] → lista paginada (campos: ...)
- GET /api/v1/[recurso]/:id → detalle (campos: ...)
- POST /api/v1/[recurso] → crear (body: ...)
- PATCH /api/v1/[recurso]/:id → actualizar (body: ...)

## Roles y permisos
- [rol]: puede [acciones]
- [rol]: puede [acciones]

## Validaciones visibles en UI
- [campo]: [regla] (ej: "email: formato email, requerido")
- [campo]: [regla]

## Vista solicitada
[Tipo: tabla/lista, formulario, dashboard, detalle, etc.]
[Descripción del layout deseado]
[Comportamientos: filtros, ordenamiento, paginación, modales]

## Estilo
- Design system: shadcn/ui (Radix primitives + Tailwind CSS)
- Colores: tokens semánticos (primary, destructive, muted)
- Responsive: mobile-first
- Modo oscuro: sí (dark mode con class strategy)
```

### Ejemplo concreto

```
## Contexto
Panel de administración de una app de gestión de equipos. Módulo de usuarios.

## Entidad principal
- Campos: nombre (texto, requerido), email (email, único, requerido),
  rol (enum, requerido), avatar (URL, opcional), fecha de registro (fecha)
- Estados: active, suspended, invited
- Relaciones: pertenece a un equipo (team_id)

## Endpoints que alimentan esta vista
- GET /api/v1/users → lista paginada (nombre, email, rol, status, created_at)
- POST /api/v1/users → crear (nombre, email, rol, team_id)
- PATCH /api/v1/users/:id → actualizar (nombre, email, rol, status)

## Roles y permisos
- admin: puede crear, editar, suspender usuarios
- member: solo puede ver la lista (sin botones de acción)

## Validaciones visibles en UI
- nombre: mínimo 2 caracteres, requerido
- email: formato email válido, requerido, único
- rol: selección entre admin, member, viewer

## Vista solicitada
Tabla de usuarios con:
- Columnas: avatar + nombre, email, rol (badge de color), estado (badge), fecha
- Filtros: por rol (select), por estado (select), búsqueda por nombre/email
- Paginación: 20 por página, botones prev/next con contador
- Botón "Nuevo usuario" que abre un modal con formulario de creación
- Acciones por fila: editar (modal), suspender (confirmación)
- Fila clickeable → navega al detalle del usuario

## Estilo
- Design system: shadcn/ui (Radix primitives + Tailwind CSS)
- Colores: tokens semánticos (primary para acciones, destructive para suspender,
  muted para estados inactivos)
- Responsive: mobile-first (tabla se convierte en cards en mobile)
- Modo oscuro: sí
```

---

## Reglas para buenos prompts

1. **Campos en lenguaje de negocio, no SQL.** Stitch entiende "email (texto,
   requerido)" mejor que `VARCHAR(255) NOT NULL`.

2. **Incluir estados y enums.** Si la entidad tiene status con valores fijos,
   listarlos — Stitch los convierte en badges, selects o filtros.

3. **Especificar el tipo de vista.** "Tabla con paginación y filtros" es mejor
   que "mostrar los datos".

4. **Indicar qué campos mostrar.** Si la entidad tiene 15 campos, decir cuáles
   van en la tabla vs cuáles solo en el detalle.

5. **Incluir permisos cuando afectan la UI.** Si un rol no puede editar, el
   prompt debe decir "botón de editar visible solo para admin".

6. **Pedir responsive siempre.** Incluir "mobile-first, responsive" para que
   Stitch no genere layouts solo desktop.

7. **Pedir tokens semánticos.** Decir "colores semánticos: primary, destructive,
   muted" en vez de "azul, rojo, gris".

---

## Checklist de adaptación del export

### TypeScript
- [ ] Convertir a `.tsx` si viene como `.jsx` o `.html`
- [ ] Tipar props con interface explícita (`XxxProps`)
- [ ] Derivar tipos de schemas Zod existentes (`z.infer<>`), no crear nuevos
- [ ] Named export, no default export

### shadcn/ui
- [ ] `<button>` → `<Button>` de shadcn
- [ ] `<input>` → `<Input>` de shadcn
- [ ] `<table>` → componentes `Table` de shadcn
- [ ] Modales custom → `<Dialog>` de shadcn
- [ ] Selects → `<Select>` de shadcn
- [ ] Etiquetas de estado → `<Badge>` de shadcn
- [ ] Loading states → `<Skeleton>` de shadcn

### Tailwind CSS
- [ ] Colores literales (`blue-500`) → tokens semánticos (`primary`)
- [ ] Verificar orden mobile-first (base → `md:` → `lg:`)
- [ ] Clases condicionales con `cn()`, nunca template literals
- [ ] Si hay 20+ clases en un elemento, evaluar extraer a `cva`

### Estructura de archivos
- [ ] Ubicar en `features/<dominio>/components/`
- [ ] Separar componente de página vs componente reutilizable
- [ ] Importar desde barrel file (`@/features/<dominio>`)

### Formularios
- [ ] Usar schema Zod de `features/<dominio>/schemas.ts` (no crear otro)
- [ ] Integrar con React Hook Form + `<Form>` de shadcn
- [ ] Estructura: `FormField` → `FormItem` → `FormLabel` → `FormControl` → `FormMessage`
- [ ] Ver [react-hook-form.md](react-hook-form.md) para el patrón completo

### Datos
- [ ] Reemplazar datos hardcoded → fetch desde `features/<dominio>/services/`
- [ ] Usar hooks del feature para estado
- [ ] Paginación: `?page=1&per_page=20` (ver [api.md](api.md))

---

## Integración con OpenSpec (OPSX)

El prompt de Stitch vive dentro del artefacto **`design.md`** del change OPSX:

```markdown
## Diseño UI — Stitch

### Prompt
[prompt completo siguiendo el template]

### Output
- URL del diseño en Stitch: [link]
- Código exportado adaptado en: features/<dominio>/components/XxxComponent.tsx

### Decisiones de adaptación
- [qué se cambió del export y por qué]
- [qué primitivos de shadcn reemplazaron HTML nativo]
- [qué se descartó del export]
```

La spec del change debe mencionar los campos y endpoints que informan el prompt.
La task de apply debe incluir "adaptar export de Stitch" como paso explícito.

---

## Anti-patrones

```
❌ Usar el código de Stitch tal cual sin adaptar
   → Siempre pasa por el checklist de adaptación

❌ Prompt sin datos del schema
   → El prompt debe estar informado por campos reales, no inventados

❌ Crear tipos nuevos para el componente exportado
   → Los tipos ya existen en schemas.ts (Zod). Derivar, no duplicar

❌ Prompt que dice "hazlo bonito" o "diseña la página"
   → Especificar: tipo de vista, campos visibles, interacciones, responsive

❌ Editar los primitivos de shadcn para que se parezcan al export
   → Crear wrappers de dominio encima de los primitivos (ver shadcn.md)

❌ Ignorar el mobile-first del export
   → Stitch puede generar layouts desktop-only. Siempre adaptar a mobile-first

❌ Prompt de Stitch en un chat sin documentar
   → Vive en design.md del change OPSX. Si no está documentado, no existe
```
