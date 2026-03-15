# Manejo de Errores — Convenciones

---

## Principio

Los errores son ciudadanos de primera clase. Se anticipan, se tipan, se
propagan con contexto y se muestran al usuario de forma útil.
Nunca silenciar un error. Nunca mostrar un stack trace al usuario.

---

## Backend (Go)

### Errores como valores — siempre con contexto

```go
// ❌ Mal: error sin contexto
return err

// ✅ Bien: envolver con contexto
return fmt.Errorf("get user id=%s: %w", id, err)
```

### Errores de dominio tipados

```go
// internal/apperror/errors.go

// AppError representa un error de dominio con código y mensaje para el API.
type AppError struct {
    Code    string // "NOT_FOUND", "VALIDATION_ERROR", etc.
    Message string // mensaje para el usuario
    Status  int    // HTTP status code
    Err     error  // error original (no se expone al cliente)
}

func (e *AppError) Error() string { return e.Message }
func (e *AppError) Unwrap() error { return e.Err }

func NotFound(resource string, id string) *AppError {
    return &AppError{
        Code:    "NOT_FOUND",
        Message: fmt.Sprintf("%s with id %s not found", resource, id),
        Status:  http.StatusNotFound,
    }
}

func ValidationError(message string) *AppError {
    return &AppError{
        Code:    "VALIDATION_ERROR",
        Message: message,
        Status:  http.StatusUnprocessableEntity,
    }
}

func Forbidden(message string) *AppError {
    return &AppError{
        Code:    "FORBIDDEN",
        Message: message,
        Status:  http.StatusForbidden,
    }
}

func Internal(err error) *AppError {
    return &AppError{
        Code:    "INTERNAL_ERROR",
        Message: "An unexpected error occurred",
        Status:  http.StatusInternalServerError,
        Err:     err, // se loguea, no se envía al cliente
    }
}
```

### Handler: centralizar respuesta de errores

```go
// internal/handler/helper.go

// handleError inspecciona el error, loguea si es interno, y responde con
// el formato estándar del API.
func handleError(w http.ResponseWriter, r *http.Request, err error) {
    var appErr *apperror.AppError
    if errors.As(err, &appErr) {
        if appErr.Status >= 500 {
            slog.Error("internal error",
                "method", r.Method,
                "path", r.URL.Path,
                "error", appErr.Err,
            )
        }
        respondJSON(w, appErr.Status, map[string]any{
            "error": map[string]any{
                "code":    appErr.Code,
                "message": appErr.Message,
            },
        })
        return
    }

    // Error no tipado — tratar como interno
    slog.Error("unhandled error",
        "method", r.Method,
        "path", r.URL.Path,
        "error", err,
    )
    respondJSON(w, http.StatusInternalServerError, map[string]any{
        "error": map[string]any{
            "code":    "INTERNAL_ERROR",
            "message": "An unexpected error occurred",
        },
    })
}
```

### Regla: nunca exponer errores internos al cliente

```go
// ❌ Mal: expone detalles de implementación
respondError(w, 500, err.Error())

// ✅ Bien: mensaje genérico, error real en logs
handleError(w, r, apperror.Internal(err))
```

---

## Frontend (React)

### Tipos de error y cómo manejarlos

| Tipo de error | Ejemplo | Manejo |
|---------------|---------|--------|
| **Validación** | Campo vacío, email inválido | Inline en el form (Zod + RHF) |
| **API — esperado** | 404, 409, 422 | Mostrar mensaje en UI (toast, inline) |
| **API — auth** | 401, 403 | Redirect a login / pantalla de "sin permisos" |
| **Red** | Timeout, offline | Toast + opción de reintentar |
| **Runtime** | Componente explota | Error Boundary con fallback |

### Error Boundary — atrapar errores de render

```tsx
// components/ErrorBoundary.tsx
import { Component, type ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback: ReactNode;
}

interface State {
  hasError: boolean;
}

/**
 * Atrapa errores de render en el árbol de hijos y muestra un fallback.
 * No atrapa errores en event handlers ni async — solo render/lifecycle.
 */
export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo): void {
    // Aquí enviar a servicio de monitoreo (Sentry, etc.)
    console.error('ErrorBoundary caught:', error, info.componentStack);
  }

  render(): ReactNode {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

### Uso: envolver por sección, no toda la app

```tsx
// ❌ Mal: un solo boundary para todo — un error tumba toda la UI
<ErrorBoundary fallback={<FullPageError />}>
  <App />
</ErrorBoundary>

// ✅ Bien: boundaries granulares por sección
<Layout>
  <ErrorBoundary fallback={<SidebarError />}>
    <Sidebar />
  </ErrorBoundary>
  <ErrorBoundary fallback={<ContentError />}>
    <MainContent />
  </ErrorBoundary>
</Layout>
```

### Error en llamadas a servicios

```ts
// lib/api-error.ts

/**
 * Error tipado para respuestas del API. Encapsula código y mensaje
 * del formato estándar de error.
 */
export class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    public userMessage: string,
  ) {
    super(userMessage);
    this.name = 'ApiError';
  }

  get isNotFound(): boolean { return this.status === 404; }
  get isValidation(): boolean { return this.status === 422; }
  get isForbidden(): boolean { return this.status === 403; }
  get isUnauthorized(): boolean { return this.status === 401; }
  get isServerError(): boolean { return this.status >= 500; }
}
```

### Manejo en componentes

```tsx
import { toast } from '@/components/ui/use-toast';

async function handleDelete(id: string) {
  try {
    await deleteUser(id);
    toast({ title: 'Usuario eliminado' });
  } catch (error) {
    if (error instanceof ApiError) {
      if (error.isNotFound) {
        toast({ title: 'Usuario no encontrado', variant: 'destructive' });
        return;
      }
      if (error.isForbidden) {
        toast({ title: 'Sin permisos para esta acción', variant: 'destructive' });
        return;
      }
    }
    // Error inesperado
    toast({ title: 'Algo salió mal. Intenta de nuevo.', variant: 'destructive' });
  }
}
```

---

## Propagación backend → frontend

```
Go service error
  → apperror.NotFound("user", id)
    → handler: { "error": { "code": "NOT_FOUND", "message": "..." } }
      → apiFetch: throw new ApiError(404, "NOT_FOUND", "...")
        → componente: if (error.isNotFound) → toast / inline message
```

La cadena nunca se rompe. Cada capa agrega contexto, ninguna pierde información.

---

## Reglas

- **Nunca `catch` vacío.** Todo error se maneja o se re-lanza.
- **Nunca `console.error` como único manejo** — eso no es manejo, es logging.
- **Errores de validación** se resuelven inline en el form (Zod), no con toast.
- **Errores de red** siempre ofrecen reintentar.
- **Errores 500** muestran mensaje genérico al usuario + se envían a monitoreo.
- **Error boundaries** por sección de UI, no uno global.
