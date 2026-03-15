# Continuous Deployment — Convenciones

---

## Arquitectura de entornos

Dos servidores, dos ramas, dos propósitos. Sin ambigüedad.

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub Repository                        │
│                                                                 │
│   feat/xyz ──PR──► testing ──PR──► main                        │
│                       │                │                        │
│                   CI + Tests       CI + Tests                   │
│                       │                │                        │
│                   ✅ Pass?         ✅ Pass?                     │
│                       │                │                        │
│                   Deploy auto      Deploy auto                  │
│                       ▼                ▼                        │
│              ┌──────────────┐  ┌──────────────┐                │
│              │   TESTING    │  │  PRODUCTION   │                │
│              │  IP: X.X.X.A │  │  IP: X.X.X.B │                │
│              │  testing.app │  │  app.com      │                │
│              └──────────────┘  └──────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

| Aspecto | Testing | Production |
|---------|---------|------------|
| Rama | `testing` | `main` |
| Servidor | Servidor A (IP dedicada) | Servidor B (IP dedicada) |
| Dominio | `testing.miapp.com` | `miapp.com` |
| Base de datos | DB de testing (datos de prueba) | DB de producción (datos reales) |
| Variables de entorno | `ENVIRONMENT=testing` | `ENVIRONMENT=production` |
| Deploy trigger | Merge a `testing` + CI pass | Merge a `main` + CI pass |
| Approval manual | No — automático | Sí — requiere aprobación |

---

## Flujo de ramas

```
feat/user-auth ──PR──► testing ──PR──► main

1. Developer crea rama feat/ o fix/ desde testing
2. PR a testing → CI corre tests → merge → deploy automático a servidor testing
3. QA valida en testing.miapp.com
4. PR de testing a main → CI corre tests → approval → deploy a producción
```

### Reglas de ramas

- **`main`**: producción. Protegida. Solo recibe PRs desde `testing`. Requiere
  approval manual + CI pass.
- **`testing`**: entorno de pruebas. Protegida. Recibe PRs desde ramas de
  feature/fix. Requiere CI pass.
- **`feat/*`, `fix/*`, `refactor/*`**: ramas de trabajo. Se crean desde
  `testing`, se mergean a `testing`.
- **Nunca** mergear directo a `main` saltándose `testing`.
- **Nunca** commitear directo a `testing` o `main`.

---

## GitHub Environments

Configurar en **Settings → Environments**:

### Environment: `testing`

```
- Deployment branch: testing
- No protection rules (deploy automático)
- Environment secrets:
    DEPLOY_HOST          → IP del servidor de testing
    DEPLOY_USER          → usuario SSH
    DEPLOY_SSH_KEY       → clave privada SSH
    DATABASE_URL         → conexión a DB de testing
    JWT_SECRET           → secret de testing (diferente al de prod)
    VITE_API_URL         → https://testing.miapp.com/api
```

### Environment: `production`

```
- Deployment branch: main
- Required reviewers: 1 (mínimo)
- Wait timer: 0 (o N minutos para cool-down)
- Environment secrets:
    DEPLOY_HOST          → IP del servidor de producción
    DEPLOY_USER          → usuario SSH
    DEPLOY_SSH_KEY       → clave privada SSH (DIFERENTE al de testing)
    DATABASE_URL         → conexión a DB de producción
    JWT_SECRET           → secret de producción
    VITE_API_URL         → https://miapp.com/api
```

---

## Protección de ramas

### `main` (producción)

- ✅ Require pull request before merging
- ✅ Require approvals (mínimo 1)
- ✅ Require status checks to pass (CI)
- ✅ Require branches to be up to date
- ✅ Restrict pushes — solo desde PRs
- ✅ Do not allow bypassing

### `testing`

- ✅ Require pull request before merging
- ✅ Require status checks to pass (CI)
- ✅ Require branches to be up to date
- ❌ No require approvals (agiliza el ciclo de testing)

---

## GitHub Actions — CI/CD completo

### Workflow 1: CI (corre en todo PR)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [testing, main]

jobs:
  test-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - run: npm ci
        working-directory: frontend

      - run: npm run lint
        working-directory: frontend

      - run: npm run type-check
        working-directory: frontend

      - run: npm run test
        working-directory: frontend

      - run: npm run build
        working-directory: frontend

  test-backend:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgis/postgis:16-3.4-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U test"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache-dependency-path: backend/go.sum

      - run: go vet ./...
        working-directory: backend

      - run: golangci-lint run
        working-directory: backend

      - run: go test -race -coverprofile=coverage.out ./...
        working-directory: backend
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb?sslmode=disable
          JWT_SECRET: test-secret-minimum-32-characters-long
          ENVIRONMENT: test
```

### Workflow 2: Deploy a testing (auto en merge a testing)

```yaml
# .github/workflows/deploy-testing.yml
name: Deploy to Testing

on:
  push:
    branches: [testing]

# Evitar deploys concurrentes al mismo entorno
concurrency:
  group: deploy-testing
  cancel-in-progress: false

jobs:
  # Primero: correr todos los tests
  ci:
    uses: ./.github/workflows/ci.yml

  # Después: deploy solo si CI pasó
  deploy:
    needs: ci
    runs-on: ubuntu-latest
    environment: testing

    steps:
      - uses: actions/checkout@v4

      # Build de imágenes Docker
      - name: Build backend image
        run: |
          docker build -t app-backend:testing -f backend/Dockerfile backend/

      - name: Build frontend image
        run: |
          docker build \
            --build-arg VITE_API_URL=${{ secrets.VITE_API_URL }} \
            -t app-frontend:testing \
            -f frontend/Dockerfile frontend/

      # Guardar imágenes como archivo para transferir
      - name: Save Docker images
        run: |
          docker save app-backend:testing app-frontend:testing | gzip > images.tar.gz

      # Transferir al servidor
      - name: Copy images to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          source: "images.tar.gz"
          target: "/tmp/deploy"

      # Copiar compose file
      - name: Copy compose file
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          source: "deploy/compose.testing.yaml"
          target: "/opt/app"
          strip_components: 1

      # Deploy en el servidor
      - name: Deploy on server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            cd /opt/app
            docker load < /tmp/deploy/images.tar.gz
            docker compose -f compose.testing.yaml up -d --force-recreate
            rm -f /tmp/deploy/images.tar.gz

            # Esperar health check
            echo "Waiting for health check..."
            for i in $(seq 1 30); do
              if curl -sf http://localhost:8080/health > /dev/null 2>&1; then
                echo "Health check passed!"
                exit 0
              fi
              sleep 2
            done
            echo "Health check failed!"
            exit 1

      # Notificar resultado
      - name: Notify deployment
        if: always()
        run: |
          echo "## Testing Deploy: ${{ job.status }}" >> $GITHUB_STEP_SUMMARY
          echo "- Branch: testing" >> $GITHUB_STEP_SUMMARY
          echo "- Commit: ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
          echo "- Server: testing.miapp.com" >> $GITHUB_STEP_SUMMARY
```

### Workflow 3: Deploy a producción (con approval, en merge a main)

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches: [main]

concurrency:
  group: deploy-production
  cancel-in-progress: false

jobs:
  ci:
    uses: ./.github/workflows/ci.yml

  deploy:
    needs: ci
    runs-on: ubuntu-latest
    environment: production  # ← requiere approval manual configurado en GitHub

    steps:
      - uses: actions/checkout@v4

      - name: Build backend image
        run: |
          docker build -t app-backend:production -f backend/Dockerfile backend/

      - name: Build frontend image
        run: |
          docker build \
            --build-arg VITE_API_URL=${{ secrets.VITE_API_URL }} \
            -t app-frontend:production \
            -f frontend/Dockerfile frontend/

      - name: Save Docker images
        run: |
          docker save app-backend:production app-frontend:production | gzip > images.tar.gz

      - name: Copy images to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          source: "images.tar.gz"
          target: "/tmp/deploy"

      - name: Copy compose file
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          source: "deploy/compose.production.yaml"
          target: "/opt/app"
          strip_components: 1

      - name: Deploy on server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            cd /opt/app
            docker load < /tmp/deploy/images.tar.gz
            docker compose -f compose.production.yaml up -d --force-recreate
            rm -f /tmp/deploy/images.tar.gz

            echo "Waiting for health check..."
            for i in $(seq 1 30); do
              if curl -sf http://localhost:8080/health > /dev/null 2>&1; then
                echo "Health check passed!"
                exit 0
              fi
              sleep 2
            done
            echo "Health check failed!"
            exit 1

      - name: Notify deployment
        if: always()
        run: |
          echo "## Production Deploy: ${{ job.status }}" >> $GITHUB_STEP_SUMMARY
          echo "- Branch: main" >> $GITHUB_STEP_SUMMARY
          echo "- Commit: ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
          echo "- Server: miapp.com" >> $GITHUB_STEP_SUMMARY
```

---

## Docker Compose por entorno

### Estructura de archivos

```
deploy/
  compose.testing.yaml       → compose para servidor de testing
  compose.production.yaml    → compose para servidor de producción
```

### Testing

```yaml
# deploy/compose.testing.yaml
name: app-testing

services:
  backend:
    image: app-backend:testing
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - ENVIRONMENT=testing
      - PORT=8080
      - DATABASE_URL=${DATABASE_URL}
      - JWT_SECRET=${JWT_SECRET}
      - CORS_ORIGINS=https://testing.miapp.com
      - LOG_LEVEL=debug     # más verboso en testing
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3

  frontend:
    image: app-frontend:testing
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"

  db:
    image: postgis/postgis:16-3.4-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db_data:
```

### Production

```yaml
# deploy/compose.production.yaml
name: app-production

services:
  backend:
    image: app-backend:production
    restart: always                # always en prod, unless-stopped en testing
    ports:
      - "8080:8080"
    environment:
      - ENVIRONMENT=production
      - PORT=8080
      - DATABASE_URL=${DATABASE_URL}
      - JWT_SECRET=${JWT_SECRET}
      - CORS_ORIGINS=https://miapp.com
      - LOG_LEVEL=info              # solo info+ en prod
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 512M              # límites de recursos en prod
        reservations:
          memory: 256M

  frontend:
    image: app-frontend:production
    restart: always
    ports:
      - "80:80"
      - "443:443"

  db:
    image: postgis/postgis:16-3.4-alpine
    restart: always
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db_data:
```

### Diferencias clave entre composes

| Aspecto | Testing | Production |
|---------|---------|------------|
| `restart` | `unless-stopped` | `always` |
| `LOG_LEVEL` | `debug` | `info` |
| `CORS_ORIGINS` | `testing.miapp.com` | `miapp.com` |
| Resource limits | Sin límites | Con `deploy.resources` |
| DB backups | No | Sí (cron externo o script) |

---

## Preparación del servidor

Script de primera vez para ambos servidores (testing y producción):

```bash
#!/bin/bash
# deploy/setup-server.sh
# Ejecutar como root una sola vez en cada servidor nuevo.

set -euo pipefail

# 1. Instalar Docker
curl -fsSL https://get.docker.com | sh

# 2. Crear usuario de deploy (sin password, solo SSH)
useradd -m -s /bin/bash deploy
usermod -aG docker deploy

# 3. Crear directorio de la app
mkdir -p /opt/app
chown deploy:deploy /opt/app

# 4. Configurar SSH para el usuario deploy
mkdir -p /home/deploy/.ssh
# (agregar la clave pública de GitHub Actions aquí)
# echo "ssh-ed25519 AAAA..." > /home/deploy/.ssh/authorized_keys
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh

# 5. Crear archivo .env en el servidor (una sola vez, manualmente)
cat > /opt/app/.env << 'ENVEOF'
DATABASE_URL=postgres://user:password@db:5432/appdb?sslmode=disable
JWT_SECRET=generate-with-openssl-rand-hex-32
DB_USER=user
DB_PASSWORD=password
DB_NAME=appdb
ENVEOF
chown deploy:deploy /opt/app/.env
chmod 600 /opt/app/.env

echo "Server setup complete. Remember to:"
echo "  1. Add the deploy SSH public key to /home/deploy/.ssh/authorized_keys"
echo "  2. Edit /opt/app/.env with real values"
echo "  3. Configure firewall (ufw allow 80,443,22)"
```

---

## Rollback

Si un deploy falla o introduce un bug en producción:

### Rollback rápido — revertir en Git

```bash
# 1. Crear rama de rollback
git checkout main
git checkout -b fix/rollback-v1.2.3

# 2. Revertir el merge commit que causó el problema
git revert -m 1 <merge-commit-sha>

# 3. PR directo a main (saltar testing en emergencias)
gh pr create --base main --title "fix: rollback v1.2.3 due to [reason]"
```

### Rollback manual — en el servidor

```bash
# SSH al servidor de producción
ssh deploy@PROD_IP

# Volver a la imagen anterior (Docker guarda las previas)
cd /opt/app
docker compose -f compose.production.yaml down
docker tag app-backend:previous app-backend:production
docker compose -f compose.production.yaml up -d
```

### Prevención: canary check post-deploy

El health check en el workflow de deploy ya verifica que el servicio responda.
Para verificación adicional:

```yaml
# Agregar al final del step de deploy
- name: Smoke test
  run: |
    sleep 10
    # Verificar que el API responde correctamente
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://${{ vars.APP_DOMAIN }}/api/health)
    if [ "$STATUS" != "200" ]; then
      echo "Smoke test failed! Status: $STATUS"
      exit 1
    fi
    echo "Smoke test passed"
```

---

## Secrets — qué va en cada lugar

| Secret | Dónde vive | Por qué ahí |
|--------|-----------|-------------|
| `DEPLOY_SSH_KEY` | GitHub Environment secrets | Usado por Actions para SSH |
| `DEPLOY_HOST` | GitHub Environment secrets | IP del servidor |
| `DEPLOY_USER` | GitHub Environment secrets | Usuario SSH |
| `DATABASE_URL` | `.env` en el servidor | No viaja por el pipeline |
| `JWT_SECRET` | `.env` en el servidor | No viaja por el pipeline |
| `VITE_API_URL` | GitHub Environment secrets | Se inyecta en build time |

**Regla:** los secrets de la app (`DATABASE_URL`, `JWT_SECRET`) viven en el
servidor, no en GitHub. GitHub solo necesita las credenciales de acceso SSH
para deployar y las variables de build time del frontend.

---

## Resumen del flujo completo

```
Developer
  └─ git push feat/xyz
       └─ PR a testing
            └─ CI: lint + type-check + tests + build
                 └─ ✅ Merge a testing
                      └─ CD: build images → scp → deploy → health check
                           └─ QA valida en testing.miapp.com
                                └─ PR de testing a main
                                     └─ CI: lint + type-check + tests + build
                                          └─ ✅ Approval manual
                                               └─ Merge a main
                                                    └─ CD: build → deploy → health check
                                                         └─ Live en miapp.com
```
