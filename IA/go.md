# Go — Convenciones

---

## Estructura de proyecto

Seguir la convención de la comunidad Go (no el repo "golang-standards" — ese
no es oficial). Estructura recomendada:

```
cmd/<nombre>/          → entrypoint (main.go mínimo, solo wiring)
internal/              → código privado del módulo
  handler/             → capa de transporte HTTP (recibe request, devuelve response)
  service/             → lógica de negocio (no sabe de HTTP)
  repository/          → acceso a datos (queries, no lógica)
  model/               → tipos del dominio
  middleware/           → middleware HTTP reutilizable
pkg/                   → código exportable a otros proyectos (solo si aplica)
migrations/            → archivos SQL de migración
```

---

## Convenciones

- **Errores como valores.** Siempre wrapping con contexto:
  `fmt.Errorf("repository.GetUser id=%d: %w", id, err)`. Nunca `log.Fatal`
  fuera de `main.go`.
- **Interfaces en el consumidor**, no en el productor. Definir la interfaz
  donde se usa, no donde se implementa.
- **Sin init()** salvo casos excepcionales justificados.
- **Context como primer parámetro** en toda función que haga I/O.
- **Naming**: `snake_case` para archivos, `camelCase` para variables locales,
  `PascalCase` para exportados. Nombres cortos en scope corto, descriptivos
  en scope largo.
- **Structs de configuración** en vez de muchos parámetros. Si una función
  tiene más de 3 parámetros, agrupar en struct.
- **Constructores con `New`**: `NewUserService(repo UserRepository) *UserService`.
- **Cero dependencias innecesarias.** La stdlib de Go es excelente. Justificar
  cada dependencia externa.
- **Linter**: `golangci-lint` con configuración del proyecto. Como mínimo
  habilitar: `errcheck`, `govet`, `staticcheck`, `unused`.

---

## HTTP y APIs

- **Router**: `chi` o `net/http` (Go 1.22+ con su router mejorado).
- **JSON encoding**: `encoding/json` de la stdlib. Usar struct tags completos:
  `json:"field_name,omitempty"`.
- **Middleware estándar**: logging, recovery, request ID, CORS, auth.
  Implementar como `func(next http.Handler) http.Handler`.
- **Respuestas**: seguir el formato global definido en
  [api.md](api.md).
- **Validación en la capa handler**, antes de llamar al service.

---

## Testing

- Tests en el mismo paquete para caja blanca, en `_test` package para caja
  negra (preferir caja negra).
- **Table-driven tests** como patrón por defecto.
- Naming: `TestNombreFuncion_escenario_resultado`.
- **Fixtures** en `testdata/`.
- **No mocks salvo para boundaries** (DB, HTTP externo). Preferir fakes o
  implementaciones in-memory.
