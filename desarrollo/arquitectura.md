# Arquitectura

GNFIX usa Arquitectura Hexagonal + DDD táctico + vertical slicing.

```text
geohub-backend/src/geohub/
├── auth/
├── bootstrap/          # composition root
├── delivery/
├── ephemeris/
├── geotagging/
├── jobs/
├── postprocessing/
├── rinex/
├── shared/
├── stations/
└── workers/
```

Cada módulo tiene:

```text
domain/          # entidades, VOs, servicios y errores puros
application/     # casos de uso, DTOs y puertos
infrastructure/  # adaptadores inbound/outbound
```

Reglas:

- `domain` no importa frameworks ni capas externas.
- `application` no importa infraestructura ni `bootstrap`.
- Solo `bootstrap/container.py` construye adaptadores.
- Los contratos son `typing.Protocol`.
- Los modelos ORM no salen de persistencia.
- UTC en todo el sistema; el tiempo GPS es un value object.

## Flujo de postprocesamiento

1. `QueueProcessingJob` valida modo/base y persiste el job + evento outbox.
2. `outbox.dispatch` publica `postprocessing.run`.
3. `RunProcessingJob` descarga rover/base, resuelve navegación y efemérides.
4. `RtklibCommandBuilder` construye el comando CLI de RTKLIB.
5. `Rnx2RtkpEngine` ejecuta `rnx2rtkp` sin shell.
6. `PosParser` + `SolutionProcessor` generan conteos, CTM12 y GeoJSON.
7. Se guardan `.pos`, reporte y trayectoria; se encola `delivery.package`.

## Flujo de geotagging de imágenes

1. `StartGeotagUpload` valida que el job sea `ready` y cinemático y crea el
   lote en estado `uploading` (antes de recibir archivos).
2. El cliente sube el MRK y el ZIP por partes (`UploadGeotagPart`, 16 MiB);
   cada parte confirmada se registra en `upload_parts` para poder reanudar.
3. `CompleteGeotagUpload` valida tamaño y MRK, compone el objeto en MinIO y
   deja el lote en `queued`. Un barrido periódico cancela las subidas
   abandonadas (`CancelStaleUploads`).
4. La tarea `geotagging.run` descarga la solución `.pos` y el MRK.
5. `DjiMrkParser` lee las exposiciones y `GeoTagMatcher` detecta GPS/UTC e
   interpola contra las épocas corregidas.
6. Se generan `geoetiquetas.txt`, CSV y `report.json`.
7. Si se solicita EXIF, `ExifToolWriter` escribe GPS en copias de las imágenes y
   se empaqueta `images_geotagged.zip`.
8. El lote queda `ready`; la limpieza diaria borra los lotes expirados y el
   barrido de 15 minutos cancela subidas inactivas.

## Flujo de calidad

1. `GetFlightQuality` lee el CSV de geotags del lote listo y calcula métricas
   geométricas (GSD, AGL, solape estimado, cobertura) con la proyección CTM12.
2. Los parámetros de cámara se capturan del EXIF de la primera imagen durante
   el geotagging y se guardan en el lote.
3. `GetBaseQuality` asocia el `MARKER NAME` del RINEX con una estación y compara
   la posición media del trabajo estático contra su ficha.
4. `GetFlightReport` compone el PDF con ReportLab, lo guarda como artefacto y
   lo marca en el lote para listarlo en Entregas.
5. `TransformCoordinates` convierte WGS84 ↔ CTM12 y aplica QGeoidCOL2023 para
   la altura ortométrica.

## Modelo de elevación (QA de vuelo)

1. `GetFlightQuality` pide a `TerrainElevationSource` la altura del terreno bajo
   cada geotag.
2. `TerrariumTerrainSource` calcula la tesela Web Mercator, la descarga (o la
   lee de `data/dem-cache`) y muestrea bilinealmente el RGB terrarium.
3. El DEM es ortométrico: se suma la ondulación local `N` (geoide del job o
   base) para compararlo con las alturas elipsoidales.
4. La elevación manual se interpreta como **elipsoidal**, se valida finita y
   entre -500 y 9000 m, y no se le suma ondulación.
5. Sin DEM ni elevación manual no se calcula AGL/GSD.

## Lectura agregada de actividad

`GET /activity` usa `GetActivity` y `SqlAlchemyActivityReader`: una sola sesión
ejecuta las dos consultas (trabajos y lotes) y devuelve las entidades ya
mapeadas. Evita dos autenticaciones y dos transacciones por refresco.

Las trayectorias ya **no** se guardan en `processing_jobs`; viven únicamente en
MinIO (`trajectory.geojson`) y se sirven por su clave.

## Endurecimiento de autenticación

- **OAuth sin pre-apropiación**: si ya existe una cuenta local con el correo,
  solo se vincula cuando el proveedor y la cuenta local tienen el correo
  verificado. Si la cuenta local no está verificada se devuelve
  `auth.account_link_unverified` y se audita el intento.
- **State atado al navegador**: `authorize` emite una cookie `HttpOnly` con el
  hash del `state`; el callback exige que coincida y la borra al terminar.
- **Token web solo en cookie**: login, registro y refresh no devuelven
  `access_token` en JSON. Los clientes máquina usan `POST /auth/token`.
- **Gracia de rotación**: un refresh rotado se acepta durante
  `refresh_rotation_grace_seconds` si coinciden UA/IP; fuera de la ventana se
  revoca la familia.
- **Outbox**: `list_pending` usa `FOR UPDATE SKIP LOCKED` para que dos
  dispatchers no publiquen el mismo evento.
- **Postproceso**: el `.pos` se parsea en streaming, se corta al superar
  `max_points` y no se carga completo para subirlo a MinIO.

## Antiabuso y verificación

- **CSRF pre-auth**: login, registro, refresh y logout comprueban el token CSRF
  o el `Origin`/`Sec-Fetch-Site` del navegador; los clientes sin cabeceras de
  navegador (CLI, tests) siguen funcionando.
- **Turnstile**: `require_turnstile(action)` valida `siteverify` antes de
  hashear contraseñas; el widget usa `X-Turnstile-Token` y la CSP permite
  `challenges.cloudflare.com`.
- **Cuota por IP**: `guest_action_quotas` usa un upsert atómico por
  `(ip_hash, acción, día)`; complementa la cuota por cookie.
- **Verificación de correo**: `email_verification_tokens` guarda solo el hash,
  caduca en 24 h y se invalida al reenviar; el enlace apunta a
  `/verify-email`. Hasta verificar, OAuth no vincula la cuenta.
- **SSRF**: las plantillas de estaciones pasan por `ensure_safe_remote_url`
  (esquema, credenciales, IP privada) y no se siguen redirecciones.
