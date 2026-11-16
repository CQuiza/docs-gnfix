# Pruebas

## Backend

```bash
cd geohub/geohub-backend
ruff check .
ruff format --check .
pytest -q
```

Integración con PostgreSQL real:

```bash
docker run --rm --network host --env-file .env \
  -v "$PWD":/app -w /app geohub-backend:test pytest -q
```

## Frontend

```bash
cd geohub/geohub-frontend
pnpm lint
pnpm build
pnpm test
```

## E2E

```bash
cd geohub/geohub-frontend
docker run --rm --network host -v "$PWD":/work -w /work \
  -e E2E_API_BASE_URL=http://127.0.0.1:8000 \
  mcr.microsoft.com/playwright:v1.65.0-jammy npx playwright test
```

## Cobertura actual

- Backend: 666 tests, 82 %.
- Frontend: 49 tests.
- E2E: 11.

## Auditoría de dependencias

```bash
# Backend (usa uv.lock y pip-audit)
cd geohub/geohub-backend
./scripts/security-audit.sh

# Frontend
cd geohub/geohub-frontend
pnpm audit
```
