# Python — Convenciones

Python **no es parte del stack principal**. Se usa como lenguaje auxiliar para:

- **Microservicios especializados** — procesamiento geográfico (GeoPandas,
  Shapely, Rasterio), ML/data pipelines, integraciones con APIs externas.
- **Scripts de automatización** — ETL, migración de datos, tareas de consola.
- **Utilidades de infraestructura** — procesamiento batch, cron jobs,
  generadores de reportes.

El backend principal es Go. Python entra donde Go no tiene ecosistema
maduro o donde la productividad de Python justifica el trade-off.

---

## Reglas fundamentales

1. **Tipado estricto siempre.** Todo parámetro, retorno y variable relevante
   lleva type hint. Sin excepciones.
2. **Pydantic para datos.** Todo dato que entra o sale del servicio se valida
   con Pydantic `BaseModel`.
3. **Estructura clara.** Aunque sea un script pequeño, sigue la estructura
   definida abajo. No hay "scripts sueltos".
4. **Virtual environment.** Siempre `venv` o `uv`. Nunca instalar en el
   sistema global.
5. **Python 3.11+.** Usar las features modernas del lenguaje.

---

## Estructura de proyecto

### Microservicio

```
service-name/
├── src/
│   └── service_name/
│       ├── __init__.py
│       ├── main.py            → entrypoint (FastAPI app o CLI)
│       ├── config.py          → settings con Pydantic BaseSettings
│       ├── models/            → Pydantic models (entrada/salida)
│       │   ├── __init__.py
│       │   └── geo.py
│       ├── services/          → lógica de negocio
│       │   ├── __init__.py
│       │   └── processor.py
│       ├── routes/            → endpoints HTTP (si aplica)
│       │   ├── __init__.py
│       │   └── health.py
│       └── utils/             → funciones auxiliares puras
│           ├── __init__.py
│           └── coordinates.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   └── test_processor.py
├── Dockerfile
├── pyproject.toml
└── README.md
```

### Script / utilidad

```
scripts/
└── migrate_data/
    ├── __init__.py
    ├── main.py               → entrypoint con argparse o typer
    ├── models.py             → Pydantic models
    ├── processor.py          → lógica
    ├── pyproject.toml
    └── README.md
```

---

## Tipado

### Reglas de tipado

```python
from typing import Any  # Solo cuando es genuinamente Any — casi nunca.

# Bien: todo tipado, explícito
def calculate_distance(
    origin: tuple[float, float],
    destination: tuple[float, float],
    unit: str = "km",
) -> float:
    """Calcula la distancia entre dos coordenadas geográficas.

    Args:
        origin: Tupla (latitud, longitud) del punto de origen.
        destination: Tupla (latitud, longitud) del punto de destino.
        unit: Unidad de medida ("km" o "mi"). Default: "km".

    Returns:
        Distancia calculada en la unidad especificada.

    Example:
        >>> calculate_distance((23.1136, -82.3666), (40.7128, -74.0060))
        2089.42
    """
    ...


# Mal: sin tipos, sin docs
def calculate_distance(origin, destination, unit="km"):
    ...
```

### Convenciones de tipos

- **`list[str]`** en vez de `List[str]` — usar tipos built-in (Python 3.9+).
- **`X | None`** en vez de `Optional[X]` — union syntax (Python 3.10+).
- **`dict[str, Any]`** solo cuando el dict es genuinamente abierto.
  Preferir `TypedDict` o Pydantic model.
- **Generics con `type`** para funciones genéricas:

```python
from typing import TypeVar

T = TypeVar("T")

def first_or_none(items: list[T]) -> T | None:
    """Retorna el primer elemento de la lista o None si está vacía."""
    return items[0] if items else None
```

---

## Pydantic — modelos y validación

### Modelos de datos

```python
from pydantic import BaseModel, Field, field_validator


class GeoPoint(BaseModel):
    """Representa un punto geográfico con coordenadas WGS84.

    Attributes:
        lat: Latitud en grados decimales (-90 a 90).
        lon: Longitud en grados decimales (-180 a 180).
        label: Etiqueta descriptiva opcional del punto.
    """

    lat: float = Field(..., ge=-90, le=90, description="Latitud WGS84")
    lon: float = Field(..., ge=-180, le=180, description="Longitud WGS84")
    label: str | None = Field(default=None, max_length=100)


class RouteRequest(BaseModel):
    """Request para calcular una ruta entre puntos.

    Attributes:
        origin: Punto de origen de la ruta.
        destination: Punto de destino de la ruta.
        waypoints: Puntos intermedios opcionales (máx. 25).
        avoid_tolls: Si debe evitar peajes.
    """

    origin: GeoPoint
    destination: GeoPoint
    waypoints: list[GeoPoint] = Field(default_factory=list, max_length=25)
    avoid_tolls: bool = False

    @field_validator("waypoints")
    @classmethod
    def validate_waypoints_not_duplicate_origin(
        cls, v: list[GeoPoint], info
    ) -> list[GeoPoint]:
        """Valida que los waypoints no repitan el origen."""
        # Validación custom cuando Field constraints no alcanzan.
        return v
```

### Configuración con BaseSettings

```python
from pydantic_settings import BaseSettings
from pydantic import Field


class Settings(BaseSettings):
    """Configuración del microservicio cargada desde variables de entorno.

    Attributes:
        app_name: Nombre del servicio para logging.
        port: Puerto HTTP del servicio.
        database_url: URL de conexión a PostgreSQL.
        debug: Modo debug (nunca True en producción).
        allowed_origins: Orígenes CORS permitidos.
    """

    app_name: str = "geo-service"
    port: int = Field(default=8080, ge=1, le=65535)
    database_url: str
    debug: bool = False
    allowed_origins: list[str] = Field(default_factory=lambda: ["http://localhost:3000"])

    model_config = {
        "env_prefix": "GEO_",       # GEO_PORT, GEO_DATABASE_URL, etc.
        "env_file": ".env",
        "env_file_encoding": "utf-8",
    }


# Singleton — instanciar una vez, importar donde se necesite.
settings = Settings()
```

### Reglas de Pydantic

- **`BaseModel` para datos**, `BaseSettings` para configuración.
- **`Field(...)` explícito** con constraints (`ge`, `le`, `max_length`,
  `pattern`) en vez de validadores custom cuando sea posible.
- **`field_validator`** para validaciones que `Field` no puede expresar.
- **`model_validator`** para validaciones que dependen de múltiples campos.
- **Inmutabilidad por defecto**: usar `model_config = {"frozen": True}`
  para modelos que no deben mutarse después de crearse.

---

## Estructura del código

### Services (lógica de negocio)

```python
import logging
from dataclasses import dataclass

from .models.geo import GeoPoint, RouteRequest, RouteResponse

logger = logging.getLogger(__name__)


@dataclass
class GeoService:
    """Servicio de procesamiento geográfico.

    Encapsula la lógica de cálculo de rutas y distancias.
    No conoce HTTP ni frameworks — solo recibe y devuelve modelos.

    Attributes:
        api_key: API key para el proveedor de mapas.
    """

    api_key: str

    def calculate_route(self, request: RouteRequest) -> RouteResponse:
        """Calcula la ruta óptima entre origen y destino.

        Args:
            request: Datos de la ruta a calcular.

        Returns:
            RouteResponse con distancia, duración y geometría.

        Raises:
            GeoServiceError: Si el proveedor de mapas falla.
        """
        logger.info(
            "Calculating route",
            extra={"origin": f"{request.origin.lat},{request.origin.lon}"},
        )
        ...
```

### Manejo de errores

```python
class ServiceError(Exception):
    """Error base para errores del servicio.

    Attributes:
        message: Descripción del error.
        code: Código de error para el cliente.
        status_code: HTTP status code correspondiente.
    """

    def __init__(
        self,
        message: str,
        code: str = "INTERNAL_ERROR",
        status_code: int = 500,
    ) -> None:
        self.message = message
        self.code = code
        self.status_code = status_code
        super().__init__(message)


class NotFoundError(ServiceError):
    """Recurso no encontrado."""

    def __init__(self, message: str = "Resource not found") -> None:
        super().__init__(message, code="NOT_FOUND", status_code=404)


class ValidationError(ServiceError):
    """Error de validación de datos de entrada."""

    def __init__(self, message: str = "Validation failed") -> None:
        super().__init__(message, code="VALIDATION_ERROR", status_code=422)
```

---

## HTTP con FastAPI (si aplica)

Si el microservicio expone una API HTTP, usar **FastAPI** — tipado nativo,
Pydantic integrado, async, documentación automática.

```python
from fastapi import FastAPI, HTTPException
from contextlib import asynccontextmanager

from .config import settings
from .models.geo import RouteRequest, RouteResponse
from .services.geo import GeoService


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Inicializa y limpia recursos del servicio."""
    app.state.geo_service = GeoService(api_key=settings.maps_api_key)
    yield
    # Cleanup si es necesario


app = FastAPI(
    title=settings.app_name,
    lifespan=lifespan,
)


@app.post("/routes", response_model=RouteResponse)
async def calculate_route(request: RouteRequest) -> RouteResponse:
    """Calcula la ruta óptima entre dos puntos geográficos.

    Args:
        request: Origen, destino y opciones de la ruta.

    Returns:
        RouteResponse con distancia, duración y geometría.
    """
    try:
        return app.state.geo_service.calculate_route(request)
    except ServiceError as e:
        raise HTTPException(status_code=e.status_code, detail=e.message)


@app.get("/health")
async def health() -> dict[str, str]:
    """Health check del servicio."""
    return {"status": "ok"}
```

---

## CLI con Typer (si aplica)

Para scripts y utilidades de consola, usar **Typer** — tipado, auto-help,
colores.

```python
import typer
from pathlib import Path

app = typer.Typer(help="Herramientas de procesamiento geográfico.")


@app.command()
def convert(
    input_file: Path = typer.Argument(..., help="Archivo de entrada (GeoJSON/Shapefile)"),
    output_file: Path = typer.Argument(..., help="Archivo de salida"),
    format: str = typer.Option("geojson", "--format", "-f", help="Formato de salida"),
    verbose: bool = typer.Option(False, "--verbose", "-v", help="Output detallado"),
) -> None:
    """Convierte archivos geográficos entre formatos.

    Example:
        $ python -m geo_tools convert input.shp output.geojson
    """
    if verbose:
        typer.echo(f"Converting {input_file} → {output_file} ({format})")
    ...


if __name__ == "__main__":
    app()
```

---

## Testing

```python
import pytest
from .models.geo import GeoPoint, RouteRequest
from .services.geo import GeoService


class TestGeoService:
    """Tests para GeoService."""

    @pytest.fixture
    def service(self) -> GeoService:
        """Crea instancia del servicio para testing."""
        return GeoService(api_key="test-key")

    @pytest.fixture
    def havana(self) -> GeoPoint:
        """Coordenadas de La Habana."""
        return GeoPoint(lat=23.1136, lon=-82.3666, label="Havana")

    def test_calculate_distance_returns_positive(
        self, service: GeoService, havana: GeoPoint
    ) -> None:
        """La distancia entre dos puntos siempre es positiva."""
        nyc = GeoPoint(lat=40.7128, lon=-74.0060, label="NYC")
        result = service.calculate_distance(havana, nyc)
        assert result > 0

    def test_geo_point_validates_latitude_range(self) -> None:
        """GeoPoint rechaza latitudes fuera del rango -90 a 90."""
        with pytest.raises(ValueError):
            GeoPoint(lat=91.0, lon=0.0)
```

### Reglas de testing

- **pytest** como framework — nunca `unittest`.
- **Fixtures** para setup reutilizable.
- **Nombre**: `test_<qué>_<condición>` — describe el comportamiento, no
  la implementación.
- **Pydantic models en tests** — validar que los modelos rechazan datos
  inválidos. Son parte del contrato.

---

## Tooling

| Herramienta | Propósito |
|-------------|-----------|
| **uv** | Package manager (rápido, reemplaza pip+venv) |
| **ruff** | Linter + formatter (reemplaza flake8+black+isort) |
| **mypy** | Type checker estático (`--strict`) |
| **pytest** | Testing framework |
| **pyproject.toml** | Config única (no `setup.py`, no `requirements.txt`) |

### pyproject.toml mínimo

```toml
[project]
name = "geo-service"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "pydantic>=2.0",
    "pydantic-settings>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "ruff>=0.4",
    "mypy>=1.10",
]

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "ANN", "B", "SIM", "TCH"]

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
```

---

## Docker

El microservicio Python se despliega en un container independiente del
backend Go. Multi-stage build para imagen mínima.

```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install --no-cache-dir .

FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY src/ ./src/
CMD ["python", "-m", "service_name.main"]
```

---

## Integración con el stack Go

El microservicio Python se comunica con el backend Go vía:

- **HTTP REST** — el servicio Python expone endpoints que Go consume
  como cliente HTTP. Seguir el formato de API definido en `api.md`.
- **Cola de mensajes** — para tareas async (procesamiento pesado de
  datos geográficos, ETL). Redis, RabbitMQ o similar.
- **CLI directo** — Go ejecuta el script Python como subproceso para
  tareas one-shot (migración de datos, generación de reportes).

El microservicio Python **nunca accede directamente** a la base de datos
del backend Go. Si necesita datos, los pide vía API.

---

## Anti-patrones

| No hagas | Haz en cambio |
|----------|---------------|
| Código sin type hints | Tipar todo: parámetros, retorno, variables relevantes |
| `dict` sueltos para datos | Pydantic `BaseModel` con validación |
| `requirements.txt` | `pyproject.toml` con `uv` |
| `print()` para debugging | `logging` con niveles |
| Variables de entorno con `os.getenv` | Pydantic `BaseSettings` |
| Scripts sin estructura | Estructura de proyecto definida arriba |
| `Any` como tipo default | Tipos concretos; `Any` solo si es genuinamente cualquier cosa |
| Instalar paquetes globalmente | `venv` o `uv` siempre |
| `unittest.TestCase` | `pytest` con fixtures |
| Acceso directo a la DB de Go | Comunicación vía API REST |
