# Integración con la API

## Base

- API local: `http://127.0.0.1:8000`
- Prefijo: `/api/v1`
- OpenAPI: `/openapi.json`
- Swagger: `/docs`

## Autenticación

La API usa cookies HttpOnly:

- `geohub_access` — access token JWT de 15 min.
- `geohub_refresh` — refresh token opaco rotado.
- `geohub_csrf` — token CSRF para métodos mutables.
- `geohub_guest` — sesión de invitado.

Login:

```bash
curl -c cookies.txt \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'username=usuario@example.com' \
  --data-urlencode 'password=Secreta123!' \
  http://127.0.0.1:8000/api/v1/auth/login
```

Registro:

```bash
curl -c cookies.txt \
  -H 'Content-Type: application/json' \
  -d '{"email":"usuario@example.com","password":"Secreta123!","display_name":"Usuario"}' \
  http://127.0.0.1:8000/api/v1/auth/register
```

Clientes máquina (CLI, integraciones) que no quieran usar cookies:

```bash
curl -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'username=usuario@example.com' \
  --data-urlencode 'password=Secreta123!' \
  http://127.0.0.1:8000/api/v1/auth/token
# → {"access_token":"...","token_type":"bearer","expires_in":900}
```

Después: `Authorization: Bearer <access_token>`. Las respuestas de
`/auth/login`, `/auth/register` y `/auth/refresh` **no** incluyen el token: la
sesión web vive solo en cookies `HttpOnly`.

## Verificación de correo

- `POST /auth/verify-email` con `{"token": "..."}` consume el enlace enviado
  por correo y marca `email_verified=true`.
- `POST /auth/resend-verification` requiere sesión y CSRF.

## Turnstile

Si `TURNSTILE_ENABLED=true`, el registro y el login exigen la cabecera
`X-Turnstile-Token` con un token válido para la acción `register` o `login`.
Los endpoints de máquina (`/auth/token`) no usan CAPTCHA.

## OAuth

`GET /auth/oauth/{provider}/authorize` devuelve la URL del proveedor y setea la
cookie `HttpOnly` `geohub_oauth_state` (hash del `state`, ruta
`/api/v1/auth/oauth`). El callback exige esa cookie, la borra y rechaza el flujo
si no coincide. No compartas la URL de callback entre navegadores.

## CSRF

Todo `POST`, `PUT`, `PATCH` o `DELETE` autenticado requiere:

```http
X-CSRF-Token: <valor de la cookie geohub_csrf>
```

```bash
CSRF=$(awk '$6=="geohub_csrf" {print $7}' cookies.txt)
curl -b cookies.txt -H "X-CSRF-Token: $CSRF" ...
```

## Ejemplos

Subir RINEX de observación o navegación (máximo 200 MB):

```bash
curl -b cookies.txt -H "X-CSRF-Token: $CSRF" \
  -F 'file=@GEOS1550.19o' \
  http://127.0.0.1:8000/api/v1/rinex/files
```

Crear efemérides:

```bash
curl -b cookies.txt -H "X-CSRF-Token: $CSRF" \
  -H 'Content-Type: application/json' \
  -d '{"start_date":"2026-10-01","end_date":"2026-10-03","product_type":"final","constellations":["GPS","GLONASS"]}' \
  http://127.0.0.1:8000/api/v1/ephemeris/requests
```

Crear postprocesamiento:

```bash
curl -b cookies.txt -H "X-CSRF-Token: $CSRF" \
  -H 'Content-Type: application/json' \
  -d '{
    "rinex_file_id": "<uuid>",
    "base_rinex_file_id": null,
    "navigation_file_id": null,
    "auto_cors": true,
    "solution": "ppk_precise",
    "ephemeris_product_type": "final",
    "use_geoid": false
  }' \
  http://127.0.0.1:8000/api/v1/postprocessing/jobs
```

`navigation_file_id` es opcional: referencia a un RINEX de navegación subido
previamente. Si se informa, se usa esa navegación en lugar de buscarla en IGAC.

Modos de solución:

| `solution` | Base | Efemérides |
|---|---|---|
| `ppk_broadcast` | manual o auto-CORS | broadcast |
| `ppk_precise` | manual o auto-CORS | SP3/CLK/ERP con fallback |
| `ppk_static` | manual o auto-CORS | SP3/CLK/ERP con fallback y salida única |
| `ppp_static` | no | SP3/CLK/ERP |
| `ppp_kinematic` | no | SP3/CLK/ERP |

Catálogo de estaciones (lectura pública):

```bash
curl 'http://127.0.0.1:8000/api/v1/stations?limit=50&offset=0'
curl 'http://127.0.0.1:8000/api/v1/stations?search=BOG&active=true'
```

Descarga de puntos de un trabajo listo:

```bash
curl -b cookies.txt \
  'http://127.0.0.1:8000/api/v1/postprocessing/jobs/<uuid>/points?format=geojson' \
  -o points.geojson
```

Formatos disponibles: `geojson`, `kml`.

## Geotagging de imágenes de drone

Requiere una cuenta autenticada y un job PPK/PPP `ready` con solución
cinemática (`ppk_broadcast`, `ppk_precise` o `ppp_kinematic`).

Crear un lote con un ZIP de imágenes y el MRK:

La subida es una **sesión por partes**: el lote se crea primero (estado
`uploading`) y luego se envían los archivos. Así el proceso es visible en
`/geotagging` y `/activity` desde el primer segundo y se puede reanudar.

```bash
# 1. Abrir el lote (JSON pequeño); devuelve chunk_size y missing_parts
curl -b cookies.txt -H "X-CSRF-Token: $CSRF" -H 'Content-Type: application/json' \
  -d '{
        "processing_job_id": "<uuid-del-job>",
        "images_filename": "fotos.zip",
        "images_size_bytes": 187532894,
        "mrk_filename": "Timestamp.MRK",
        "time_system": "auto",
        "quality": "fix_float",
        "include_exif": true,
        "apply_lever_arm": true
      }' \
  http://127.0.0.1:8000/api/v1/geotagging/batches

# 2. Subir el MRK
curl -b cookies.txt -X PUT -H "X-CSRF-Token: $CSRF" \
  -H 'Content-Type: text/plain' --data-binary @Timestamp.MRK \
  http://127.0.0.1:8000/api/v1/geotagging/batches/<uuid>/mrk

# 3. Subir el ZIP por partes (chunk_size por defecto: 16 MiB)
split -b 16777216 fotos.zip part-
for n in 1 2 3; do
  curl -b cookies.txt -X PUT -H "X-CSRF-Token: $CSRF" \
    -H 'Content-Type: application/octet-stream' --data-binary @part-0$n \
    http://127.0.0.1:8000/api/v1/geotagging/batches/<uuid>/archive/parts/$n
done

# 4. Cerrar la subida y encolar el lote
curl -b cookies.txt -X POST -H "X-CSRF-Token: $CSRF" \
  http://127.0.0.1:8000/api/v1/geotagging/batches/<uuid>/complete
```

Reanudar tras una interrupción:

```bash
# Devuelve uploaded_bytes, received_parts y missing_parts
curl -b cookies.txt http://127.0.0.1:8000/api/v1/geotagging/batches/<uuid>
# Reenviar solo las partes de missing_parts y volver a ejecutar /complete
```

Cancelar una subida o reintentar un lote fallido:

```bash
curl -b cookies.txt -X POST -H "X-CSRF-Token: $CSRF" \
  http://127.0.0.1:8000/api/v1/geotagging/batches/<uuid>/cancel
curl -b cookies.txt -X POST -H "X-CSRF-Token: $CSRF" \
  http://127.0.0.1:8000/api/v1/geotagging/batches/<uuid>/retry
```

Las imágenes sueltas se envían con `POST /batches/{id}/images` y el campo
repetido `images`; el servidor construye el ZIP en una sola petición. Para
vuelos grandes conviene subir un **ZIP**: se envía por partes en paralelo (3 a
la vez) y es más rápido.

Consultar y descargar:

```bash
curl -b cookies.txt 'http://127.0.0.1:8000/api/v1/geotagging/batches'
curl -b cookies.txt 'http://127.0.0.1:8000/api/v1/geotagging/batches?processing_job_id=<uuid>'
curl -b cookies.txt -o geoetiquetas.txt \
  'http://127.0.0.1:8000/api/v1/geotagging/batches/<uuid>/geotags?format=txt'
curl -b cookies.txt -o geoetiquetas.csv \
  'http://127.0.0.1:8000/api/v1/geotagging/batches/<uuid>/geotags?format=csv'
curl -b cookies.txt -o reporte.json \
  'http://127.0.0.1:8000/api/v1/geotagging/batches/<uuid>/report'
curl -b cookies.txt \
  'http://127.0.0.1:8000/api/v1/geotagging/batches/<uuid>/images'
```

`/images` devuelve una URL firmada temporal para descargar el ZIP con EXIF
corregido.

## Perfil de postprocesamiento

`GET /api/v1/postprocessing/profile` describe el motor y las opciones fijas que
se aplican a cualquier trabajo (no son configurables):

```json
{
  "engine": "rtklib",
  "engine_label": "RTKLIB / RTKPOST",
  "rtklib": {
    "frequencies": 2,
    "elevation_mask_deg": 15.0,
    "snr_mask": null,
    "dynamics": false
  },
  "constellations_by_solution": { "ppk_precise": ["G", "R", "E"] },
  "modes_by_solution": { "ppk_precise": { "value": 2, "label": "kinematic" } },
  "outputs": ["solution.pos", "report.json", "trajectory.geojson"]
}
```

El reporte de cada trabajo incluye el bloque `profile` con lo realmente
ejecutado (incluida la efeméride usada y el fallback).

## Estaciones

`GET /api/v1/stations` es público y devuelve, además del catálogo IGAC, los
campos de la ficha técnica cuando estén cargados:

```json
{
  "code": "BOGT",
  "latitude": 4.640075,
  "longitude": -74.08094,
  "height_m": 0.0,
  "ctm12_e": 1012345.678,
  "ctm12_n": 1012345.678,
  "monument_height_m": 1.35,
  "monument_type": "Pilar",
  "source": "IGAC",
  "source_date": "2024-05-31"
}
```

## Calidad

```bash
# Lotes listos para evaluar
curl -b cookies.txt 'http://127.0.0.1:8000/api/v1/quality/flights'

# Métricas de vuelo (elevación del terreno opcional)
curl -b cookies.txt \
  'http://127.0.0.1:8000/api/v1/quality/flights/<batch>?ground_height_m=2550'

# Informe PDF (deliverable independiente)
curl -b cookies.txt -o informe.pdf \
  'http://127.0.0.1:8000/api/v1/quality/flights/<batch>/report.pdf'

# Control de base de un trabajo estático/PPP estático
curl -b cookies.txt 'http://127.0.0.1:8000/api/v1/quality/base/<job>'

# Transformación WGS84 -> CTM12
curl -b cookies.txt -H 'Content-Type: application/json' \
  -H "X-CSRF-Token: $CSRF" \
  -d '{"direction":"wgs84_to_ctm12","points":[{"latitude":4.64,"longitude":-74.08,"height_m":2600}]}' \
  http://127.0.0.1:8000/api/v1/quality/transform
```

## Errores

Formato:

```json
{
  "error": {
    "code": "postprocessing.invalid_solution",
    "message": "La combinación de modo de solución y base no es válida.",
    "details": {}
  },
  "request_id": "..."
}
```

Códigos HTTP habituales:

- `401` sesión ausente/expirada.
- `403` CSRF o permisos.
- `404` recurso no visible.
- `409` transición de estado inválida.
- `413` archivo demasiado grande.
- `415` extensión no soportada.
- `422` validación.
- `429` rate limit o cuota.
- `502/503/504` fallos de fuentes, almacenamiento o timeout.
