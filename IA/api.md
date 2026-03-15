# APIs — Formato de respuestas (convención global)

Independiente del lenguaje o framework, todas las APIs siguen esta estructura.

---

## Estructura de respuestas

```json
// Éxito - recurso individual
{
  "data": { ... }
}

// Éxito - lista con paginación
{
  "data": [...],
  "meta": {
    "total": 150,
    "page": 1,
    "per_page": 20,
    "total_pages": 8
  }
}

// Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "email is required",
    "details": [...]
  }
}
```

---

## Códigos HTTP

| Código | Significado             | Cuándo usarlo                              |
|--------|-------------------------|--------------------------------------------|
| `200`  | OK                      | GET exitoso, PUT/PATCH exitoso             |
| `201`  | Created                 | POST que crea un recurso nuevo             |
| `204`  | No Content              | DELETE exitoso, operaciones sin body        |
| `400`  | Bad Request             | Request malformado, JSON inválido          |
| `401`  | Unauthorized            | Sin autenticación o token inválido         |
| `403`  | Forbidden               | Autenticado pero sin permisos              |
| `404`  | Not Found               | Recurso no existe                          |
| `409`  | Conflict                | Duplicado, violación de constraint         |
| `422`  | Unprocessable Entity    | Validación de negocio fallida              |
| `500`  | Internal Server Error   | Error no controlado del servidor           |

---

## Códigos de error

Los `code` dentro de `error` deben ser strings constantes en `UPPER_SNAKE_CASE`
que el frontend pueda usar para lógica condicional:

```
VALIDATION_ERROR      → campo requerido, formato inválido, rango excedido
NOT_FOUND             → recurso no existe
UNAUTHORIZED          → token ausente o expirado
FORBIDDEN             → sin permisos para la acción
CONFLICT              → duplicado o violación de estado
INTERNAL_ERROR        → error inesperado del servidor
```

---

## Paginación

Usar offset-based como default. Cambiar a cursor-based solo si hay
requisito de rendimiento con datasets muy grandes (100k+ registros).

Query params estándar: `?page=1&per_page=20`.
Defaults: `page=1`, `per_page=20`. Máximo `per_page=100`.

---

## Versionado

Prefijo en la URL: `/api/v1/...`. No usar headers para versionado.
Solo crear `v2` cuando hay breaking changes reales — no por cada feature.
