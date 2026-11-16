# Backend

Ver también:

- [`geohub-backend/README.md`](../../geohub-backend/README.md)
- [Arquitectura](arquitectura.md)
- [Base de datos](base-datos.md)

## Comandos

```bash
cd geohub/geohub-backend
ruff check .
ruff format --check .
pytest -q
```

Con base real:

```bash
docker run --rm --network host --env-file .env \
  -v "$PWD":/app -w /app geohub-backend:test pytest -q
```

## Módulos

- `auth`: usuarios, sesiones, cuotas, OAuth.
- `ephemeris`: fuentes IGS, productos, disponibilidad.
- `rinex`: subida, normalización, parser.
- `postprocessing`: PPK/PPP, RTKLIB, CTM12.
- `stations`: catálogo CORS, IGAC, navegación broadcast.
- `delivery`: ZIP, URLs firmadas, SMTP.
- `jobs`: auditoría de workers.
- `workers`: Celery, beat, tareas.
