# GORM — Convenciones

ORM oficial del backend Go. GORM se usa **exclusivamente en la capa repository**.
Los services nunca importan `gorm.io/gorm` — reciben y devuelven structs del
dominio (`model/`). La capa handler no sabe que GORM existe.

```
handler/ → service/ → repository/ (GORM aquí) → PostgreSQL
```

---

## Conexión y configuración

### Inicialización

```go
import (
  "gorm.io/driver/postgres"
  "gorm.io/gorm"
  "gorm.io/gorm/logger"
)

// NewDB crea la conexión a PostgreSQL con GORM.
// Recibe el DSN y el nivel de log. Retorna *gorm.DB o error.
//
// Ejemplo:
//   db, err := NewDB("host=localhost user=app dbname=myapp sslmode=disable", "info")
func NewDB(dsn string, logLevel string) (*gorm.DB, error) {
  lvl := logger.Info
  if logLevel == "silent" {
    lvl = logger.Silent
  }

  db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
    Logger:                 logger.Default.LogMode(lvl),
    SkipDefaultTransaction: true,  // performance: no tx implícita en cada query
    PrepareStmt:            true,  // reutiliza prepared statements
  })
  if err != nil {
    return nil, fmt.Errorf("gorm.Open: %w", err)
  }

  sqlDB, err := db.DB()
  if err != nil {
    return nil, fmt.Errorf("db.DB(): %w", err)
  }

  sqlDB.SetMaxIdleConns(10)
  sqlDB.SetMaxOpenConns(100)
  sqlDB.SetConnMaxLifetime(time.Hour)

  return db, nil
}
```

### Reglas de configuración

- **`SkipDefaultTransaction: true`** — GORM envuelve cada operación en una
  transacción por defecto. Desactivar para performance; usar transacciones
  explícitas cuando se necesiten.
- **`PrepareStmt: true`** — cachea prepared statements, mejora performance
  en queries repetidas.
- **Pool de conexiones** — siempre configurar `MaxIdleConns`, `MaxOpenConns`
  y `ConnMaxLifetime`. Los valores dependen del entorno (ajustar en `config.md`).
- **Logger level** — `Silent` en producción para performance, `Info` en
  desarrollo para debugging.

---

## Modelos

### Estructura base

```go
// User representa un usuario del sistema.
// Embebe gorm.Model para ID, CreatedAt, UpdatedAt, DeletedAt (soft delete).
type User struct {
  gorm.Model
  Name     string `gorm:"type:varchar(100);not null" json:"name"`
  Email    string `gorm:"type:varchar(255);uniqueIndex;not null" json:"email"`
  Role     string `gorm:"type:varchar(50);default:'user';not null" json:"role"`
  IsActive bool   `gorm:"default:true;not null" json:"is_active"`

  // Asociaciones
  Posts    []Post    `gorm:"foreignKey:AuthorID" json:"posts,omitempty"`
  Profile *Profile  `gorm:"foreignKey:UserID" json:"profile,omitempty"`
}
```

### Convenciones de modelos

- **Embeber `gorm.Model`** para obtener `ID`, `CreatedAt`, `UpdatedAt`,
  `DeletedAt` automáticos. Solo omitir si la tabla tiene PK compuesta o
  no necesita soft delete.
- **Struct tags siempre explícitos**: `gorm:"..."` para constraints de DB,
  `json:"..."` para serialización. No depender de convenciones implícitas.
- **Tipos de columna explícitos**: usar `type:varchar(100)` en vez de dejar
  que GORM infiera (evita sorpresas entre PostgreSQL y SQLite en tests).
- **Modelos en `internal/model/`** — un archivo por entidad o agrupar
  entidades relacionadas del mismo dominio.
- **Nombres de tabla**: GORM pluraliza automáticamente (`User` → `users`).
  Para nombres custom: implementar `TableName()`.

```go
// TableName define el nombre de tabla custom para auditoría.
func (AuditLog) TableName() string {
  return "audit_logs"
}
```

---

## Asociaciones

```go
// Post pertenece a un User (author) y tiene muchos Comments.
type Post struct {
  gorm.Model
  Title    string `gorm:"type:varchar(200);not null" json:"title"`
  Body     string `gorm:"type:text;not null" json:"body"`
  AuthorID uint   `gorm:"not null;index" json:"author_id"`

  // BelongsTo
  Author User `gorm:"foreignKey:AuthorID" json:"author,omitempty"`

  // HasMany
  Comments []Comment `gorm:"foreignKey:PostID" json:"comments,omitempty"`
}
```

### Reglas de asociaciones

- **Siempre especificar `foreignKey`** en el tag — no depender de la
  convención de nombre.
- **Índice en FK** — todo `foreignKey` lleva `index` en el campo ID.
- **Preload explícito** — nunca preload automático. Usar `Preload()` solo
  donde se necesite para evitar N+1.

```go
// GetUserWithPosts carga un usuario con sus posts.
// Usa Preload explícito para evitar N+1 queries.
func (r *UserRepository) GetUserWithPosts(ctx context.Context, id uint) (*model.User, error) {
  var user model.User
  err := r.db.WithContext(ctx).
    Preload("Posts", func(db *gorm.DB) *gorm.DB {
      return db.Order("created_at DESC").Limit(20)
    }).
    First(&user, id).Error

  if err != nil {
    return nil, fmt.Errorf("repository.GetUserWithPosts id=%d: %w", id, err)
  }
  return &user, nil
}
```

---

## Repository pattern

El repository encapsula todo acceso a GORM. Los services reciben la interfaz,
nunca la implementación concreta.

```go
// UserRepository define las operaciones de persistencia para User.
type UserRepository interface {
  Create(ctx context.Context, user *model.User) error
  GetByID(ctx context.Context, id uint) (*model.User, error)
  GetByEmail(ctx context.Context, email string) (*model.User, error)
  List(ctx context.Context, filter UserFilter) ([]model.User, int64, error)
  Update(ctx context.Context, user *model.User) error
  Delete(ctx context.Context, id uint) error
}

// userRepository implementa UserRepository con GORM.
type userRepository struct {
  db *gorm.DB
}

// NewUserRepository crea un nuevo UserRepository.
// Recibe *gorm.DB. Retorna UserRepository.
func NewUserRepository(db *gorm.DB) UserRepository {
  return &userRepository{db: db}
}
```

### Implementación tipo

```go
// Create persiste un nuevo usuario en la base de datos.
// Retorna error si el email ya existe (unique constraint).
func (r *userRepository) Create(ctx context.Context, user *model.User) error {
  if err := r.db.WithContext(ctx).Create(user).Error; err != nil {
    return fmt.Errorf("repository.CreateUser: %w", err)
  }
  return nil
}

// GetByID busca un usuario por ID.
// Retorna el usuario o error si no existe.
func (r *userRepository) GetByID(ctx context.Context, id uint) (*model.User, error) {
  var user model.User
  if err := r.db.WithContext(ctx).First(&user, id).Error; err != nil {
    return nil, fmt.Errorf("repository.GetByID id=%d: %w", id, err)
  }
  return &user, nil
}
```

### Reglas del repository

- **`WithContext(ctx)` siempre** — toda query lleva context para
  cancelación y timeouts.
- **Wrapping de errores** — `fmt.Errorf("repository.NombreMetodo: %w", err)`.
  Nunca retornar errores de GORM sin contexto.
- **Interfaz en el consumidor** — la interfaz `UserRepository` se define
  donde se consume (en `service/`), no donde se implementa. GORM es un
  detalle de implementación.
- **Un repository por entidad raíz** — no por tabla. Si `Profile` siempre
  se accede a través de `User`, va en `UserRepository`.

---

## Scopes (queries reutilizables)

Los scopes encapsulan filtros y condiciones reutilizables.

```go
// Paginate retorna un scope que aplica paginación.
// page empieza en 1, pageSize máximo 100.
func Paginate(page, pageSize int) func(db *gorm.DB) *gorm.DB {
  return func(db *gorm.DB) *gorm.DB {
    if page <= 0 {
      page = 1
    }
    if pageSize <= 0 || pageSize > 100 {
      pageSize = 20
    }
    offset := (page - 1) * pageSize
    return db.Offset(offset).Limit(pageSize)
  }
}

// ActiveOnly filtra solo registros activos (is_active = true).
func ActiveOnly(db *gorm.DB) *gorm.DB {
  return db.Where("is_active = ?", true)
}

// Uso en repository:
func (r *userRepository) List(ctx context.Context, f UserFilter) ([]model.User, int64, error) {
  var users []model.User
  var total int64

  query := r.db.WithContext(ctx).Model(&model.User{}).Scopes(ActiveOnly)

  if f.Role != "" {
    query = query.Where("role = ?", f.Role)
  }

  if err := query.Count(&total).Error; err != nil {
    return nil, 0, fmt.Errorf("repository.ListUsers count: %w", err)
  }

  err := query.Scopes(Paginate(f.Page, f.PageSize)).
    Order("created_at DESC").
    Find(&users).Error

  if err != nil {
    return nil, 0, fmt.Errorf("repository.ListUsers: %w", err)
  }

  return users, total, nil
}
```

### Reglas de scopes

- **Scopes genéricos** (paginación, soft delete, ordenamiento) van en
  `internal/repository/scopes.go`.
- **Scopes de dominio** van en el archivo del repository que los usa.
- **Composición** — los scopes se encadenan. Diseñar cada uno para hacer
  una sola cosa.

---

## Transacciones

```go
// TransferCredits transfiere créditos entre usuarios dentro de una transacción.
// Si cualquier operación falla, se hace rollback automático.
func (r *userRepository) TransferCredits(ctx context.Context, fromID, toID uint, amount int) error {
  return r.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    var from model.User
    if err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
      First(&from, fromID).Error; err != nil {
      return fmt.Errorf("lock sender id=%d: %w", fromID, err)
    }

    if from.Credits < amount {
      return apperror.New(apperror.BadRequest, "insufficient credits")
    }

    if err := tx.Model(&from).Update("credits", gorm.Expr("credits - ?", amount)).Error; err != nil {
      return fmt.Errorf("deduct sender: %w", err)
    }

    if err := tx.Model(&model.User{}).Where("id = ?", toID).
      Update("credits", gorm.Expr("credits + ?", amount)).Error; err != nil {
      return fmt.Errorf("credit receiver: %w", err)
    }

    return nil // commit
  })
}
```

### Reglas de transacciones

- **Usar `Transaction()`** con callback — GORM hace commit/rollback
  automáticamente según el error retornado.
- **`SkipDefaultTransaction: true`** en config global + transacciones
  explícitas donde se necesiten. Mejor que tx implícita en todo.
- **Locking** — usar `clause.Locking{Strength: "UPDATE"}` para evitar
  race conditions en operaciones de lectura-escritura.
- **No anidar transacciones** salvo que sea necesario. GORM soporta
  savepoints pero complican el flujo.

---

## Hooks

Los hooks ejecutan lógica antes/después de operaciones CRUD.

```go
// BeforeCreate genera UUID y valida el rol antes de insertar.
func (u *User) BeforeCreate(tx *gorm.DB) error {
  if u.UUID == "" {
    u.UUID = uuid.New().String()
  }
  if !isValidRole(u.Role) {
    return errors.New("invalid role")
  }
  return nil
}
```

### Reglas de hooks

- **Solo para lógica de persistencia** — generación de UUID, hashing,
  timestamps custom. La lógica de negocio va en el service.
- **Hooks disponibles**: `BeforeCreate`, `AfterCreate`, `BeforeUpdate`,
  `AfterUpdate`, `BeforeSave`, `AfterSave`, `BeforeDelete`, `AfterDelete`,
  `AfterFind`.
- **No hacer queries adicionales en hooks** a menos que sea estrictamente
  necesario — afecta performance y dificulta testing.
- **Documentar cada hook** — explicar qué hace y por qué no puede vivir
  en el service.

---

## Migraciones

### Desarrollo

`AutoMigrate` solo en desarrollo para prototipar rápido:

```go
// Solo en desarrollo — nunca en producción.
if cfg.Environment == "development" {
  db.AutoMigrate(&model.User{}, &model.Post{}, &model.Comment{})
}
```

### Producción

Usar archivos SQL de migración (`migrations/`) con herramientas como
`golang-migrate/migrate` o `pressly/goose`. **Nunca `AutoMigrate` en
producción** — no controla rollbacks, no versiona cambios, puede perder
datos.

```
migrations/
  000001_create_users.up.sql
  000001_create_users.down.sql
  000002_create_posts.up.sql
  000002_create_posts.down.sql
```

---

## Manejo de errores

```go
import "gorm.io/gorm"

// Verificar si un registro no existe.
if errors.Is(err, gorm.ErrRecordNotFound) {
  return nil, apperror.New(apperror.NotFound, "user not found")
}

// Verificar constraint violations (PostgreSQL).
var pgErr *pgconn.PgError
if errors.As(err, &pgErr) && pgErr.Code == "23505" {
  return apperror.New(apperror.Conflict, "email already exists")
}
```

### Reglas de errores

- **Traducir errores de GORM a `AppError`** en el repository — el service
  nunca recibe errores de GORM directamente.
- **`ErrRecordNotFound`** → `apperror.NotFound`.
- **Constraint violation `23505`** (unique) → `apperror.Conflict`.
- **Constraint violation `23503`** (FK) → `apperror.BadRequest` con mensaje
  descriptivo.
- **Todo lo demás** → wrapping con contexto y log en el service.

---

## Anti-patrones

| No hagas | Haz en cambio |
|----------|---------------|
| Usar `*gorm.DB` en el service | Definir interfaz de repository |
| `AutoMigrate` en producción | Archivos SQL versionados |
| Preload sin control (`Preload("*")`) | Preload explícito con condiciones |
| Ignorar errores de GORM | Traducir a `AppError` con contexto |
| Queries raw sin parametrizar | `db.Where("email = ?", email)` |
| Hooks con lógica de negocio | Hooks solo para persistencia, lógica en service |
| Crear `*gorm.DB` en cada request | Pool de conexiones compartido |
| Omitir `WithContext(ctx)` | Context en toda query |
| Depender de convenciones implícitas de nombre | Tags `gorm:"..."` explícitos |
| N+1 queries por lazy loading | Preload explícito donde se necesite |
