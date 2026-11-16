# Backups

La tarea `maintenance.backup` ejecuta `pg_dump`, sube el archivo a MinIO/S3 y
aplica retención.

## Configuración

```env
BACKUP_ENABLED=false
BACKUP_STORAGE_PREFIX=backups/postgres
BACKUP_RETENTION_DAYS=14
```

Cuando `BACKUP_ENABLED=true`, beat la programa a diario a las 03:00 UTC.

## Verificación

```bash
docker logs --tail 100 geohub-backend-worker-1 | grep maintenance.backup
```

El objeto queda en:

```text
s3://<bucket>/backups/postgres/<archivo>.dump
```

## Restauración

1. Descargar el `.dump` desde MinIO/S3.
2. Restaurar con `pg_restore` sobre una base vacía.
3. Validar `alembic current` y `/health/ready`.

Para el procedimiento completo, ver el
[runbook](runbook.md).
