# OpenSpec (OPSX) — Workflow oficial de desarrollo

---

## Por qué OpenSpec

El desarrollo asistido por IA es más preciso cuando el contexto es reducido.
OpenSpec formaliza esto: cada tarea se define en un **change pequeño y focalizado**
antes de escribir código. Contexto acotado → precisión mayor → menos iteraciones.

**OpenSpec es la metodología oficial de este proyecto.** Los comandos OPSX
(`/opsx:explore`, `/opsx:propose`, `/opsx:apply`, `/opsx:archive`, etc.) los
define y ejecuta OpenSpec. Este documento establece cuándo usarlos, cómo integran
con nuestras convenciones y qué criterios seguir para changes de calidad.

---

## Ciclo de trabajo en este proyecto

```
/opsx:explore → /opsx:propose → /opsx:apply → verificar → commit → /opsx:archive
```

### 1. Explorar antes de proponer

Usar `/opsx:explore` cuando:
- La tarea toca código que no conoces bien.
- Antes de crear un feature que se conecta con módulos existentes.
- Cuando no estás seguro de si ya existe algo que resuelve el problema.

No necesitas `/opsx:explore` para:
- Tareas triviales (cambiar un texto, corregir un typo).
- Cuando ya sabes exactamente qué archivos vas a tocar.

### 2. Crear el change

Usar `/opsx:propose` para crear el change con todos los artefactos de
planificación (proposal, specs, design, tasks) en un solo paso.

Para control más granular (workflow expandido), usar `/opsx:new` + `/opsx:continue`
para crear artefactos uno por uno, o `/opsx:ff` para crearlos todos de golpe.

**Criterios para un buen change en este proyecto:**
- **Scope mínimo.** Si el change toca más de 5 archivos, dividirlo en varios.
- **Una sola responsabilidad.** Un change = una cosa bien hecha.
- **Referenciar convenciones.** Mencionar qué archivos de `IA/` aplican
  (ej: "seguir IA/react.md para componentes, IA/shadcn.md para UI").
- **Definición de hecho medible.** Checklist concreto, no "funciona bien".
- **Si el change incluye UI nueva:** incluir en el artefacto `design.md` el
  prompt de Stitch informado por la spec (campos, endpoints, roles). Ver
  [stitch.md](stitch.md) para el template y el checklist de adaptación.

### 3. Aplicar el change

Usar `/opsx:apply` cuando los artefactos están listos.

**Reglas durante el apply:**
- No modificar el scope a mitad del apply. Si aparece algo inesperado,
  parar, actualizar los artefactos y hacer un nuevo apply.
- Si el apply falla o produce algo incorrecto, corregir los artefactos — no
  intentar guiar con mensajes adicionales.
- Verificar el resultado según la tabla de verificación de `CLAUDE.md`
  (regla #1) antes de hacer commit.

### 4. Verificar (opcional pero recomendado)

Usar `/opsx:verify` antes de archivar para validar que la implementación
coincide con los artefactos. Detecta tareas incompletas, specs no cubiertas
y divergencias entre diseño y código.

### 5. Archivar

Usar `/opsx:archive` cuando el change fue aplicado, verificado y commiteado.

---

## Comandos adicionales

| Comando | Cuándo usarlo |
|---------|---------------|
| `/opsx:continue` | Crear artefactos uno por uno (workflow expandido) |
| `/opsx:ff` | Crear todos los artefactos de golpe (workflow expandido) |
| `/opsx:verify` | Validar implementación antes de archivar |
| `/opsx:sync` | Mergear delta specs en specs principales |
| `/opsx:bulk-archive` | Archivar varios changes completados a la vez |
| `/opsx:onboard` | Tutorial guiado para aprender el workflow |

---

## Integración con el flujo de ramas

```
1. git checkout -b feat/user-avatar  (desde testing)

2. /opsx:explore → entender el área de usuarios

3. /opsx:propose → crear el change del feature

4. Commitear los artefactos (buena práctica):
   feat(spec): add user-avatar change

5. /opsx:apply → ejecutar las tasks del change

6. Verificar resultado (visual / curl / tests)

7. Commitear el código:
   feat(users): add UserAvatar component with initials fallback

8. /opsx:archive → archivar el change completado

9. Commitear el archive:
   chore(specs): archive user-avatar change

10. PR a testing → CI → deploy a testing server
```

---

## Cuándo usar OpenSpec vs conversación directa

| Situación | Usar |
|-----------|------|
| Feature nuevo | OpenSpec (explore + propose + apply) |
| Bug fix simple, 1-2 archivos | Conversación directa |
| Refactor de módulo completo | OpenSpec — el scope acotado es crítico |
| Pregunta o explicación | Conversación directa |
| Feature que toca múltiples dominios | OpenSpec — dividir en varios changes |
| Typo, renombrar variable | Conversación directa |
| Integración con sistema externo | OpenSpec — contexto externo en la spec |

---

## Anti-patrones

```
❌ Change con scope "agregar autenticación completa"
   → Demasiado amplio. Dividir: jwt-middleware, refresh-token, protected-routes

❌ Change que dice "hacer que funcione mejor"
   → Sin definición de hecho. ¿Qué significa "mejor"?

❌ Usar /opsx:apply sin haber hecho /opsx:explore en código desconocido
   → Claude puede colisionar con patrones existentes

❌ Modificar el scope durante el apply porque "apareció algo"
   → Parar, actualizar los artefactos, nuevo apply

❌ No archivar los changes completados
   → El historial se pierde, el directorio de changes se llena de ruido

❌ Redefinir en el change lo que ya dice una convención de IA/
   → Referenciar el archivo, no copiar su contenido
```
