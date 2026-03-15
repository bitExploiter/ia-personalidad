# OpenSpec — Workflow oficial de desarrollo

---

## Filosofía

OpenSpec resuelve el problema principal del desarrollo asistido por IA:
**el contexto amplio produce respuestas genéricas**. Cuando le das a Claude
todo el proyecto, obtiene una respuesta promedio. Cuando le das exactamente
lo que necesita saber para esta tarea, obtiene la respuesta correcta.

La metodología funciona así: antes de escribir código, escribes una **spec**
— un documento pequeño y preciso que define el scope, el objetivo y las
restricciones de la tarea. Claude trabaja sobre la spec, no sobre el proyecto
entero. Contexto acotado → precisión mayor → menos iteraciones.

---

## Ciclo de vida de una spec

```
/explore        →      /new         →      /apply       →    /archive

Entender el      Crear la spec       Ejecutar la spec    Guardar como
terreno          (scope + reglas)    contra el código    registro
```

---

## Los 4 comandos

### `/explore`

**Cuándo:** al inicio de cualquier tarea no trivial, antes de crear la spec.

**Qué hace:** explora el codebase en el área relevante para entender
estructura actual, patrones existentes, dependencias y posibles colisiones
antes de proponer nada.

**Cuándo usarlo:**
- La tarea toca código que no conoces bien.
- Antes de crear un feature que se conecta con módulos existentes.
- Cuando no estás seguro de si ya existe algo que resuelve el problema.

**Cuándo NO usarlo:**
- Tareas triviales (cambiar un texto, corregir un typo).
- Cuando ya sabes exactamente qué archivos vas a tocar.

---

### `/new`

**Cuándo:** después de `/explore` (o directamente si el scope es claro).

**Qué hace:** crea un archivo de spec en `specs/` con el scope exacto,
objetivo, archivos involucrados y restricciones de la tarea.

**Estructura de una spec bien escrita:**

```markdown
# spec: <nombre-en-kebab-case>

## Objetivo
<!-- Una sola frase: qué se va a lograr -->

## Scope
<!-- Qué archivos se van a crear o modificar. Ser específico. -->
- Crear: `features/users/components/UserAvatar.tsx`
- Modificar: `features/users/schemas.ts`
- No tocar: el resto del proyecto

## Contexto
<!-- Solo lo que Claude necesita saber para esta tarea -->
- El componente usa shadcn/ui Avatar
- El schema ya tiene `avatarUrl?: string`
- Ver convenciones: IA/react.md, IA/shadcn.md

## Restricciones
<!-- Reglas que no se pueden violar -->
- Named export, no default
- Props tipadas con interface
- No agregar nuevas dependencias

## Definición de hecho
<!-- Cómo saber que la tarea está completa -->
- [ ] Componente renderiza avatar o fallback con iniciales
- [ ] Tests unitarios del comportamiento de fallback
- [ ] Sin errores de TypeScript
```

**Reglas para escribir specs:**
- **Scope mínimo.** Si la spec toca más de 5 archivos, dividirla.
- **Una sola responsabilidad.** Una spec = una cosa bien hecha.
- **Explícita sobre implícita.** Mencionar los archivos de convenciones
  relevantes. No asumir que Claude los recuerda.
- **Definición de hecho medible.** Checklist concreto, no "funciona bien".

---

### `/apply`

**Cuándo:** la spec está lista y revisada.

**Qué hace:** ejecuta la spec — Claude lee la spec + los archivos del scope
y produce el código. Solo carga lo que la spec indica. Nada más.

**Reglas:**
- No modificar el scope durante el apply. Si aparece algo inesperado,
  parar, actualizar la spec y hacer un nuevo apply.
- Si el apply falla o produce algo incorrecto, corregir la spec, no
  intentar "guiar" a Claude con mensajes adicionales.
- Verificar el resultado según la tabla de verificación de `CLAUDE.md`
  antes de hacer commit.

---

### `/archive`

**Cuándo:** la spec fue aplicada, verificada, y el commit está hecho.

**Qué hace:** mueve la spec de `specs/` a `specs/archive/` con la fecha
de completado. Sirve como registro histórico de decisiones.

**Estructura de archivo:**

```
specs/
  active-spec.md          ← specs en progreso
  another-spec.md

specs/archive/
  2026-03-14-user-avatar.md    ← specs completadas (con fecha)
  2026-03-10-auth-refresh.md
```

**Convención de nombre al archivar:**
```
specs/archive/<YYYY-MM-DD>-<nombre-original>.md
```

---

## Estructura de carpetas en el proyecto

```
mi-proyecto/
  specs/              ← specs activas (commiteadas, revisables en PR)
  specs/archive/      ← specs completadas (historial)
  IA/                 ← convenciones (este repo)
  CLAUDE.md
  PRD.md
```

Las specs se commitean. Son documentación de intención — cualquier miembro
del equipo puede entender qué se planeó hacer y por qué.

---

## Integración con el flujo de ramas

```
1. Crear rama desde testing:
   git checkout -b feat/user-avatar

2. /explore → entender el área de usuarios

3. /new → crear specs/user-avatar.md

4. Commitear la spec (opcional, buena práctica):
   feat(spec): add user-avatar spec

5. /apply → Claude ejecuta la spec

6. Verificar resultado (visual / curl / tests)

7. Commitear el código:
   feat(users): add UserAvatar component with initials fallback

8. /archive → mover spec a specs/archive/2026-03-14-user-avatar.md

9. Commitear el archive:
   chore(specs): archive user-avatar spec

10. PR a testing → CI → deploy a testing server
```

---

## Cuándo usar OpenSpec vs conversación directa

| Situación | Usar |
|-----------|------|
| Feature nuevo, scope claro | OpenSpec completo (explore + new + apply) |
| Bug fix simple, 1-2 archivos | Conversación directa |
| Refactor de módulo completo | OpenSpec — el scope acotado es crítico |
| Pregunta o explicación | Conversación directa |
| Feature que toca múltiples dominios | OpenSpec — dividir en varias specs |
| Typo, renombrar variable | Conversación directa |
| Integración con sistema externo | OpenSpec — contexto externo en la spec |

---

## Anti-patrones

```
❌ Spec con scope "agregar autenticación completa"
   → Demasiado amplio. Dividir: jwt-middleware, refresh-token, protected-routes

❌ Spec que dice "hacer que funcione mejor"
   → Sin definición de hecho. ¿Qué significa "mejor"?

❌ Usar /apply sin haber hecho /explore en código desconocido
   → Claude puede colisionar con patrones existentes

❌ Modificar el scope durante el apply porque "apareció algo"
   → Parar, actualizar la spec, nuevo apply

❌ No archivar las specs completadas
   → El historial se pierde, el directorio specs/ se llena de ruido
```
