# ia-personalidad

Sistema de convenciones para desarrollo de software con Claude Code.
Un conjunto modular de reglas, patrones y ejemplos que garantizan código
consistente, seguro y mantenible en todo proyecto.

---

## Qué es esto

Un framework de convenciones diseñado para trabajar con Claude Code (AI-assisted
development). Define **cómo** se escribe código, **cómo** se estructura un proyecto
y **qué patrones** seguir en cada capa del stack — desde la base de datos hasta
el componente de React.

Cada convención está en su propio archivo dentro de `IA/`. Claude Code carga
**solo el archivo relevante** para la tarea en curso, optimizando el uso del
contexto.

---

## Stack

### Frontend

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **React** | 19 | UI framework — hooks nuevos: `use()`, `useActionState`, `useOptimistic` |
| **TypeScript** | 5.x | Tipado estricto (`strict: true`), nunca `any` |
| **Zustand** | 5.x | Estado global del cliente — un store por dominio |
| **React Hook Form** | 7.x | Manejo de formularios — siempre con Zod |
| **Zod** | 3.x | Validación con schema — fuente de verdad para tipos |
| **shadcn/ui** | latest | Componentes UI (Radix + Tailwind) — código propio, no dependencia |
| **Tailwind CSS** | 4.x | Utility-first CSS — tokens semánticos, mobile-first |
| **Vitest** | latest | Test runner — unit + integration |
| **React Testing Library** | latest | Testing de componentes — comportamiento, no implementación |
| **Playwright** | latest | E2E testing — solo flujos críticos de negocio |
| **MSW** | latest | Mock de APIs en tests — intercepta fetch a nivel de red |

### Backend

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **Go** | 1.22+ | Backend — stdlib-first, mínimas dependencias |
| **GORM** | 2.x | ORM — repository pattern, scopes, hooks, migrations |
| **PostgreSQL** | 15+ | Base de datos relacional |
| **PostGIS** | 3.x | Extensión geoespacial para PostgreSQL |

### Infraestructura

| Tecnología | Propósito |
|------------|-----------|
| **Docker** | Contenedores — multi-stage builds, un servicio por container |
| **GitHub Actions** | CI: lint, type-check, test, build. CD: deploy automático |
| **GitHub Environments** | Testing (auto-deploy) + Production (approval manual) |
| **GitHub** | Repositorio, PRs, protección de ramas |

---

## Estructura del repositorio

```
ia-personalidad/
├── CLAUDE.md                  → Identidad, reglas universales, tabla de routing
├── README.md                  → Este documento
└── IA/                        → Convenciones por tecnología (1 archivo = 1 tema)
    ├── react.md               → React 19: hooks, estructura, patrones
    ├── zustand.md             → Estado global: stores, slices, selectores
    ├── react-hook-form.md     → Formularios: RHF + Zod, schema-first
    ├── shadcn.md              → Componentes UI: composición, cn(), temas
    ├── tailwind.md            → CSS: utility-first, tokens, dark mode, responsive
    ├── go.md                  → Backend: estructura, errores, HTTP, testing
    ├── gorm.md                → ORM: modelos, repository, scopes, transacciones
    ├── postgresql-postgis.md  → Base de datos: naming, indexes, spatial queries
    ├── docker.md              → Containers: builds, compose, health checks
    ├── api.md                 → API: formato de respuesta, códigos HTTP, paginación
    ├── github.md              → Git: commits, ramas, PRs, CI/CD, protección
    ├── auth.md                → Seguridad: JWT, refresh tokens, RBAC, middleware
    ├── error-handling.md      → Errores: tipados (Go), boundaries (React), propagación
    ├── testing.md             → Testing: pirámide, herramientas, qué testear
    ├── logging.md             → Logs: slog, niveles, qué loguear, correlación
    ├── config.md              → Configuración: env vars, validación, fail fast
    ├── deployment.md          → CD: testing/production servers, workflows, rollback
    ├── openspec.md            → Workflow: OpenSpec OPSX, cuándo usar cada comando
    └── stitch.md              → Diseño UI: flujo CDD, prompts Stitch, adaptación
```

---

## Arquitectura

### Frontend — Feature-based

```
src/
  components/
    ui/              ← shadcn/ui primitivos (generados, no editar)
    layout/          ← Header, Sidebar, Layout
  features/          ← Módulos por dominio de negocio
    users/
      components/    ← UI específica del feature
      hooks/         ← Hooks del feature
      services/      ← Llamadas al API
      store.ts       ← Store Zustand (si aplica)
      schemas.ts     ← Schemas Zod
      types.ts       ← Tipos derivados de Zod
      index.ts       ← Barrel export
  hooks/             ← Hooks globales reutilizables
  lib/               ← Utilidades (env.ts, api-client.ts, utils.ts)
  store/             ← Stores Zustand globales
  types/             ← Tipos globales y DTOs
  pages/             ← Componentes de página
```

### Backend — Handler / Service / Repository

```
cmd/<nombre>/        ← Entrypoint (main.go mínimo, solo wiring)
internal/
  config/            ← Struct de configuración, carga desde env
  handler/           ← HTTP transport (recibe request, devuelve response)
  service/           ← Lógica de negocio (no sabe de HTTP)
  repository/        ← Acceso a datos (queries SQL, no lógica)
  model/             ← Tipos del dominio
  middleware/         ← Auth, logging, request ID, CORS
  apperror/          ← Errores tipados de dominio
migrations/          ← SQL: up/down pairs
```

### Flujo de datos end-to-end

```
React Component
  → React Hook Form (validación Zod en cliente)
    → Service (fetch al API)
      → Go Handler (valida, delega)
        → Go Service (lógica de negocio)
          → Go Repository (query SQL)
            → PostgreSQL
          ← Resultado o AppError
        ← JSON: { data: ... } o { error: { code, message } }
      ← ApiError tipado en frontend
    ← Toast / inline error / redirect
  ← UI actualizada
```

---

## Principios fundamentales

### 1. Schema-first

Zod define el schema. El tipo se deriva con `z.infer<>`. Nunca crear tipos a mano
que repitan lo que el schema ya valida. Una sola fuente de verdad.

### 2. Errores como ciudadanos de primera clase

Backend: `AppError` tipado con código → Handler: JSON estandarizado → Frontend:
`ApiError` con helpers (`isNotFound`, `isForbidden`). La cadena nunca se rompe.

### 3. Estado local primero

```
useState → useReducer → Context (3+ niveles) → Zustand (global de cliente)
```

No saltar directamente a Zustand cuando `useState` resuelve el problema.

### 4. Composición sobre modificación

Los componentes de shadcn/ui no se editan. Se construyen componentes de dominio
encima de los primitivos.

### 5. Fail fast

Configuración inválida al inicio → la app no arranca. Mejor un error claro al
deploy que un `undefined` silencioso en producción a las 3 AM.

### 6. Seguridad por defecto

- JWT con access token (15 min) + refresh token (7 días, rotación obligatoria)
- RBAC: permisos en formato `<recurso>:<acción>`
- Nunca loguear tokens, passwords, PII
- HTTPS obligatorio en staging y producción
- bcrypt con cost 12+ para passwords
- CORS explícito, nunca wildcard en producción
- Rate limiting en login y refresh

---

## Testing

### Pirámide

```
        ┌─────────┐
        │  E2E    │  Pocos — login, registro, flujo principal
        ├─────────┤
        │ Integr. │  Moderados — endpoints completos, DB real
        ├─────────┤
        │  Unit   │  Muchos — schemas, stores, utils, componentes
        └─────────┘
```

| Nivel | Frontend | Backend |
|-------|----------|---------|
| Unit | Vitest + RTL + MSW | `go test` (table-driven) |
| Integración | Vitest + MSW | `go test` + testcontainers |
| E2E | Playwright | — |

### CI Pipeline

```yaml
# En cada PR:
- npm run lint
- npm run type-check    # tsc --noEmit
- npm run test          # Vitest
- npm run build
- go test ./...

# Nightly / pre-release:
- npx playwright test   # E2E
```

---

## Workflow de desarrollo — OpenSpec (OPSX)

El desarrollo sigue la metodología **[OpenSpec](https://github.com/Fission-AI/OpenSpec)**
con el flujo **OPSX**: cada tarea se define en un *change* pequeño y focalizado
antes de escribir código. Contexto acotado para la IA → resultados más precisos
→ menos iteraciones.

### Ciclo por tarea

```
/opsx:explore → /opsx:propose → /opsx:apply → verificar → commit → /opsx:archive
```

| Paso | Comando | Qué pasa |
|------|---------|----------|
| 1 | `/opsx:explore` | Investigar el área del codebase antes de proponer nada |
| 2 | `/opsx:propose` | Crear el change con artefactos: proposal, specs, design, tasks |
| 3 | `/opsx:apply` | Ejecutar las tasks del change contra el código |
| 4 | Verificar | Comprobar resultado (visual, curl, tests) |
| 5 | Commit | Commit semántico por capa lógica |
| 6 | `/opsx:archive` | Archivar el change completado |

### Cuándo usar OpenSpec vs conversación directa

| Situación | Usar |
|-----------|------|
| Feature nuevo | OpenSpec (explore + propose + apply) |
| Bug fix simple, 1-2 archivos | Conversación directa |
| Refactor de módulo completo | OpenSpec — el scope acotado es crítico |
| Pregunta o explicación | Conversación directa |
| Feature que toca múltiples dominios | OpenSpec — dividir en varios changes |
| Typo, renombrar variable | Conversación directa |

### Comandos adicionales

| Comando | Cuándo usarlo |
|---------|---------------|
| `/opsx:verify` | Validar que la implementación coincide con los artefactos |
| `/opsx:ff` | Crear todos los artefactos de planificación de golpe |
| `/opsx:continue` | Crear artefactos uno por uno (control granular) |
| `/opsx:bulk-archive` | Archivar varios changes completados a la vez |
| `/opsx:onboard` | Tutorial guiado para aprender el workflow |

> Ver [IA/openspec.md](IA/openspec.md) para la referencia completa de integración
> con las convenciones del proyecto.

---

## Git workflow y Deployment

### Ramas

```
feat/xyz ──PR──► testing ──PR──► main
                    │                │
                CI + Tests       CI + Tests
                    │                │
                Deploy auto     Deploy + Approval
                    ▼                ▼
             Testing Server    Production Server
```

- **Commits:** `<tipo>(<scope>): <descripción en imperativo>`
  - Tipos: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `style`, `perf`
- **Ramas de trabajo:** `<tipo>/<descripcion-kebab-case>` — se crean desde `testing`
- **`testing`:** recibe PRs de features. Merge → CI → deploy automático al servidor de testing
- **`main`:** recibe PRs desde `testing`. Merge → CI → approval manual → deploy a producción
- **PRs:** pequeños (< 400 líneas), una sola responsabilidad, squash and merge

### Entornos

| | Testing | Production |
|---|---|---|
| Rama | `testing` | `main` |
| Servidor | IP dedicada | IP dedicada |
| Deploy | Automático | Con approval |
| Logs | `debug` | `info` |
| CORS | `testing.miapp.com` | `miapp.com` |

---

## Cómo usar en un proyecto nuevo

1. **Clonar este repo** en tu máquina como referencia de convenciones.

2. **Copiar `CLAUDE.md`** a la raíz de tu nuevo proyecto. Editar la tabla de
   stack para reflejar las tecnologías que aplican.

3. **Copiar la carpeta `IA/`** al nuevo proyecto. Eliminar los archivos de
   tecnologías que no uses.

4. **Crear un `PRD.md`** en la raíz del proyecto con:
   - Nombre y descripción del proyecto
   - Stack específico (del que aplique de este repo)
   - Estructura de datos / modelos principales
   - Features principales y prioridades

5. **Claude Code** leerá `CLAUDE.md` automáticamente y consultará los archivos
   de `IA/` según la tarea en curso.

---

## Convenciones incluidas

| Archivo | Qué define | Líneas |
|---------|-----------|--------|
| `CLAUDE.md` | Identidad, reglas universales, flujo de trabajo | ~125 |
| `IA/react.md` | React 19: hooks, estructura, componentes, testing | ~155 |
| `IA/zustand.md` | Stores, slices, selectores, anti-patrones | ~130 |
| `IA/react-hook-form.md` | Forms + Zod: schema-first, validación, testing | ~160 |
| `IA/shadcn.md` | UI components: composición, cn(), temas, catálogo | ~100 |
| `IA/tailwind.md` | CSS: utility-first, tokens, responsive, dark mode | ~115 |
| `IA/go.md` | Backend: estructura, errores, HTTP, testing | ~70 |
| `IA/gorm.md` | ORM: modelos, repository, scopes, transacciones, hooks | ~250 |
| `IA/postgresql-postgis.md` | DB: naming, constraints, indexes, spatial | ~60 |
| `IA/docker.md` | Containers: multi-stage, compose, healthchecks | ~50 |
| `IA/api.md` | API: response format, HTTP codes, pagination | ~85 |
| `IA/github.md` | Git: commits, branches, PRs, CI/CD, protection | ~215 |
| `IA/auth.md` | Security: JWT, refresh, RBAC, middleware, routes | ~280 |
| `IA/error-handling.md` | Errors: typed (Go), boundaries (React), chain | ~275 |
| `IA/testing.md` | Testing: pyramid, tools, what to test, CI | ~310 |
| `IA/logging.md` | Logging: slog, levels, correlation, health checks | ~180 |
| `IA/config.md` | Config: env vars, typed loading, fail fast | ~170 |
| `IA/deployment.md` | CD: two-server setup, workflows, rollback | ~370 |
| `IA/openspec.md` | Workflow: OpenSpec OPSX, cuándo y cómo usar cada comando | ~125 |
| `IA/stitch.md` | Diseño UI: flujo CDD, prompts Stitch, checklist adaptación | ~200 |
| **Total** | | **~3,425** |

---

## Licencia

MIT
