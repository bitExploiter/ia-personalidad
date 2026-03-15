# Testing — Estrategia integral

---

## Filosofía

Testear lo que importa, no lo que es fácil. Un test que nunca falla no
protege de nada. Un test que falla por razones irrelevantes erosiona la
confianza en el suite.

---

## Pirámide de testing

```
        ┌─────────┐
        │  E2E    │  Pocos, lentos, alto valor — flujos críticos de negocio
        ├─────────┤
        │ Integr. │  Moderados — servicios + DB, API endpoints completos
        ├─────────┤
        │  Unit   │  Muchos, rápidos — lógica pura, schemas, utilidades
        └─────────┘
```

| Nivel | Qué se testea | Herramienta | Velocidad |
|-------|---------------|-------------|-----------|
| Unit | Funciones puras, schemas Zod, stores Zustand, utils | Vitest (frontend), `go test` (backend) | ms |
| Integración | Endpoints completos, queries a DB, servicios con dependencias reales | Vitest + MSW (frontend), `go test` + testcontainers (backend) | segundos |
| E2E | Flujos de usuario completos en browser | Playwright | segundos-minutos |

---

## Frontend — Vitest + React Testing Library

### Reglas

- **Testear comportamiento, no implementación.** Nunca testear state interno
  de un componente ni snapshots de HTML.
- **Buscar por rol, label o texto** — nunca por clase CSS o `data-testid`
  (salvo último recurso).
- **Un test = un comportamiento.** Si el test necesita más de un `expect`
  para cosas distintas, dividirlo.

### Unit tests

```ts
// features/users/schemas.test.ts
import { describe, it, expect } from 'vitest';
import { createUserSchema } from './schemas';

describe('createUserSchema', () => {
  it('acepta datos válidos', () => {
    const result = createUserSchema.safeParse({
      name: 'Ana García',
      email: 'ana@ejemplo.com',
      role: 'admin',
    });
    expect(result.success).toBe(true);
  });

  it('rechaza email inválido', () => {
    const result = createUserSchema.safeParse({
      name: 'Ana',
      email: 'no-es-email',
      role: 'admin',
    });
    expect(result.success).toBe(false);
    expect(result.error?.flatten().fieldErrors.email).toBeDefined();
  });
});
```

### Tests de componentes

```tsx
// features/users/components/UserCard.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { UserCard } from './UserCard';

describe('UserCard', () => {
  const mockUser = { id: '1', name: 'Ana García', email: 'ana@test.com' };

  it('muestra el nombre del usuario', () => {
    render(<UserCard user={mockUser} onSelect={vi.fn()} />);
    expect(screen.getByText('Ana García')).toBeInTheDocument();
  });

  it('llama onSelect al hacer click', async () => {
    const onSelect = vi.fn();
    render(<UserCard user={mockUser} onSelect={onSelect} />);
    await userEvent.click(screen.getByRole('button'));
    expect(onSelect).toHaveBeenCalledWith('1');
  });
});
```

### Tests de stores Zustand

```ts
// features/cart/store.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { useCartStore } from './store';

describe('useCartStore', () => {
  beforeEach(() => {
    // Reset del store entre tests
    useCartStore.setState({ items: [] });
  });

  it('agrega un item al carrito', () => {
    useCartStore.getState().addItem({ id: '1', name: 'Item', qty: 1 });
    expect(useCartStore.getState().items).toHaveLength(1);
  });

  it('incrementa qty si el item ya existe', () => {
    const item = { id: '1', name: 'Item', qty: 1 };
    useCartStore.getState().addItem(item);
    useCartStore.getState().addItem(item);
    expect(useCartStore.getState().items[0].qty).toBe(2);
  });
});
```

### Mocking de APIs con MSW

```ts
// tests/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json({
      data: [{ id: '1', name: 'Ana', email: 'ana@test.com' }],
    });
  }),

  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ data: { id: '2', ...body } }, { status: 201 });
  }),
];

// tests/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// tests/setup.ts — configurar en vitest.config.ts como setupFiles
import { beforeAll, afterEach, afterAll } from 'vitest';
import { server } from './mocks/server';

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

## Backend (Go)

### Reglas

- **Table-driven tests** como patrón por defecto.
- **Tests de integración con DB real** — usar testcontainers para PostgreSQL.
  No mocks de base de datos.
- **Interfaces en el consumidor** — facilita testear services con repositorios
  fake solo cuando la DB no es práctica.

### Table-driven tests

```go
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        want  bool
    }{
        {"valid email", "user@example.com", true},
        {"missing @", "userexample.com", false},
        {"empty string", "", false},
        {"with subdomain", "user@mail.example.com", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := ValidateEmail(tt.email)
            if got != tt.want {
                t.Errorf("ValidateEmail(%q) = %v, want %v", tt.email, got, tt.want)
            }
        })
    }
}
```

### Tests de handlers (HTTP)

```go
func TestListUsersHandler(t *testing.T) {
    svc := &mockUserService{
        users: []model.User{{ID: "1", Name: "Ana"}},
    }
    handler := NewUserHandler(svc)

    req := httptest.NewRequest(http.MethodGet, "/api/users", nil)
    rec := httptest.NewRecorder()

    handler.List().ServeHTTP(rec, req)

    if rec.Code != http.StatusOK {
        t.Errorf("status = %d, want %d", rec.Code, http.StatusOK)
    }

    var resp struct {
        Data []model.User `json:"data"`
    }
    json.NewDecoder(rec.Body).Decode(&resp)
    if len(resp.Data) != 1 {
        t.Errorf("got %d users, want 1", len(resp.Data))
    }
}
```

---

## E2E — Playwright

### Cuándo escribir E2E

Solo para **flujos críticos de negocio**:
- Login / Logout
- Registro de usuario
- Flujo de compra completo (si aplica)
- CRUD principal del dominio

No testear cada variante de UI en E2E — eso es responsabilidad de unit/integration.

### Estructura

```
e2e/
  fixtures/       → datos de prueba, helpers
  auth.spec.ts    → login, logout, refresh
  users.spec.ts   → CRUD de usuarios
```

### Ejemplo

```ts
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('user can login with valid credentials', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('admin@test.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Iniciar sesión' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('Bienvenido')).toBeVisible();
  });

  test('shows error for invalid credentials', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('wrong@test.com');
    await page.getByLabel('Password').fill('wrong');
    await page.getByRole('button', { name: 'Iniciar sesión' }).click();

    await expect(page.getByText('Credenciales inválidas')).toBeVisible();
  });
});
```

---

## Naming

| Tipo | Patrón | Ejemplo |
|------|--------|---------|
| Unit (componente) | `NombreComponente.test.tsx` | `UserCard.test.tsx` |
| Unit (hook) | `useHook.test.ts` | `useCartStore.test.ts` |
| Unit (schema) | `schemas.test.ts` | `schemas.test.ts` |
| Unit (util) | `util.test.ts` | `formatDate.test.ts` |
| Integration (API) | `feature.integration.test.ts` | `users.integration.test.ts` |
| E2E | `feature.spec.ts` | `auth.spec.ts` |
| Go | `archivo_test.go` | `user_service_test.go` |

---

## CI — qué corre en cada etapa

```yaml
# En cada PR:
- npm run test          # unit + integration (Vitest)
- go test ./...         # unit + integration (Go)

# Nightly o antes de release:
- npx playwright test   # E2E (más lento, requiere browser)
```

---

## Qué NO testear

- Implementación interna de componentes (state, renders internos).
- Código generado (shadcn/ui components, tipos auto-generados).
- Getters/setters triviales sin lógica.
- Frameworks y librerías de terceros — confía en sus propios tests.
