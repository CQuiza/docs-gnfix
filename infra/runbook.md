# GNFIX — Runbook operativo

## Documentación

- Manual de usuario: [`../usuario/manual-usuario.md`](../usuario/manual-usuario.md) y `/manual`.
- README del backend: [`../../geohub-backend/README.md`](../../geohub-backend/README.md).
- README del frontend: [`../../geohub-frontend/README.md`](../../geohub-frontend/README.md).
- OpenAPI interactivo: `http://127.0.0.1:8000/docs`.

## Servicios

| Servicio | Puerto | Notas |
|---|---|---|
| API FastAPI | `8000` | Uvicorn, cookies de sesión y API REST. |
| Worker Celery | — | Colas `geohub.ephemeris`, `geohub.postprocessing`, `geohub.delivery`, `geohub.maintenance`. |
| Beat Celery | — | Outbox, sondeo de efemérides, limpieza y backups. |
| PostgreSQL/PostGIS | `5434` | Externo al compose. |
| RabbitMQ | `5672` | Externo al compose. |
| MinIO | `9000` | Externo al compose. |
| Frontend Vite | `5173` | Proxy `/api` hacia la API en desarrollo. |

## Arranque y actualización

```bash
cd geohub/geohub-backend
docker compose -f docker-compose.dev.yml up -d --build
```

El servicio `migrate` aplica `alembic upgrade head` antes de arrancar API,
worker y beat. Para actualizar solo un servicio:

```bash
docker compose -f docker-compose.dev.yml up -d --build api
```

Frontend:

```bash
cd geohub/geohub-frontend
cp .env.example .env
pnpm install
pnpm dev
```

Abrir `http://localhost:5173`. El `.env` de desarrollo usa
`VITE_API_BASE_URL=` vacío y `VITE_API_PROXY_TARGET=http://127.0.0.1:8000`;
así las cookies son del mismo origen y funcionan con `SameSite=Lax`.

## Health checks

```bash
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/health/ready
curl http://127.0.0.1:8000/health/queue
curl http://127.0.0.1:8000/metrics
```

- `/health/ready`: PostgreSQL y PostGIS.
- `/health/queue`: conectividad TCP con RabbitMQ.
- `/metrics`: métricas Prometheus; restringir por red/proxy en producción.

## Logs

```bash
docker logs -f geohub-backend-api-1
docker logs -f geohub-backend-worker-1
docker logs -f geohub-backend-beat-1
```

Cada request incluye `X-Request-ID` y los logs llevan `request_id`. Las tareas
Celery usan el `task_id` como identificador de correlación.

## Migraciones

```bash
docker compose -f docker-compose.dev.yml run --rm migrate
docker run --rm --network host --env-file .env geohub-backend:phase11 alembic current
```

## Celery

Colas y tareas:

| Tarea | Cola | Frecuencia |
|---|---|---|
| `ephemeris.resolve` | `geohub.ephemeris` | Outbox / sondeo. |
| `ephemeris.poll_waiting` | `geohub.maintenance` | 15 min. |
| `postprocessing.run` | `geohub.postprocessing` | Outbox. |
| `delivery.package` | `geohub.delivery` | Outbox. |
| `outbox.dispatch` | `geohub.maintenance` | 30 s. |
| `delivery.cleanup` | `geohub.maintenance` | 1 h. |
| `maintenance.audit_cleanup` | `geohub.maintenance` | Diario 04:00 UTC. |
| `maintenance.backup` | `geohub.maintenance` | Diario 03:00 UTC si `BACKUP_ENABLED=true`. |
| `maintenance.stations_refresh` | `geohub.maintenance` | Día 1 de cada mes, 05:00 UTC. |
| `maintenance.geotagging_cleanup` | `geohub.maintenance` | Diario 04:30 UTC. |
| `geotagging.run` | `geohub.postprocessing` | Bajo demanda al crear un lote. |

Los lotes de geotagging se eliminan tras `GEOTAGGING_RETENTION_HOURS` (168 h por
defecto). El ZIP de imágenes originales se borra al terminar el lote si
`GEOTAGGING_DELETE_INPUT_IMAGES=true`.

Auditoría de ejecuciones en `worker_audit_events`.

- `outbox.dispatch` y `ephemeris.poll_waiting` solo dejan registro cuando
  publican/encolan algo o cuando fallan. Los no-op no generan filas.
- `WORKER_AUDIT_RETENTION_DAYS` (30 por defecto) controla la purga diaria.
- Los logs de Celery muestran `received` y `task_succeeded`/`task_failed`
  (este último añadido por el runtime de GNFIX).

## Almacenamiento de objetos

- `MINIO_ENDPOINT` es el endpoint interno que usan API/worker para subir,
  descargar y borrar (`minio:9000` en `proxy_network`).
- `MINIO_PUBLIC_ENDPOINT` es el hostname público que se usa **solo para firmar**
  las URLs de descarga (`s3.cristhianquiza.com`, sin `https://`).
- `MINIO_PUBLIC_SECURE=true` cuando el endpoint público va por HTTPS.
- Si `MINIO_PUBLIC_ENDPOINT` está vacío se usa el interno (dev/túnel).
- El endpoint público debe apuntar al API S3, no a la consola, y permitir solo
  `GET`/`HEAD` en el proxy.
- Prueba las URLs prefirmadas con `GET`, nunca con `curl -I`: la firma incluye
  el método HTTP y un `HEAD` sobre una URL firmada para `GET` devuelve
  `AccessDenied`.
- Si un `GET` prefirmado falla, comprueba que el proxy conserva la query string
  (`proxy_pass` sin `$uri`) y que reenvía el `Host` original
  (`proxy_set_header Host $host`); ambos forman parte de la firma.

## Observabilidad

### Proyecto Sentry

- Organización: `cristhianquiza` (región US).
- Proyecto: `gnfix` — <https://cristhianquiza.sentry.io/projects/gnfix/>.
- Release local: `gnfix@0.1.0`.
- DSN: en `.env` del backend (`SENTRY_DSN`) y del frontend
  (`VITE_SENTRY_DSN`). No se versiona el DSN real; `.env.example` queda vacío.
- Para subir source maps en CI: `SENTRY_AUTH_TOKEN`, `SENTRY_ORG=cristhianquiza`
  y `SENTRY_PROJECT=gnfix`.
- El evento `GNFIX Sentry setup check` (issue `GNFIX-1`) confirma la ingesta;
  se puede resolver desde Sentry.

- `SENTRY_DSN` habilita Sentry en API, workers y beat.
- `SENTRY_ENVIRONMENT` y `SENTRY_RELEASE` deben ser iguales entre backend y
  frontend para correlacionar releases.
- El frontend solo inicializa Sentry si existe `VITE_SENTRY_DSN`.
- Se eliminan cuerpos, cookies, cabeceras, query strings, correos y tokens.
- `METRICS_ENABLED=true` publica `/metrics`; `prometheus-client` mantiene
  contadores por método, ruta y estado.
- `API_RATE_LIMIT` y `API_RATE_WINDOW_SECONDS` aplican un límite en memoria
  por proceso. Con varias réplicas de API hay que sustituirlo por un limitador
  compartido (Redis) o multiplicar el límite por réplica. Los límites de
  login/registro/refresh sí viven en la base de datos y son globales.
  En varios procesos o réplicas, sustituir por un limitador compartido
  (Redis) para que sea global.

## Backups

El worker puede ejecutar un `pg_dump` comprimido y subirlo a
`BACKUP_STORAGE_PREFIX` en MinIO/S3:

```env
BACKUP_ENABLED=true
BACKUP_STORAGE_PREFIX=backups/postgres
BACKUP_RETENTION_DAYS=14
```

La tarea `maintenance.backup`:

1. ejecuta `pg_dump --format=custom --compress=6`;
2. sube el `.dump` a MinIO;
3. elimina dumps más antiguos que la retención;
4. registra estado, duración y error en `worker_audit_events`;
5. elimina el temporal siempre.

Ejecución manual desde un contenedor:

```bash
docker run --rm --network host --env-file .env \
  -v "$PWD":/app -w /app geohub-backend:phase11 \
  python -c "import asyncio; from geohub.bootstrap.container import build_container; from geohub.bootstrap.settings import get_settings; c=build_container(get_settings()); print(asyncio.run(c.postgres_backup.execute()))"
```

Restauración (probar en una base vacía):

```bash
pg_restore --clean --if-exists --no-owner --no-acl \
  -h 127.0.0.1 -p 5434 -U "$DB_USER" -d geohub_restore geohub-YYYYMMDDTHHMMSSZ.dump
```

## E2E

Requisitos:

- API de E2E en `8001` con una base aislada.
- Frontend Vite en `5173` con proxy a `8001`.
- Playwright en contenedor para evitar dependencias del navegador.

```bash
cd geohub/geohub-frontend
VITE_API_PROXY_TARGET=http://127.0.0.1:8001 pnpm dev -- --host 127.0.0.1 --port 5173
docker run --rm --network host -v "$PWD":/work -w /work \
  -e E2E_BASE_URL=http://localhost:5173 \
  mcr.microsoft.com/playwright:v1.65.0-jammy npx playwright test
```

Los tests cubren login, modo invitado, navegación y creación de efemérides.
Usar una base de datos desechable; los tests crean solicitudes.

## Fallos comunes

### MinIO no responde

- El mapa y las subidas fallan con `object_storage_unavailable` o error 503.
- Verificar túnel y health: `curl http://127.0.0.1:9000/minio/health/live`.
- `/health/ready` no comprueba MinIO; revisar logs y la ruta de storage.

### RabbitMQ no responde

- `/health/queue` devuelve 503.
- Worker y beat no consumen tareas.
- Verificar túnel `5672` y credenciales en `RABBITMQ_URL`.

### 401/403 desde el frontend

- En desarrollo, usar el proxy de Vite y `VITE_API_BASE_URL` vacío.
- Si frontend y API están en orígenes distintos, configurar
  `COOKIE_DOMAIN`, `COOKIE_SAMESITE=none`, `COOKIE_SECURE=true` y el origen
  correcto en `CORS_ORIGINS`.
- Un 403 `auth.csrf_failed` indica cookie CSRF ausente o desincronizada.

### Ruido de `GET /v1/models` en los logs

Si ves muchas líneas `GET /v1/models 404`, **no vienen de GNFIX**. Es un
cliente compatible con OpenAI que busca un servidor de modelos en
`127.0.0.1:8000`. En este entorno se identificó como **OpenCode CLI**
(`User-Agent: opencode/latest/...`).

Opciones:

- cambiar la URL base del proveedor/modelo en OpenCode para que no apunte a
  GNFIX;
- desactivar la comprobación de modelos locales si no se usa;
- mover GNFIX a otro puerto (por ejemplo `8010`) y actualizar
  `VITE_API_PROXY_TARGET`, `.env` y compose.

No añadir una ruta falsa `/v1/models`: GNFIX no es un servidor de modelos y
ocultar el error solo confunde.

### RTKLIB falla

- Revisar `processing_jobs.error` y `worker_audit_events`.
- Verificar que rover y base correspondan a la misma ventana temporal.
- Para CORS automática, el rover debe tener `APPROX POSITION XYZ`.
- Si el error es `error : no nav data`, la estación no devolvió efemérides.
  El adaptador del IGAC descarga la navegación mixta (`_MN.rnx.gz`) junto con la
  observación; si el día no tiene archivos, el job falla antes de RTKLIB.
- `CORS_MAX_DISTANCE_KM` es 20 km por defecto para que la base comparta
  condiciones atmosféricas con el rover. Si una zona no tiene estación dentro de
  ese radio, el usuario debe subir un RINEX base propio.

### Modos PPK/PPP y efemérides

- Modos: `ppk_broadcast`, `ppk_precise`, `ppk_static`, `ppp_static`,
  `ppp_kinematic`.
- PPK admite base manual o `auto_cors`; PPP no admite ninguna de las dos.
- `ppk_static` es el modo **Static** de RTKLIB (`-p 3`): usa base y
  `out-solstatic=single`, por lo que entrega una coordenada consolidada. Si no
  hay productos precisos, cae a static broadcast. Usa el timeout extendido de
  `PPP_PROCESSING_TIMEOUT_SECONDS`.
- `ppk_broadcast` usa navegación de broadcast. La base manual también necesita
  navegación: el backend la busca en IGAC con
  `CORS_NAVIGATION_MAX_DISTANCE_KM`.
- `ppk_precise` y PPP usan SP3/CLK/ERP. Si no hay productos precisos:
  - PPK preciso cae a broadcast con `fallback_reason` y `warning`;
  - PPP falla porque no existe un PPP broadcast equivalente.
- El usuario puede pedir `final`, `rapid`, `ultra_rapid` o automático. Si el
  producto pedido no está disponible, se usa el mejor disponible.
- Para fechas históricas se prueban nombres cortos IGS
  (`igsWWWWD.sp3.Z`, `igs20567.erp.Z`) después de los nombres largos. Esto cubre,
  por ejemplo, GEOS 2019.
- PPP usa `PPP_PROCESSING_TIMEOUT_SECONDS` (900 s) y advierte si la observación
  dura menos de `PPP_MIN_DURATION_MINUTES` (60 min).
- RTKLIB se ejecuta con argumentos CLI (`-p`, `-f`, `-sys`, `-r`) y un `.conf`
  mínimo para `sateph`/`ionoopt`/`tropopt`. No volver al `.conf` completo: el
  binario Debian pierde la base en modo relativo.
- `PosParser` acepta coma y espacio; si se cambia el separador de salida de
  RTKLIB, no debería romper el reporte.

### Catálogo CORS del IGAC

- Recargar el catálogo:

  ```bash
  docker compose -f docker-compose.dev.yml run --rm api \
    python -m geohub.scripts.import_cors_stations --igac
  ```

- El importador hace upsert por `code` y guarda `external_id` (`t_id` del IGAC).
  No hay que borrar la tabla antes de recargar.
- El listado de archivos por estación se cachea
  `IGAC_LISTING_CACHE_SECONDS` (300 s). Si un archivo recién publicado no
  aparece, reiniciar worker o bajar el TTL.
- Para diagnosticar una estación concreta:

  ```bash
  curl -s 'https://ccg.igac.gov.co/api/rinex?id=<t_id>' | head -c 500
  ```

- `POST /api/rinex` devuelve `200` con un ZIP vacío cuando no hay archivos para
  ese día. El adaptador lo interpreta como “sin observación”, no como éxito.


## Seguridad operativa

- No versionar `.env`, DSNs, tokens de CI ni credenciales.
- Restringir `/metrics` y `/docs` por red o proxy.
- Configurar `TRUSTED_HOSTS` en producción.
- Mantener `COOKIE_SECURE=true` y HTTPS en producción.
- Revisar `pip check`, `pip-audit`, `pnpm audit` y actualizar dependencias.
- Al cierre de la Fase 11: `pip-audit` limpio y `pnpm audit --prod` sin
  vulnerabilidades. El árbol de desarrollo mantiene hallazgos moderados en la
  cadena de ESLint/minimatch; no afectan al bundle de producción.

### Verificación de correo y cuota

- `EMAIL_VERIFICATION_REQUIRED=true` (por defecto): una cuenta sin verificar
  puede iniciar sesión, pero no subir RINEX, crear solicitudes de efemérides,
  lanzar postprocesos ni usar geotagging. Recibe
  `403 auth.email_not_verified`.
- El registro envía un enlace a `{FRONTEND_URL}/verify-email?token=...`;
  requiere SMTP configurado. Sin correo real no hay verificación y la cuenta
  queda limitada a lectura.
- Los invitados conservan su cuota diaria por IP; las cuentas verificadas no
  tienen cuota. Una cuenta falsa ya no obtiene cuota ilimitada.
- Activar Cloudflare Turnstile en producción (`TURNSTILE_ENABLED=true` con la
  clave secreta en el backend y `VITE_TURNSTILE_SITE_KEY` en el frontend) para
  frenar registros automatizados.

## Variables de marca y DEM

- `APP_NAME` (backend): nombre visible en `/health`, OpenAPI, correos y PDF.
  Por defecto `GNFIX`.
- `VITE_APP_NAME` y `VITE_APP_NAME_ACCENT` (frontend): nombre y parte que se
  pinta en naranja (`GN` crema + `FIX` naranja).
- `DEM_ENABLED`, `DEM_URL_TEMPLATE`, `DEM_ZOOM`, `DEM_CACHE_DIR`,
  `DEM_TIMEOUT_SECONDS`, `DEM_MAX_TILES`: modelo de elevación del QA de vuelo.
  La caché debe montarse como volumen para no re-descargar teselas.
- `REPORT_MAP_ENABLED`, `REPORT_MAP_URL_TEMPLATE`, `REPORT_MAP_CACHE_DIR`,
  `REPORT_MAP_TIMEOUT_SECONDS`, `REPORT_MAP_MAX_TILES`, `REPORT_MAP_WIDTH`,
  `REPORT_MAP_HEIGHT`, `REPORT_MAP_ATTRIBUTION`: mapa base OSM del informe PDF.
  Genera la trayectoria, la CORS base, norte, escala, grilla y leyenda. Si no
  hay red o las teselas fallan, el informe cae al croquis de cobertura.
- El informe lista las teselas DEM usadas y la URL del DEM en la sección
  “Fuentes”.

No renombrar el paquete `geohub`, las colas `geohub.*`, las métricas
`geohub_http_*` ni el bucket: no aportan al usuario y complican el despliegue.
Las cookies e identificadores de sesión sí usan `gnfix_*`.
