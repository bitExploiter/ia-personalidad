# Autenticación y Autorización — Convenciones

---

## Estrategia general

**JWT con access + refresh token.** Stateless en el backend, tokens firmados,
rotación de refresh token para prevenir reutilización.

---

## Tokens

### Access Token

- **Corta duración**: 15 minutos.
- Se envía en el header `Authorization: Bearer <token>`.
- **Nunca almacenar en localStorage** — vulnerable a XSS.
- Almacenar en memoria (variable de la app o Zustand) + cookie `HttpOnly`
  como respaldo para revalidar.
- Payload mínimo:
  ```json
  {
    "sub": "user-uuid",
    "role": "admin",
    "permissions": ["users:read", "users:write"],
    "exp": 1700000000,
    "iat": 1699999100
  }
  ```

### Refresh Token

- **Larga duración**: 7 días (configurable por proyecto).
- Se almacena en cookie `HttpOnly`, `Secure`, `SameSite=Strict`.
- **Rotación obligatoria**: cada vez que se usa un refresh token para obtener
  un nuevo access token, se invalida el refresh anterior y se emite uno nuevo.
- Si se detecta reutilización de un refresh token ya rotado → **invalidar toda
  la familia de tokens** del usuario (posible compromiso).

---

## Backend (Go)

### Middleware de autenticación

```go
// internal/middleware/auth.go

// AuthMiddleware valida el access token JWT y inyecta el usuario en el context.
// Si el token es inválido o ha expirado, responde 401.
func AuthMiddleware(jwtSecret []byte) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := extractBearerToken(r)
            if token == "" {
                respondError(w, http.StatusUnauthorized, "UNAUTHORIZED", "Token missing")
                return
            }

            claims, err := validateJWT(token, jwtSecret)
            if err != nil {
                respondError(w, http.StatusUnauthorized, "UNAUTHORIZED", "Invalid or expired token")
                return
            }

            ctx := context.WithValue(r.Context(), userClaimsKey, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

### Middleware de autorización (RBAC)

```go
// internal/middleware/rbac.go

// RequirePermission verifica que el usuario autenticado tenga el permiso requerido.
// Si no lo tiene, responde 403.
func RequirePermission(permission string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims := getUserClaims(r.Context())
            if claims == nil {
                respondError(w, http.StatusUnauthorized, "UNAUTHORIZED", "Not authenticated")
                return
            }

            if !claims.HasPermission(permission) {
                respondError(w, http.StatusForbidden, "FORBIDDEN", "Insufficient permissions")
                return
            }

            next.ServeHTTP(w, r.WithContext(r.Context()))
        })
    }
}
```

### Aplicación en rutas

```go
// cmd/api/routes.go
mux.Handle("GET /api/users",
    AuthMiddleware(cfg.JWTSecret)(
        RequirePermission("users:read")(
            handler.ListUsers(),
        ),
    ),
)
```

---

## Frontend (React)

### Almacenamiento de tokens

```ts
// lib/auth.ts
// Access token en memoria — NO en localStorage
let accessToken: string | null = null;

export function getAccessToken(): string | null {
  return accessToken;
}

export function setAccessToken(token: string | null): void {
  accessToken = token;
}
```

### HTTP client con refresh automático

```ts
// lib/api-client.ts

/**
 * Fetch wrapper que inyecta el access token y refresca automáticamente
 * si recibe un 401.
 *
 * @param url - URL relativa al API base
 * @param options - RequestInit estándar
 * @returns Response parseada
 */
export async function apiFetch<T>(url: string, options?: RequestInit): Promise<T> {
  const token = getAccessToken();
  const response = await fetch(`${API_BASE}${url}`, {
    ...options,
    credentials: 'include', // envía cookies (refresh token)
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options?.headers,
    },
  });

  // Token expirado — intentar refresh
  if (response.status === 401) {
    const refreshed = await refreshAccessToken();
    if (refreshed) {
      return apiFetch<T>(url, options); // reintentar con nuevo token
    }
    // Refresh falló — cerrar sesión
    setAccessToken(null);
    window.location.href = '/login';
    throw new Error('Session expired');
  }

  if (!response.ok) {
    const error = await response.json();
    throw new ApiError(response.status, error);
  }

  return response.json();
}
```

### Protección de rutas

```tsx
// components/auth/ProtectedRoute.tsx
import { Navigate, Outlet } from 'react-router-dom';
import { useAuthStore } from '@/store/auth';

interface ProtectedRouteProps {
  requiredPermission?: string;
}

/**
 * Wrapper de ruta que redirige a /login si no hay sesión activa.
 * Opcionalmente verifica un permiso específico.
 */
export function ProtectedRoute({ requiredPermission }: ProtectedRouteProps) {
  const { user, isAuthenticated } = useAuthStore();

  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }

  if (requiredPermission && !user?.permissions.includes(requiredPermission)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return <Outlet />;
}
```

### Uso en router

```tsx
// routes.tsx
<Route element={<ProtectedRoute />}>
  <Route path="/dashboard" element={<Dashboard />} />
  <Route element={<ProtectedRoute requiredPermission="users:write" />}>
    <Route path="/admin/users" element={<UserManagement />} />
  </Route>
</Route>
```

---

## RBAC — Modelo de permisos

### Estructura

```
Roles → tienen → Permisos

admin   → users:read, users:write, users:delete, reports:read, settings:write
editor  → users:read, reports:read, reports:write
viewer  → users:read, reports:read
```

### Formato de permisos

`<recurso>:<acción>` en minúsculas con dos puntos como separador:
- `users:read`, `users:write`, `users:delete`
- `reports:read`, `reports:export`
- `settings:write`

### En base de datos

```sql
CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  key TEXT NOT NULL UNIQUE,  -- 'users:read'
  description TEXT NOT NULL
);

CREATE TABLE role_permissions (
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
  PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  PRIMARY KEY (user_id, role_id)
);
```

---

## Reglas de seguridad

- **Nunca** loguear tokens, contraseñas o datos sensibles.
- **bcrypt** para hashear contraseñas (cost mínimo: 12).
- **Rate limiting** en endpoints de login y refresh (ej: 5 intentos / minuto).
- **CORS** configurado explícitamente — nunca `Access-Control-Allow-Origin: *`
  en producción.
- **HTTPS obligatorio** — sin excepción en staging y producción.
- Validar **todos los inputs** con Zod en el frontend y con validación explícita
  en el backend. Nunca confiar en el cliente.
