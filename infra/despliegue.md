# Despliegue

## Requisitos

- Docker y Docker Compose.
- PostgreSQL/PostGIS accesible.
- RabbitMQ accesible.
- MinIO/S3 accesible.
- Variables en `geohub-backend/.env` (ver `.env.example`).
- La imagen incluye `rnx2rtkp`, `crx2rnx` y `exiftool` para el geotagging.

## Arranque local

```bash
cd geohub/geohub-backend
docker compose -f docker-compose.dev.yml up -d --build
```

Servicios:

- `migrate`: ejecuta `alembic upgrade head` y termina.
- `api`: Uvicorn en `0.0.0.0:8000`.
- `worker`: Celery con colas `geohub.*`.
- `beat`: tareas programadas.

## Verificación

```bash
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/health/ready
curl http://127.0.0.1:8000/health/queue
curl http://127.0.0.1:8000/metrics
```

## Migraciones

```bash
docker compose -f docker-compose.dev.yml run --rm api alembic upgrade head
```

## Dependencias del backend

El backend usa `uv.lock`. El Dockerfile instala con `uv sync --frozen --no-dev`,
así que una actualización de dependencias exige regenerar el lock:

```bash
cd geohub/geohub-backend
docker run --rm -v "$PWD":/app -w /app ghcr.io/astral-sh/uv:python3.12-bookworm uv lock
```

## Frontend

```bash
cd geohub/geohub-frontend
pnpm install
pnpm dev
```

En desarrollo, Vite proxya `/api` a `VITE_API_PROXY_TARGET`
(`http://127.0.0.1:8000`). Mantén `VITE_API_BASE_URL` vacío para cookies
same-origin.

## Producción

- HTTPS obligatorio.
- `COOKIE_SECURE=true`, `COOKIE_SAMESITE` según dominio.
- `TRUSTED_HOSTS` y `CORS_ORIGINS` restringidos.
- `SENTRY_DSN` y `METRICS_ENABLED` según operación.
- Backups y retención configurados.
- No exponer `/docs` y `/metrics` sin control de red.
