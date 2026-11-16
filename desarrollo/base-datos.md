# Base de datos

PostgreSQL + PostGIS. Migraciones con Alembic.

## Tablas principales

| Tabla | Uso |
|---|---|
| `users`, `oauth_accounts`, `refresh_sessions` | Autenticación |
| `auth_audit_events`, `auth_throttle` | Auditoría y rate limit |
| `ephemeris_requests`, `ephemeris_products` | Solicitudes y productos |
| `rinex_files` | RINEX normalizados y metadatos |
| `postprocessing_jobs` | Jobs PPK/PPP y resultados |
| `image_geotag_batches` | Lotes de imágenes, MRK y geoetiquetas |
| `cors_stations` | Catálogo CORS/IGAC |
| `outbox_events` | Outbox transaccional |
| `delivery_packages` | ZIP y URLs firmadas |
| `worker_audit_events` | Auditoría de Celery |

## Geometrías

- `rinex_files.geometry` y `cors_stations.geometry`: `PointZ` SRID 4326.
- `postprocessing_jobs.trajectory`: `LineStringZ` SRID 4326.
- Índices GiST en geometrías.

## Migraciones

```bash
alembic revision --autogenerate -m "descripcion"
alembic upgrade head
alembic downgrade -1
```

Toda migración debe tener `downgrade`.
