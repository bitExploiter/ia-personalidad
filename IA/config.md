# Gestión de Configuración — Convenciones

---

## Principio

La configuración vive **fuera del código**. El binario/app es el mismo en
desarrollo, staging y producción — lo que cambia es la configuración inyectada
vía variables de entorno.

---

## Jerarquía de configuración

```
1. Variables de entorno del sistema (más alta prioridad)
2. Archivo .env (solo en desarrollo local)
3. Valores por defecto en código (solo para desarrollo)
```

En producción, la configuración viene exclusivamente de variables de entorno
(inyectadas por Docker, Kubernetes, CI/CD, etc.).

---

## Backend (Go)

### Struct de configuración tipada

```go
// internal/config/config.go

// Config contiene toda la configuración de la aplicación.
// Se carga una sola vez al inicio y se pasa por inyección de dependencias.
type Config struct {
    Port        int    `env:"PORT"         envDefault:"8080"`
    DatabaseURL string `env:"DATABASE_URL" required:"true"`
    JWTSecret   string `env:"JWT_SECRET"   required:"true"`
    Environment string `env:"ENVIRONMENT"  envDefault:"development"`
    LogLevel    string `env:"LOG_LEVEL"    envDefault:"info"`
    CORSOrigins string `env:"CORS_ORIGINS" envDefault:"http://localhost:5173"`
}

// IsProd indica si la aplicación corre en producción.
func (c Config) IsProd() bool {
    return c.Environment == "production"
}
```

### Carga y validación al inicio

```go
// internal/config/load.go
import "github.com/caarlos0/env/v11"

// Load carga la configuración desde variables de entorno.
// Falla inmediatamente si falta una variable requerida —
// es preferible no arrancar a arrancar mal configurado.
func Load() (Config, error) {
    var cfg Config
    if err := env.Parse(&cfg); err != nil {
        return Config{}, fmt.Errorf("parse config: %w", err)
    }
    return cfg, nil
}
```

### Uso en main

```go
// cmd/api/main.go
func main() {
    cfg, err := config.Load()
    if err != nil {
        slog.Error("failed to load config", "error", err)
        os.Exit(1) // fail fast — no arrancar sin config válida
    }

    db := database.Connect(cfg.DatabaseURL)
    server := handler.NewServer(cfg, db)
    // ...
}
```

### Reglas para Go

- **Un struct, una carga, al inicio.** No leer `os.Getenv()` disperso por el
  código — toda la config se carga una vez en `main()` y se inyecta.
- **Fallar rápido.** Si falta una variable requerida, no arrancar. Mejor un
  error claro al inicio que un panic a las 3 AM.
- **No usar `viper`** salvo que el proyecto necesite archivos YAML/TOML
  complejos. `caarlos0/env` + struct tags es más simple y suficiente.
- **Nunca valores por defecto para secrets.** `JWT_SECRET`, `DATABASE_URL` y
  similares deben ser `required: true`.

---

## Frontend (React / Vite)

### Variables de entorno con prefijo `VITE_`

```bash
# .env.local (solo desarrollo, en .gitignore)
VITE_API_URL=http://localhost:8080/api
VITE_APP_NAME=MiApp
```

### Acceso tipado

```ts
// lib/env.ts
import { z } from 'zod';

/**
 * Schema de las variables de entorno del frontend.
 * Se valida al iniciar la app — falla inmediatamente si falta algo.
 */
const envSchema = z.object({
  VITE_API_URL: z.string().url(),
  VITE_APP_NAME: z.string().min(1),
});

/**
 * Variables de entorno validadas y tipadas.
 * Importar desde aquí, nunca usar `import.meta.env` directo.
 */
export const env = envSchema.parse(import.meta.env);
```

### Uso en el código

```ts
// ❌ Mal: acceso directo, sin validación, sin tipado
const url = import.meta.env.VITE_API_URL;

// ✅ Bien: importar desde lib/env.ts
import { env } from '@/lib/env';
const url = env.VITE_API_URL; // tipado y validado
```

### Reglas para frontend

- **Prefijo `VITE_` obligatorio** para que Vite exponga la variable al bundle.
- **Nunca secrets en el frontend.** Todo lo que llega al browser es público.
  API keys privadas, JWT secrets, etc. van solo en el backend.
- **Validar con Zod al arrancar** — si falta una variable, mejor un error
  en consola que un `undefined` silencioso en runtime.
- **Un solo punto de acceso** (`lib/env.ts`) — si cambia el nombre de una
  variable, se cambia en un solo lugar.

---

## Archivos de entorno

### Estructura obligatoria en el repo

```
.env.example      → Todas las variables, sin valores secretos (commitear)
.env              → Valores reales de desarrollo local (en .gitignore)
.env.local        → Override local del desarrollador (en .gitignore)
.env.test         → Variables para tests (puede commitearse si no tiene secrets)
```

### `.env.example` — la documentación de la config

```bash
# .env.example — Commitear siempre. Documentar cada variable.

# === Servidor ===
PORT=8080
ENVIRONMENT=development    # development | staging | production

# === Base de datos ===
DATABASE_URL=              # postgres://user:pass@host:5432/dbname?sslmode=disable

# === Auth ===
JWT_SECRET=                # Mínimo 32 caracteres. Generar con: openssl rand -hex 32

# === Frontend ===
VITE_API_URL=http://localhost:8080/api
VITE_APP_NAME=MiApp

# === CORS ===
CORS_ORIGINS=http://localhost:5173
```

### Reglas

- `.env.example` es **obligatorio** en todo proyecto. Es la documentación
  viva de qué configuración necesita la app.
- Cada variable tiene un **comentario** explicando qué es y, si aplica, cómo
  generarla.
- **Nunca** commitear `.env` con valores reales.
- **Nunca** poner valores de producción en `.env.example`.

---

## Docker y CI/CD

### Docker Compose (desarrollo)

```yaml
# compose.yaml
services:
  api:
    env_file: .env
    environment:
      - ENVIRONMENT=development
```

### CI/CD (GitHub Actions)

```yaml
# Variables no secretas: en el workflow
env:
  ENVIRONMENT: test

# Secrets: en GitHub Settings → Secrets
steps:
  - run: go test ./...
    env:
      DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
      JWT_SECRET: ${{ secrets.TEST_JWT_SECRET }}
```

### Producción

Variables inyectadas por la plataforma de hosting:
- **Docker/K8s**: `env` en el manifiesto o ConfigMap/Secret.
- **Railway/Fly/Render**: panel de variables de entorno.
- **Nunca** montar un archivo `.env` en producción — usar variables nativas.
