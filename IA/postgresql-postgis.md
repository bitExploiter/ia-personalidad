# PostgreSQL / PostGIS — Convenciones

---

## Convenciones generales

- **Tablas**: `snake_case`, plural (`users`, `work_orders`, `flight_plans`).
- **Columnas**: `snake_case`. PKs siempre `id` (UUID o BIGSERIAL según
  contexto). FKs como `<entidad_singular>_id` (`user_id`, `project_id`).
- **Timestamps**: toda tabla lleva `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
  y `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`. Usar trigger o
  aplicación para actualizar `updated_at`.
- **Soft delete** solo si el negocio lo requiere. Si se usa, agregar
  `deleted_at TIMESTAMPTZ` (nullable). Nunca borrar datos de producción
  sin pensarlo.
- **Índices con nombre explícito**: `idx_<tabla>_<columnas>`.
  Ej: `idx_users_email`, `idx_work_orders_project_id_status`.
- **Constraints con nombre**: `fk_<tabla>_<tabla_ref>`, `uq_<tabla>_<columna>`,
  `chk_<tabla>_<condición>`.
- **Enums**: preferir `TEXT` con CHECK constraint sobre tipos ENUM nativos
  (los ENUM de Postgres son difíciles de modificar en producción).
  ```sql
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'active', 'completed', 'cancelled'))
  ```
- **No queries inline en el código.** Centralizar en la capa de repository.
  En Go: queries como constantes o archivos `.sql`.
- **Migraciones numeradas y con rollback**: toda migración `up` tiene su
  `down` funcional. Naming: `NNNN_descripcion.up.sql` / `NNNN_descripcion.down.sql`.
- **Conexión**: usar pool de conexiones siempre (`pgxpool` en Go). Nunca
  una conexión por request.

---

## PostGIS

- **Extensión**: habilitar con `CREATE EXTENSION IF NOT EXISTS postgis;` en la
  primera migración.
- **Tipo de columna**: `GEOMETRY(tipo, SRID)` con SRID explícito siempre.
  Default SRID: `4326` (WGS84) para coordenadas geográficas.
  ```sql
  location GEOMETRY(Point, 4326) NOT NULL
  ```
- **Índice espacial obligatorio** en toda columna geométrica:
  ```sql
  CREATE INDEX idx_sites_location ON sites USING GIST (location);
  ```
- **Inserción**: usar `ST_SetSRID(ST_MakePoint(lon, lat), 4326)`. Atención:
  PostGIS usa orden `(longitud, latitud)`, no `(lat, lon)`.
- **Queries espaciales comunes**:
  - Distancia: `ST_DStWithin(geom, ref, distancia_metros)` (usa índice,
    más eficiente que `ST_Distance < X`).
  - Dentro de área: `ST_Within(punto, polígono)`.
  - Transformar a metros para cálculos: `ST_Transform(geom, 3857)` o usar
    `::geography` para distancias reales.
- **GeoJSON**: `ST_AsGeoJSON(geom)` para serializar al API. Para input,
  `ST_GeomFromGeoJSON(json)`.
- **Performance**: para queries con muchos puntos, considerar
  `ST_ClusterDBSCAN` o `ST_Subdivide` para polígonos grandes.
