# Observabilidad

## Logs

- API y workers escriben logs estructurados si `LOG_FORMAT=json`.
- Cada request tiene `X-Request-ID`; los workers propagan el identificador.
- Celery registra `received`, `task_succeeded` y `task_failed`.
- Docker rota logs a 10 MB × 3.

## Sentry

```env
SENTRY_DSN=
SENTRY_ENVIRONMENT=
SENTRY_RELEASE=
SENTRY_TRACES_SAMPLE_RATE=0
```

El scrubbing elimina cookies, tokens y datos personales.

## Prometheus

`GET /metrics` publica métricas HTTP, colas y tareas. Restringir el acceso por
red o proxy.

## Auditoría

- `auth_audit_events`: acciones de autenticación y administración.
- `worker_audit_events`: ejecuciones de Celery que procesan recursos o fallan.
- Retención de auditoría de workers: `WORKER_AUDIT_RETENTION_DAYS` (30 días).

Ver [runbook](runbook.md) para diagnóstico.
