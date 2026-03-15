# Logging y Observabilidad — Convenciones

---

## Principio

Los logs son para **diagnosticar problemas**, no para narrar la ejecución.
Log estructurado (JSON), con contexto suficiente para reconstruir lo que pasó
sin necesidad de debugger ni acceso al servidor.

---

## Backend (Go) — `log/slog`

### Regla: usar `slog` de la stdlib (Go 1.21+)

No se necesita librería externa. `slog` soporta JSON, niveles y campos
estructurados out of the box.

### Setup

```go
// cmd/api/main.go
import "log/slog"

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo, // producción: Info, desarrollo: Debug
    }))
    slog.SetDefault(logger)
}
```

### Niveles y cuándo usar cada uno

| Nivel | Cuándo | Ejemplo |
|-------|--------|---------|
| `Debug` | Detalle para desarrollo. Nunca en producción. | `slog.Debug("cache hit", "key", cacheKey)` |
| `Info` | Eventos normales relevantes para operaciones. | `slog.Info("user created", "user_id", id)` |
| `Warn` | Algo inesperado pero recuperable. | `slog.Warn("retry attempt", "attempt", 3, "error", err)` |
| `Error` | Fallo que necesita atención. | `slog.Error("payment failed", "order_id", id, "error", err)` |

### Campos estructurados — siempre pares key-value

```go
// ❌ Mal: mensaje con interpolación
slog.Info(fmt.Sprintf("user %s created order %s", userID, orderID))

// ✅ Bien: campos estructurados
slog.Info("order created",
    "user_id", userID,
    "order_id", orderID,
    "total", total,
)
```

Esto produce JSON buscable:
```json
{"time":"2026-03-14T10:00:00Z","level":"INFO","msg":"order created","user_id":"abc","order_id":"123","total":59.99}
```

### Request logging — middleware

```go
// internal/middleware/logging.go

// RequestLogger loguea method, path, status y duración de cada request HTTP.
func RequestLogger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        wrapped := &statusRecorder{ResponseWriter: w, statusCode: http.StatusOK}

        next.ServeHTTP(wrapped, r)

        slog.Info("http request",
            "method", r.Method,
            "path", r.URL.Path,
            "status", wrapped.statusCode,
            "duration_ms", time.Since(start).Milliseconds(),
            "remote_addr", r.RemoteAddr,
        )
    })
}
```

### Contexto de request — propagar request ID

```go
// internal/middleware/request_id.go

// RequestID genera un ID único por request y lo inyecta en el context.
// Todos los logs dentro de ese request incluyen el ID para correlación.
func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = uuid.NewString()
        }
        ctx := context.WithValue(r.Context(), requestIDKey, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

---

## Frontend (React)

### Regla: no loguear en producción sin propósito

- **Desarrollo**: `console.log`, `console.warn`, `console.error` — libre.
- **Producción**: solo errores que van a servicio de monitoreo.
  Nunca `console.log` en producción — contamina y expone información.

### Servicio de error reporting

```ts
// lib/error-reporter.ts

/**
 * Reporta errores a un servicio externo (Sentry, LogRocket, etc.).
 * En desarrollo, solo imprime en consola.
 */
export function reportError(error: Error, context?: Record<string, unknown>): void {
  if (import.meta.env.DEV) {
    console.error('[Error Reporter]', error, context);
    return;
  }

  // Producción: enviar a Sentry u otro servicio
  // Sentry.captureException(error, { extra: context });
}
```

### Integración con Error Boundary

```tsx
// Ver error-handling.md para la implementación completa del ErrorBoundary
componentDidCatch(error: Error, info: React.ErrorInfo): void {
  reportError(error, { componentStack: info.componentStack });
}
```

---

## Lo que NUNCA se loguea

| Dato | Por qué |
|------|---------|
| Contraseñas / hashes | Obvio |
| Tokens JWT (access o refresh) | Compromete la sesión |
| Tarjetas de crédito / datos de pago | PCI compliance |
| PII sin necesidad (email, teléfono, dirección) | GDPR / privacidad |
| Cuerpo completo de requests POST | Puede contener cualquiera de los anteriores |
| Variables de entorno | Pueden contener secrets |

### Alternativa: loguear IDs, nunca valores sensibles

```go
// ❌ Mal
slog.Info("login attempt", "email", user.Email, "password", user.Password)

// ✅ Bien
slog.Info("login attempt", "user_id", user.ID)
```

---

## Lo que SÍ se loguea

| Evento | Nivel | Campos mínimos |
|--------|-------|-----------------|
| Request HTTP completado | Info | method, path, status, duration_ms |
| Recurso creado/actualizado/eliminado | Info | resource, id, user_id |
| Error de negocio (validación, not found) | Warn | error_code, resource, id |
| Error interno (DB caída, panic recover) | Error | error, stack (solo en log, no al cliente) |
| Autenticación exitosa | Info | user_id |
| Autenticación fallida | Warn | remote_addr (nunca el password intentado) |
| Refresh token rotation | Info | user_id |
| Permiso denegado | Warn | user_id, permission, resource |

---

## Correlación de logs

Cada request debe ser trazable. El `X-Request-ID` viaja:

```
Cliente → API Gateway → Backend → DB query logs
                  ↓
            Log con request_id="abc-123"
```

En cada log dentro de un handler, incluir el request ID:

```go
func (h *UserHandler) Create() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        reqID := getRequestID(r.Context())
        // ... lógica ...
        slog.Info("user created", "request_id", reqID, "user_id", newUser.ID)
    }
}
```

---

## Health checks

Endpoint obligatorio en todo servicio:

```go
// internal/handler/health.go

// Health responde 200 si el servicio está operativo.
// Verifica conexión a DB y otros servicios críticos.
func Health(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
        defer cancel()

        if err := db.PingContext(ctx); err != nil {
            slog.Error("health check failed", "error", err)
            w.WriteHeader(http.StatusServiceUnavailable)
            json.NewEncoder(w).Encode(map[string]string{"status": "unhealthy"})
            return
        }

        json.NewEncoder(w).Encode(map[string]string{"status": "healthy"})
    }
}
```

URL estándar: `GET /health` — sin autenticación, sin rate limit.
