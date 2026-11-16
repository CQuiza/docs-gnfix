# Operación del catálogo

## Página de estaciones

`/stations` muestra el catálogo público con tabla y mapa:

- todos los puntos del catálogo;
- leyenda por red/estado;
- escala y norte;
- popup con código, nombre, red, estado y coordenadas.

## Actualización mensual

La tarea `maintenance.stations_refresh` se ejecuta el día 1 de cada mes a las
05:00 UTC. Actualiza:

- `external_id`;
- nombre, red y coordenadas;
- `is_active` según el estado del IGAC;
- `last_synced_at`.

Cola: `geohub.maintenance`.

## Actualización manual

Mientras se ejecuta el worker:

```bash
docker exec geohub-backend-worker-1 \
  celery -A geohub.workers.celery_app:celery_app call maintenance.stations_refresh
```

O el script:

```bash
docker compose -f docker-compose.dev.yml run --rm api \
  python -m geohub.scripts.import_cors_stations --igac
```

## Diagnóstico

- Verificar `last_synced_at` en `/stations`.
- Revisar `worker_audit_events` para `maintenance.stations_refresh`.
- Si el IGAC no responde, la tarea falla y se reintenta según la política de
  Celery; el catálogo anterior se conserva.
- El estado es el del catálogo IGAC (`Activa`, `Inactiva`, `Removida`), no la
  disponibilidad de RINEX del día.
