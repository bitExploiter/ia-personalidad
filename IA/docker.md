# Docker — Convenciones

---

## Imágenes y builds

- **Un servicio por contenedor.** Nunca meter app + db en el mismo container.
- **Multi-stage builds** para Go y TypeScript. Imagen final mínima:
  - Go: `FROM scratch` o `FROM alpine:3.x` (si se necesita shell/certs).
  - Node/React: `FROM node:XX-alpine` para build, `FROM nginx:alpine` para
    servir estáticos.
- **Orden de capas para cache**:
  1. Instalar dependencias del sistema.
  2. Copiar archivo de dependencias (`go.mod`, `package.json`).
  3. Instalar dependencias de la app.
  4. Copiar código fuente.
  5. Build.
- **No correr como root.** Agregar usuario no privilegiado:
  ```dockerfile
  RUN addgroup -S app && adduser -S app -G app
  USER app
  ```
- **.dockerignore** siempre. Excluir como mínimo: `.git`, `node_modules`,
  `.env`, binarios locales, `*.md`.
- **Variables de entorno** para configuración. Nunca hardcodear URLs, puertos
  o credenciales en el Dockerfile.
- **Health checks** en el Dockerfile o en compose:
  ```dockerfile
  HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost:8080/health || exit 1
  ```

---

## Docker Compose (desarrollo)

- `compose.yaml` (nombre actual, no `docker-compose.yml`).
- **Volúmenes con nombre** para datos persistentes (DB). Nunca bind mounts
  para datos de base de datos.
- **depends_on con condition** para orden de arranque:
  ```yaml
  depends_on:
    db:
      condition: service_healthy
  ```
- **Red explícita** si hay más de 2 servicios.
- **Archivo `.env.example`** en el repo con todas las variables necesarias
  (sin valores secretos). El `.env` real va en `.gitignore`.
