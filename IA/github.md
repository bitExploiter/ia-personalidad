# GitHub — Mejores prácticas

---

## Commits

Formato semántico obligatorio — sin excepciones:

```
<tipo>(<scope opcional>): <descripción en imperativo, en inglés>

[cuerpo opcional — qué y por qué, no cómo]

[footer opcional — referencias a issues, breaking changes]
```

**Tipos:**

| Tipo       | Cuándo usarlo                                              |
|------------|------------------------------------------------------------|
| `feat`     | Nueva funcionalidad visible al usuario                     |
| `fix`      | Corrección de bug                                          |
| `refactor` | Cambio de código sin alterar comportamiento ni agregar feat|
| `docs`     | Solo documentación                                         |
| `test`     | Agregar o corregir tests                                   |
| `chore`    | Tareas de mantenimiento (deps, config, build)              |
| `style`    | Formato, espacios, punto y coma — sin cambio de lógica     |
| `perf`     | Mejora de rendimiento                                      |

**Ejemplos:**

```
feat(auth): add JWT refresh token rotation
fix(cart): prevent duplicate items on rapid click
refactor(user-store): split into slices for better maintainability
chore: upgrade shadcn/ui to latest
docs(api): document pagination parameters
```

**Reglas:**

- Descripción en minúsculas, imperativo, sin punto final.
- Máximo 72 caracteres en la primera línea.
- Un commit por responsabilidad lógica. No mezclar feat + fix en el mismo commit.
- No commits de "WIP" o "fix fix fix" en ramas que se van a mergear.

---

## Ramas

Nomenclatura:

```
<tipo>/<descripción-en-kebab-case>

feat/user-authentication
fix/cart-duplicate-items
refactor/split-user-store
chore/upgrade-dependencies
```

**Reglas:**

- `main` (o `master`) es siempre deployable. Nunca commitear directo.
- `develop` como rama de integración si el equipo trabaja con Gitflow.
- Ramas cortas — idealmente un feature o fix por rama.
- Eliminar la rama después del merge.

---

## Pull Requests

**Título:** mismo formato que los commits — semántico y en inglés.

**Template de descripción:**

```markdown
## ¿Qué hace este PR?
<!-- Descripción concisa del cambio -->

## ¿Por qué?
<!-- Contexto, issue que resuelve, decisión técnica relevante -->

## Cómo probar
<!-- Pasos para verificar el cambio manualmente -->
- [ ] paso 1
- [ ] paso 2

## Checklist
- [ ] Tests agregados / actualizados
- [ ] Documentación actualizada
- [ ] No hay console.log ni código de debug
- [ ] Revisé mi propio diff antes de solicitar review

Closes #<número de issue>
```

**Reglas:**

- PRs pequeños y enfocados. Si supera 400 líneas cambiadas, evaluar si se puede dividir.
- Un PR resuelve una sola cosa. No mezclar refactor con nueva funcionalidad.
- No mergear sin al menos un review aprobado (en equipos).
- Resolver todos los comentarios antes de mergear — o marcarlos como `won't fix` con justificación.
- Usar **Squash and Merge** para features. **Merge commit** para releases. Nunca **Rebase and Merge** en ramas compartidas.

---

## Issues

**Nomenclatura de labels estándar:**

| Label        | Uso                                      |
|--------------|------------------------------------------|
| `bug`        | Algo no funciona como se espera          |
| `feat`       | Nueva funcionalidad                      |
| `refactor`   | Deuda técnica, limpieza                  |
| `docs`       | Documentación faltante o incorrecta      |
| `blocked`    | Depende de otro issue o decisión externa |
| `priority:high` | Debe resolverse en el sprint actual   |

---

## GitHub Actions — CI mínimo

Todo proyecto debe tener al menos este pipeline:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test
      - run: npm run build
```

**Reglas:**

- El CI bloquea el merge si falla. Sin excepciones.
- Agregar `type-check` (`tsc --noEmit`) como step separado del build.
- Cachear dependencias siempre (`cache: 'npm'` o equivalente).
- Secrets nunca en el código — siempre en **GitHub Secrets** o **Environments**.
- Para deploys: usar **Environments** con protección de rama en producción
  (requiere approval manual antes de deployar a prod).

---

## Protección de ramas

Configurar en **Settings → Branches** para `main`:

- ✅ Require pull request before merging
- ✅ Require approvals (mínimo 1)
- ✅ Require status checks to pass (CI)
- ✅ Require branches to be up to date
- ✅ Do not allow bypassing the above settings

---

## `.gitignore` mínimo para proyectos Node/React

```gitignore
# Dependencias
node_modules/

# Build
dist/
build/
.next/
out/

# Env — NUNCA en el repo
.env
.env.local
.env.*.local

# Editor
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Testing
coverage/

# Logs
*.log
npm-debug.log*
```

**Regla sobre `.env`:** Commitear siempre un `.env.example` con las variables
necesarias pero sin valores reales. Es la documentación de configuración.
