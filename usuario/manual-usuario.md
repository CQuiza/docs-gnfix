# GNFIX — Manual de usuario

> **Versión del documento:** 1.0  
> **Última actualización:** 2026-11-08  
> **Audiencia:** usuarios finales, administradores y personas que operan el backend.  
> **También disponible en:** la ruta `/manual` del frontend, con descarga en Markdown e impresión a PDF.

GNFIX es una plataforma para descargar efemérides GNSS, ingerir archivos
RINEX, ejecutar postprocesamiento PPK con RTKLIB y entregar resultados
geodésicos con coordenadas de Colombia.

Este manual describe qué hace cada módulo, cómo usarlo, qué productos genera,
qué límites existen, qué errores pueden aparecer y cómo interpretarlos.

---

## 1. Visión general

GNFIX integra cuatro bloques funcionales:

| Bloque | Qué resuelve |
|---|---|
| **Efemérides GNSS** | Descarga productos IGS/CDDIS y espejos, los descomprime, calcula checksum y los almacena. |
| **Ingesta RINEX** | Recibe observaciones RINEX, normaliza Hatanaka/compresión, extrae metadatos y las guarda. |
| **Postprocesamiento PPK** | Ejecuta RTKLIB con un rover y una base, calcula la trayectoria y genera reportes. |
| **Entrega y geodesia** | Empaqueta resultados en ZIP, genera URLs firmadas y calcula CTM12/altura ortométrica. |

### 1.1 Componentes técnicos

- **API FastAPI** en `http://127.0.0.1:8000` (en desarrollo).
- **Frontend React/Vite** en `http://localhost:5173` (en desarrollo).
- **PostgreSQL + PostGIS** para solicitudes, archivos, trabajos y geometrías.
- **RabbitMQ + Celery** para tareas asíncronas (worker y beat).
- **MinIO/S3** para objetos: efemérides, RINEX, soluciones y paquetes ZIP.
- **RTKLIB** (`rnx2rtkp`) para el cálculo PPK.
- **QGeoidCOL2023** para ondulación geoidal y altura ortométrica.
- **Sentry** opcional para errores y **Prometheus** para métricas.

### 1.2 Roles

| Rol | Acceso |
|---|---|
| **Invitado** | Puede usar el panel, crear solicitudes limitadas, subir RINEX limitados y ejecutar PPK. No conserva historial al cambiar de dispositivo. |
| **Usuario** | Historial persistente por cuenta, solicitudes y archivos sin la cuota diaria de invitado. |
| **Administrador** | Todo lo anterior más la sección `/admin`: usuarios y auditoría de autenticación. |

> **Nota:** el rol se guarda en el backend. Las guardas del frontend son solo
> experiencia de usuario; la autorización real la aplica la API.

### 1.3 Qué puedes hacer desde la interfaz

- Registrarte o iniciar sesión con correo/contraseña, Google o GitHub.
- Entrar en modo invitado.
- Consultar tu cuota y actividad reciente.
- Solicitar efemérides por ventana UTC, constelación y producto.
- Subir archivos RINEX con arrastrar y soltar.
- Ejecutar postprocesamiento PPK con base propia o CORS automática.
- Ver la trayectoria en un mapa coloreada por calidad.
- Descargar paquetes ZIP con enlaces firmados.
- Revisar usuarios y auditoría si eres administrador.

---

## 2. Acceso y cuentas

### 2.1 Registro

Si ya existe una cuenta con ese correo y aún no está verificada, el acceso con
Google/GitHub **no vincula** automáticamente la cuenta: inicia sesión con tu
contraseña y verifica el correo desde el enlace enviado al registrarte.

Ruta: `/register`

Al crear la cuenta se envía un correo de verificación. Puedes reenviarlo
desde el aviso que aparece en el panel. Hasta verificar el correo, el acceso con
Google/GitHub no vincula la cuenta automáticamente; entra con tu contraseña.
El enlace caduca en 24 horas.

Campos:

- **Nombre visible** (opcional, máximo 120 caracteres).
- **Correo electrónico** (obligatorio; debe tener formato válido).
- **Contraseña** (mínimo 10 caracteres).
- **Confirmación de contraseña**.

Al registrarte, la API crea la cuenta, inicia sesión y redirige al panel.

> **Importante:** actualmente no existe flujo de verificación de correo ni
> recuperación de contraseña. Si pierdes la contraseña, un administrador debe
> gestionar la cuenta desde la base de datos o los endpoints administrativos.

### 2.2 Inicio de sesión

Ruta: `/login`

- Correo y contraseña.
- Botones de OAuth si Google/GitHub están configurados.
- Enlace a registro.
- Botón **Continuar como invitado**.

Las credenciales viajan en el cuerpo del formulario. La API responde con cookies
`HttpOnly`; el frontend **no** guarda tokens en `localStorage` ni los manipula.

### 2.3 OAuth

Si el backend tiene configurados `OAUTH_GOOGLE_*` o `OAUTH_GITHUB_*`, los botones
de OAuth abren el proveedor. El callback vuelve a `/auth/callback` y la sesión se
hidrata automáticamente.

Si el proveedor no está configurado, la API responde `503` y la interfaz muestra
un mensaje comprensible.

### 2.4 Modo invitado

Un visitante sin sesión recibe una cookie de invitado. Puede:

- Ver el panel.
- Crear solicitudes de efemérides dentro de la cuota diaria.
- Subir RINEX dentro de la cuota diaria.
- Ejecutar PPK (sin cuota específica de PPK).
- Descargar entregas de su sesión.

Limitaciones del invitado:

- El historial no se conserva si cambia de navegador o borra cookies.
- No puede acceder a `/admin`.
- Algunas solicitudes con aviso por correo requieren indicar un email.
- La cuota diaria es configurable por el operador del backend.

La sesión se renueva sola: si el token de acceso caduca, la app pide uno nuevo
con el refresh token y repite la operación. Solo si el refresh también caducó
se cierra la sesión y se pide iniciar sesión de nuevo.

### 2.5 Sesión, cookies y CSRF

- El access token vive en una cookie `HttpOnly` con duración corta.
- El refresh token vive en una cookie `HttpOnly` con rotación.
- El token CSRF se guarda en una cookie legible por JavaScript y se envía en la
  cabecera `X-CSRF-Token` en escrituras.
- El frontend refresca la sesión una sola vez por petición, coordina el refresh
  entre pestañas y evita bucles.
- Si el refresh falla, la sesión se limpia y se redirige al login.

> **Importante en desarrollo:** abre el frontend en `http://localhost:5173`.
> El frontend usa el proxy de Vite `/api` hacia `http://127.0.0.1:8000`, de modo
> que cookies y CSRF son del mismo origen. Si abres el frontend en
> `http://127.0.0.1:5173`, revisa `CORS_ORIGINS` o usa el proxy.

### 2.6 Administradores

Para promover un usuario existente:

```bash
cd geohub/geohub-backend
python -m geohub.scripts.promote_admin --email admin@example.com
```

En Docker:

```bash
docker compose -f docker-compose.dev.yml run --rm api \
  python -m geohub.scripts.promote_admin --email admin@example.com
```

La cuenta debe existir. Un administrador puede cambiar roles y activar o
desactivar usuarios desde la API; la interfaz actual muestra esa información en
modo lectura.

---

## 3. Navegación y panel

### 3.1 Menú lateral

| Ruta | Nombre | Qué muestra |
|---|---|---|
| `/home` | Inicio | Landing pública: flujo, capacidades, FAQ y contacto. |
| `/` | Panel | Resumen, cuota, actividad reciente y accesos rápidos. |
| `/ephemeris` | Efemérides | Historial de solicitudes. |
| `/ephemeris/new` | Nueva solicitud | Formulario de efemérides. |
| `/rinex` | RINEX | Carga y listado de observaciones. |
| `/postprocessing` | Postproceso | Formulario PPK e historial de trabajos. |
| `/postprocessing/:jobId` | Detalle PPK | Métricas, mapa y geotagging de un trabajo. |
| `/geotagging` | Geotagging | Historial y descargas de lotes. |
| `/activity` | Actividad | Procesos activos y recientes. |
| `/quality` | QA de vuelo | Calidad de vuelo, control de base y transformación de coordenadas. |
| `/stations` | Estaciones | Catálogo público de estaciones CORS con mapa. |
| `/deliveries` | Entregas | Paquetes ZIP disponibles. |
| `/admin` | Administración | Usuarios y auditoría (solo admin). |
| `/manual` | Manual | Esta documentación. |

### 3.2 Panel

El panel muestra:

- Saludo y estado de sesión.
- Cuota:
  - invitado: usados / límite y barra de progreso;
  - usuario: sin límite diario de efemérides.
- Tarjetas de actividad reciente:
  - solicitudes de efemérides;
  - trabajos PPK;
  - paquetes de entrega.
- Accesos rápidos a nueva solicitud, RINEX, postproceso y entregas.
- Aviso de modo invitado con enlace a registro.

Las listas se actualizan automáticamente mientras haya trabajos pendientes.

### 3.3 Catálogo de estaciones

`/stations` es público (también para invitados) y muestra el catálogo
IGAC/SIRGAS-CON:

- **Resumen**: total, activas, inactivas y número de redes.
- **Mapa**: todos los puntos, con leyenda por estado del IGAC (activa,
  inactiva, removida, sin estado), escala, flecha de norte y popup con la ficha
  técnica de la estación.
- **Tabla**: código, nombre, red, tipo de monumento, estado, latitud,
  longitud, altura y última sincronización.
- **Zoom rápido**: al hacer clic (o pulsar Enter) sobre una fila, el mapa
  vuela a esa estación, la resalta en ámbar y abre su popup. Sirve para
  verificar de inmediato si la localización cargada es correcta.
- **Filtros**: búsqueda por código o nombre, red y estado.
- **Paginación** de 25 filas.

**Ficha técnica de la estación**

Además de los datos del catálogo IGAC, cada estación puede tener una ficha
geodésica cargada manualmente:

| Campo | Contenido |
|---|---|
| `ctm12_e` | Coordenada plana Este (m) en EPSG:9377 (CTM12, Origen Nacional). |
| `ctm12_n` | Coordenada plana Norte (m) en EPSG:9377. |
| `monument_height_m` | Altura del monumento sobre la marca (m). |
| `monument_type` | Tipo de monumento: `Pilar`, `Moja`, `Roca`, `Edificio`, etc. |
| `source` | Procedencia de la ficha (ver más abajo). |
| `source_date` | Fecha de vigencia de la ficha (ver más abajo). |

`source` responde **de dónde salió el dato**. Sin él no se puede distinguir una
coordenada oficial de una levantada en campo, y cualquier diferencia futura es
imposible de auditar. Ejemplos: `IGAC`, `RRGICC`, `SGC`, `levantamiento local`.

`source_date` responde **de cuándo es el dato**. Las coordenadas oficiales
cambian con los ajustes de la red y con el mantenimiento del monumento; una
ficha de 2015 no debe compararse contra una de 2024 como si fueran
equivalentes. Se guarda como fecha (`YYYY-MM-DD`) y se muestra junto a la
fuente en el popup.

Estos campos **no se sobrescriben** cuando el catálogo IGAC se refresca cada
mes; por eso conviene cargarlos a mano solo cuando exista una ficha verificable.

Ejemplo de carga directa:

```sql
UPDATE cors_stations
SET ctm12_e = 1012345.678,
    ctm12_n = 1012345.678,
    monument_height_m = 1.35,
    monument_type = 'Pilar',
    source = 'IGAC',
    source_date = DATE '2024-05-31'
WHERE code = 'BOGT';
```

El estado mostrado es el del catálogo del IGAC, no la disponibilidad de
archivos RINEX del día. El catálogo se actualiza automáticamente el día 1 de
cada mes a las 05:00 UTC.

### 3.4 Actividad

`/activity` reúne los trabajos de postprocesamiento y los lotes de geotagging en
**una sola petición**. Refresca cada **10 s en reposo** y cada **4 s** si hay
procesos activos, sin solapar peticiones; el estado vive en el servidor.

---

## 4. Efemérides GNSS

### 4.1 Qué son

Las efemérides describen la órbita y el reloj de los satélites GNSS. GNFIX
descarga productos de varias fuentes, los valida y los deja disponibles para
descarga o para usarlos como insumo en postprocesamiento.

### 4.2 Crear una solicitud

Ruta: `/ephemeris/new`

#### Ventana temporal

- **Fecha inicial UTC** y **fecha final UTC**.
- Se interpretan como días completos:
  - inicio a las `00:00:00Z`;
  - fin a las `23:59:59Z`.
- La fecha final debe ser **posterior** a la inicial.
- La fecha final debe estar en el **pasado**.
- El rango máximo es de **31 días**.

#### Constelaciones

Selecciona al menos una:

- GPS.
- GLONASS.
- Galileo.
- BeiDou.
- QZSS.
- SBAS.
- NavIC (IRNSS).

Puedes usar **Todas** o **Ninguna** como atajos.

#### Producto

| Producto | Latencia típica | Uso recomendado |
|---|---|---|
| **Final** | ≈ 12 días después del cierre | Máxima precisión para estudios geodésicos. |
| **Rápida** | ≈ 17 horas después del cierre | Equilibrio entre latencia y precisión. |
| **Ultra rápida** | ≈ 3 horas después del cierre | Planificación y trabajos que necesitan datos recientes. |

La latencia es una estimación. Si el producto aún no está publicado, la solicitud
queda en `waiting_publication` y se reintenta automáticamente.

#### Insumos auxiliares

| Insumo | Contenido |
|---|---|
| **CLK** | Correcciones de reloj satelital. |
| **ERP** | Parámetros de orientación terrestre. |
| **ATX** | Calibración de antenas. |
| **IONEX** | Mapas ionosféricos. |

Los insumos seleccionados se descargan y se empaquetan junto con los productos
principales.

#### Entrega y avisos

| Modo | Comportamiento |
|---|---|
| **Descarga** | Enlace firmado desde la plataforma. |
| **Enlace efímero** | URL temporal con retención limitada. |
| **Correo** | Requiere un correo válido; se envía el enlace al finalizar. |

La opción **Avísame cuando esté disponible**:

- reintenta la solicitud si el producto aún no está publicado;
- un invitado debe indicar un correo;
- un usuario registrado puede usar el correo de su cuenta.

#### Resumen y envío

El panel derecho muestra:

- ventana y número de días;
- constelaciones;
- producto;
- entrega;
- disponibilidad estimada;
- cuota de invitado.

El botón **Crear solicitud** se deshabilita mientras se consulta la cuota o si
la cuota de invitado está agotada.

### 4.3 Qué ocurre después

1. La API guarda la solicitud y registra un evento `ephemeris.resolve` en el
   outbox, en la misma transacción.
2. Beat publica el evento en RabbitMQ.
3. El worker consulta las fuentes en el orden configurado:
   CDDIS, BKG, IGN, ESA, GFZ y CODE.
4. Descarga, descomprime, calcula SHA-256 y sube los productos a MinIO.
5. Marca la solicitud como `ready`.
6. Se registra un evento `delivery.package` para generar el ZIP de entrega.

Fuentes y fallback:

- CDDIS usa autenticación Earthdata.
- Si una fuente falla, se intenta la siguiente.
- Si ninguna tiene el producto, la solicitud vuelve a espera o falla según el
  caso.

### 4.4 Estados de una solicitud

| Estado | Significado |
|---|---|
| `requested` | Creada, pendiente de resolución. |
| `waiting_publication` | El producto aún no está publicado; se reintentará. |
| `resolving` | El worker está resolviendo la solicitud. |
| `downloading` | Se están descargando productos. |
| `ready` | Productos descargados y entrega disponible. |
| `failed` | Falló la resolución; revisa el motivo. |
| `expired` | La retención expiró. |
| `cancelled` | Cancelada (no expuesta en la interfaz actual). |

### 4.5 Historial

Ruta: `/ephemeris`

- Tabla con rango, producto, constelaciones, entrega, estado, fecha y acciones.
- Paginación.
- Actualización automática cada 15 segundos mientras haya solicitudes
  pendientes.
- Botón **Actualizar**.
- Enlace **Ver entrega** cuando la solicitud está `ready`.
- Motivo del fallo visible en la fila cuando aplica.

### 4.6 Productos y almacenamiento

Los objetos se guardan bajo:

```text
ephemeris/{request_id}/{filename}
```

Puedes consultar los productos por API:

```http
GET /api/v1/ephemeris/requests/{request_id}/products
```

Cada producto incluye fuente, URL original, clave de MinIO, checksum SHA-256,
tamaño y fecha de descarga.

### 4.7 Errores frecuentes de efemérides

| Situación | Mensaje o código | Qué hacer |
|---|---|---|
| Fecha final futura | `ephemeris.future_date_not_allowed` | Elige una fecha pasada. |
| Rango mayor a 31 días | `ephemeris.range_too_long` | Reduce la ventana. |
| Sin constelaciones | `domain.invalid_value` / validación | Selecciona al menos una. |
| Cuota de invitado agotada | `ephemeris.quota_exceeded` | Espera la ventana o crea una cuenta. |
| Entrega por correo sin email | `ephemeris.invalid_delivery` | Indica un correo. |
| Producto aún no publicado | `ephemeris.product_not_available` | Activa el aviso o espera. |
| Fuentes caídas | `ephemeris.all_sources_unavailable` | Revisa `/api/v1/ephemeris/sources`. |
| Error de resolución | `ephemeris.resolution_failed` | Revisa el detalle y vuelve a intentar. |
| Almacenamiento caído | `domain.object_storage_unavailable` | Revisa MinIO; la API responde 503. |

### 4.8 Consejos

- Usa ventanas de 1 a 7 días para reducir tiempos de descarga.
- Para trabajos urgentes, usa **Rápida** o **Ultra rápida**.
- Para máxima precisión, usa **Final** y planifica la latencia.
- Verifica la disponibilidad de fuentes antes de solicitar productos recientes.
- Si solo necesitas un producto, evita marcar todos los insumos auxiliares.

---

## 5. Archivos RINEX

### 5.1 Qué es RINEX

RINEX (Receiver Independent Exchange Format) es el formato estándar para
observaciones GNSS. GNFIX acepta archivos de observación, los normaliza y
extrae metadatos para usarlos como rover o base en PPK.

### 5.2 Formatos aceptados

| Tipo | Extensiones |
|---|---|
| RINEX de observación | `.rnx`, `.obs`, `.o`, `.crx`, `.d` |
| RINEX de navegación | `.nav`, `.n`, `.YYn`, `.YYg`, `.YYl`, `.YYh`, `.YYp` |
| RINEX 2 por año | `.YYo`, `.YYd` (por ejemplo `.24o`, `.24d`) |
| Comprimidos | `.gz`, `.Z`, `.bz2`, `.zip` |

- Se acepta RINEX 2, 3 y 4.
- Hatanaka (`.crx`) se normaliza automáticamente.
- La compresión se descomprime automáticamente.
- Se aceptan archivos de **observación** y de **navegación**.
- La navegación se usa en postprocesamiento cuando no hay una fuente CORS
  disponible o cuando el vuelo está fuera de cobertura IGAC.
- Tamaño máximo de subida: **200 MB** por defecto.
- Tamaño máximo del RINEX normalizado: **200 MB** por defecto.

### 5.3 Subir un archivo

Ruta: `/rinex`

1. Arrastra uno o varios archivos a la zona de carga o haz clic para
   seleccionarlos. Puedes añadir más con **Añadir archivos** (hasta 10 por
   lote).
2. Revisa la lista, quita los que no quieras y pulsa **Subir y procesar
   todos**.
3. La subida es secuencial, con progreso real por archivo y una fase separada
   de **procesando en servidor** (normalización y metadatos).
4. Al terminar, el archivo queda seleccionado y sus metadatos aparecen en
   **Archivo procesado**.
5. Para ver los metadatos de cualquier otro archivo, haz clic (o pulsa Enter)
   sobre su fila en **Archivos cargados**. No está limitado al último.

Puedes cancelar una carga en curso. El archivo original y el normalizado se
guardan en MinIO.

### 5.4 Metadatos extraídos

- Versión de RINEX.
- Tipo de archivo.
- Constelaciones detectadas.
- Intervalo de observación.
- Primera y última observación.
- Número de épocas.
- Nombre del marcador.
- Tipo de receptor.
- Tipo de antena.
- Posición aproximada WGS84 (latitud, longitud, altura).
- Tamaño y checksum SHA-256.
- Número de líneas de encabezado.

El nombre normalizado usa el patrón
`<nombre-original>-geohub-<uuid8>.<extension>`, por ejemplo
`00091690D-geohub-4a3f9d05.23o`. El nombre original se conserva y se muestra en
la lista de archivos.

### 5.5 Cuota de invitado

- Por defecto: **1 archivo por día**.
- La cuota se muestra en la tarjeta de carga.
- Si se agota, el botón se bloquea y aparece un aviso.
- Un usuario registrado no tiene esa cuota diaria.

### 5.6 Errores frecuentes de RINEX

| Situación | Código | Qué hacer |
|---|---|---|
| Extensión no soportada | `rinex.invalid_filename` | Usa una extensión válida. |
| Archivo demasiado grande | `rinex.too_large` | Reduce el tamaño o comprime. |
| RINEX corrupto o no normalizable | `rinex.archive_error` | Verifica el archivo y su compresión. |
| Cuota agotada | `rinex.quota_exceeded` | Espera o crea una cuenta. |
| Archivo no encontrado para el usuario | `rinex.not_found` | Actualiza la lista y vuelve a seleccionarlo; el mensaje indica si falta el rover, la base o la navegación. |
| MinIO caído | `domain.object_storage_unavailable` | Revisa el túnel y reintenta. |

### 5.7 Consejos

- Verifica que el encabezado tenga `APPROX POSITION XYZ`; es necesario para
  seleccionar una CORS automática.
- Usa rover y base de la **misma ventana temporal**.
- Si el RINEX es muy grande, comprímelo antes de subirlo.
- Revisa la constelación detectada antes de ejecutar PPK.
- Para PPK, el archivo normalizado es el que se usa; el original queda como
  respaldo.

### 5.8 Objetos en almacenamiento

```text
rinex/{file_id}/original/{original_filename}
rinex/{file_id}/{normalized_filename}
```

---

## 6. Postprocesamiento PPK

### 6.1 Qué es PPK/PPP

PPK (Post-Processed Kinematic) calcula la posición de un rover usando las
observaciones de una base y correcciones diferenciales. PPP (Precise Point
Positioning) calcula la posición con un solo receptor y productos precisos
(SP3/CLK/ERP). GNFIX ejecuta RTKLIB (`rnx2rtkp`) y genera una solución
`.pos`, un reporte y una trayectoria GeoJSON.

Modos disponibles:

| Modo | Base | Efemérides |
|---|---|---|
| PPK broadcast | manual o auto-CORS | broadcast |
| PPK preciso | manual o auto-CORS | SP3/CLK/ERP |
| Estático | manual o auto-CORS | SP3/CLK/ERP, con fallback a broadcast |
| PPP estático | no | SP3/CLK/ERP |
| PPP cinemático | no | SP3/CLK/ERP |

> **Estático** es el modo *Static* de RTKLIB (`-p 3`). Requiere base y devuelve
> una sola coordenada consolidada, no una trayectoria época a época.

### 6.2 Requisitos

- Un RINEX de **rover** cargado.
- Para PPK: una referencia base:
  - otro RINEX de base cargado; o
  - una estación CORS activa a menos de 20 km del rover.
- Para PPP: no se necesita base, pero la observación debe ser prolongada
  (mínimo recomendado: 60 minutos) y de doble frecuencia.
- Rover y base deben cubrir la misma ventana temporal.
- Para CORS automática, el rover debe tener posición aproximada en el
  encabezado. GNFIX descarga la observación y las efemérides de la estación.
- PPK preciso y PPP descargan SP3/CLK/ERP; si no hay productos, PPK preciso cae
  a broadcast y se avisa en el detalle del trabajo.

### 6.3 Crear un trabajo

Ruta: `/postprocessing`

1. Selecciona el **RINEX del rover**.
2. Opcionalmente, selecciona un **RINEX de navegación**. Si no lo haces,
   GNFIX busca navegación en la estación IGAC más cercana; para vuelos fuera
   de Colombia debes subir el `.nav` del dron o de la base.
3. Elige el **modo de solución**:
   - PPK broadcast;
   - PPK preciso;
   - Estático (PPK static);
   - PPP estático;
   - PPP cinemático.
4. Si es PPK, elige la referencia base:
   - **CORS automática**: el backend busca la estación activa más cercana
     (máximo 20 km por defecto, `CORS_MAX_DISTANCE_KM`);
   - **RINEX propio**: selecciona un archivo distinto al rover.
   Si es PPP, los campos de base se deshabilitan.
5. Si es PPK preciso o PPP, elige la **calidad de efemérides**:
   - automático (mejor disponible);
   - final;
   - rápida;
   - ultra-rápida.
6. Activa **QGeoidCOL2023** si necesitas altura ortométrica.
7. Pulsa **Encolar postprocesamiento**.

El motor de cálculo es siempre **RTKLIB / RTKPOST**; no hay presets de salida.
Las máscaras y constelaciones son fijas (ver 6.4). Al encolar se abre un
**modal de confirmación** con el resumen de insumos, perfil del motor y archivos
que se generarán. Debes pulsar **Confirmar y encolar** para crear el trabajo.

El formulario muestra un aviso según el modo. En PPK preciso se avisa del
fallback a broadcast; en PPP se avisa de la duración mínima y de que la solución
puede no ser adecuada en sesiones cortas.

El trabajo se crea en estado `queued` y se redirige a su detalle.

### 6.4 Procesamiento

- El worker consume la cola `geohub.postprocessing`.
- RTKLIB se ejecuta en un subproceso aislado, sin shell.
- Timeout PPK: **1 800 segundos** (30 min).
- Timeout PPP y Estático: **3 600 segundos** (60 min).
- Límite duro de la tarea Celery: **7 200 s**; aviso a los 6 900 s.
- Un timeout **no se reintenta**: el trabajo pasa a `failed` con el motivo.
- Un trabajo que quedó huérfano (worker caído) se retoma si supera 130 min sin
  actividad; si no, no se re-ejecuta (idempotencia ante reentregas).
- Tamaño máximo de salida: **50 MB**.
- Épocas máximas: **200 000**.
- PPP con menos de 60 minutos guarda una advertencia en el trabajo.
- La página actualiza el estado cada 5 segundos mientras está pendiente.

**Opciones fijas del motor**

El motor es siempre **RTKLIB / RTKPOST** y no se configuran máscaras desde la
interfaz. Valores aplicados:

| Opción | Valor |
|---|---|
| Frecuencias | L1 + L2 |
| Constelaciones | GPS + GLONASS (+ Galileo con productos precisos) |
| Máscara de elevación | 15° |
| Máscara SNR | desactivada |
| Dinámica del rover | desactivada |
| Salidas | `solution.pos`, `report.json`, `trajectory.geojson` |

El modal de confirmación muestra estos valores antes de encolar y el reporte
del trabajo los guarda en el bloque `profile`.

### 6.5 Estados del trabajo

| Estado | Significado |
|---|---|
| `queued` | Encolado, esperando worker. |
| `running` | RTKLIB en ejecución. |
| `ready` | Solución y entrega disponibles. |
| `failed` | Error de procesamiento. |

### 6.6 Resultados

El detalle muestra:

- Modo de solución y motor (RTKLIB / RTKPOST).
- Efemérides usadas: broadcast o precisas, producto (final/rápida/ultra) y
  fuente.
- Avisos de fallback o de duración insuficiente.
- Épocas totales. En **Estático** será 1, porque la solución es una coordenada
  consolidada.
- Conteos **Fix / Float / Single / PPP / otros**.
- Latitud, longitud y altura media.
- Altura elipsoidal.
- Altura ortométrica (si se usó QGeoidCOL2023).
- Coordenadas **MAGNA-SIRGAS Origen Nacional CTM12**:
  - Este;
  - Norte.
- Fechas de creación y finalización.
- Enlace a la entrega ZIP cuando está `ready`.

### 6.7 Mapa de trayectoria

- Leaflet + OpenStreetMap.
- Colores por calidad:
  - **Fix**: verde;
  - **Float**: ámbar;
  - **Single**: azul;
  - **PPP**: violeta;
  - **Otros**: gris.
- Marcadores de inicio y fin.
- Ajuste automático de límites.
- La trayectoria se obtiene de:
  ```http
  GET /api/v1/postprocessing/jobs/{job_id}/trajectory
  ```
- El GeoJSON incluye `qualities` y `timestamps` por época.

> **Nota:** el mapa requiere conexión a Internet para las teselas de
> OpenStreetMap. Si no hay conexión, la trayectoria puede no verse sobre el
> mapa, aunque el GeoJSON y el ZIP siguen disponibles.

#### Descargar puntos GeoJSON/KML

En la tarjeta **Trayectoria** de un trabajo `ready`, usa **Descargar puntos**:

- **Formato**: GeoJSON o KML.
- **Puntos**: todos o solo Fix/Float.
- La descarga usa la trayectoria del trabajo, no el archivo `.pos`.
- Un trabajo estático exporta una sola coordenada consolidada.

Equivale a:

```http
GET /api/v1/postprocessing/jobs/{job_id}/points?format=geojson&quality=all
```

Formatos disponibles: `geojson`, `kml`. Filtros: `all`, `fix_float`.

### 6.8 Errores frecuentes de PPK

| Situación | Código | Qué hacer |
|---|---|---|
| Falta base y no hay CORS | `postprocessing.base_rinex_required` | Sube una base o habilita CORS cercana. |
| PPP con base | `postprocessing.invalid_solution` | Quita la base: PPP no la usa. |
| No hay CORS cercana | `postprocessing.cors_station_not_found` | Sube una base propia. |
| No hay productos precisos | `postprocessing.precise_ephemeris_unavailable` | Usa PPK broadcast o elige otra calidad. |
| No hay navegación | `postprocessing.navigation_unavailable` | Sube un RINEX de navegación o revisa la cobertura CORS. |
| Navegación inválida | `postprocessing.invalid_navigation` | Selecciona un RINEX de navegación (tipo N), no de observación. |
| RTKLIB falló | `postprocessing.rtk_engine_error` | Revisa rover/base y el detalle técnico. |
| RTKLIB excedió timeout | `postprocessing.rtk_timeout` | Reduce la ventana o revisa el equipo. |
| Solución `.pos` inválida | `postprocessing.pos_parse_error` | Verifica la calidad de las observaciones. |
| Trayectoria aún no lista | `postprocessing.trajectory_not_ready` | Espera a que el trabajo esté `ready`. |
| Trabajo no encontrado | `postprocessing.not_found` | Revisa el identificador o la sesión. |
| MinIO caído | `domain.object_storage_unavailable` | Revisa el túnel y reintenta. |

### 6.9 Consejos

- Usa ventanas cortas (1 a 7 días) para reducir tiempo de cómputo.
- Verifica que la base esté dentro de la ventana y relativamente cerca.
- Si usas CORS automática, revisa que el rover tenga posición aproximada.
- Activa QGeoidCOL2023 solo si necesitas altura ortométrica; el modelo cubre
  Colombia.
- Revisa los conteos Fix/Float antes de usar la solución.
- Si el trabajo falla, abre **Detalle técnico** y compártelo con el
  administrador junto con el ID del trabajo.

### 6.10 Geotagging de imágenes de drone (DJI MRK)

Disponible en un trabajo `ready` con solución cinemática: PPK broadcast, PPK
preciso o PPP cinemático. Las soluciones estáticas no sirven para geolocalizar
cada fotografía.

**Requisitos**

- Cuenta autenticada; el geotagging no está disponible para invitados.
- El RINEX del rover y el MRK deben ser del mismo vuelo.
- El RINEX conviene a 1 Hz o más; si las fotos caen entre épocas, GNFIX
  interpola solo huecos pequeños.
- Archivos: un ZIP de fotos o imágenes sueltas (`.jpg`, `.jpeg`, `.tif`,
  `.tiff`, `.dng`) y el `Timestamp.MRK` de DJI.
- Máximo **6 GB** y **750 imágenes** por lote. Un lote de 750 imágenes puede
  tardar alrededor de 45 minutos con reescritura EXIF.

**Formato MRK soportado (DJI)**

Columnas separadas por tabuladores o espacios:

```text
foto  segundos_semana  semana  N,E  E,E  V,E  lat,Lon  lon,Lon  h,Ellh  stdN stdE stdU  flag,Q
```

- `semana` puede venir como `[2024]`.
- Los offsets N/E/V están en milímetros y se aplican si la opción
  **Aplicar offset antena-cámara** está activa.
- La base de tiempo se detecta automáticamente comparando las coordenadas del
  MRK con la trayectoria. Si hay ambigüedad, puedes elegir `GPS` o `UTC`.

**Pasos**

1. Abre el detalle de un trabajo PPK/PPP `ready`.
2. En **Geotagging de imágenes**, selecciona el ZIP o las fotos sueltas.
3. Selecciona el MRK.
4. Elige base de tiempo (`Automática` recomendada), calidad mínima y si
   quieres reescribir EXIF.
5. Pulsa **Generar geoetiquetas**.
6. Espera a que el lote pase a **Listo**.

**Subida por partes y reanudación**

- El lote se crea en el servidor **antes** de subir los archivos, en estado
  **Subiendo**. Aparece de inmediato en `/geotagging` y `/activity`, incluso
  desde otra pestaña.
- El ZIP se envía por partes de 16 MB, **hasta 3 en paralelo** para aprovechar
  enlaces con latencia alta. La barra muestra porcentaje y bytes reales, y el
  progreso vive en el servidor.
- Puedes cambiar de página o refrescar sin perder lo subido. Al volver a la
  tarjeta verás **Subida incompleta**: vuelve a seleccionar el mismo ZIP y el
  MRK y pulsa **Reanudar subida**; solo se envían las partes que faltan.
- **Cancelar subida** aborta el envío y libera los archivos parciales.
- Si el lote falla al procesar, el botón **Reintentar procesamiento** lo
  vuelve a encolar mientras los archivos sigan disponibles.
- Las subidas abandonadas se cancelan automáticamente a las 24 horas.

**Salidas**

- `geoetiquetas.txt`: una línea por imagen con
  `nombre latitud longitud altura_elipsoidal`, sin cabecera. Es el formato
  habitual para Pix4D, Agisoft Metashape, WebODM y DJI Terra.
- `geoetiquetas.csv`: incluye calidad, método de match, timestamp GPS y número
  de foto.
- `report.json`: conteos, offset temporal detectado, imágenes no emparejadas y
  advertencias.
- `images_geotagged.zip`: copia de las imágenes con GPS EXIF corregido. Las
  imágenes sin match conservan su EXIF original.

**Errores frecuentes**

| Código | Qué hacer |
|---|---|
| `geotagging.forbidden` | Inicia sesión; el lote requiere cuenta. |
| `geotagging.unsupported_solution` | Usa PPK broadcast/preciso o PPP cinemático. |
| `geotagging.invalid_mrk` | Verifica que sea el MRK DJI del mismo vuelo. |
| `geotagging.invalid_images` | Revisa que el ZIP tenga imágenes y nombres únicos. |
| `geotagging.unsupported_image` | Usa JPEG, TIFF o DNG. |
| `geotagging.too_large` | Reduce el ZIP o divide el vuelo. |
| `geotagging.invalid_upload_part` | Reenvía la parte; pudo llegar cortada. |
| `geotagging.invalid_upload_state` | El lote ya no está subiendo; crea uno nuevo. |
| `geotagging.not_resumable` | La sesión de subida expiró; crea un lote nuevo. |
| `geotagging.not_ready` | Espera a que el job y el lote estén listos. |

### 6.11 QA de vuelo, control de base y transformación

`/quality` reúne las evaluaciones de calidad. Tiene tres pestañas.

**Vuelo**

- Lista los lotes de geotagging en estado listo.
- Al seleccionar uno calcula, a partir de las geotags corregidas: GSD medio,
  altura sobre el terreno (AGL), distancia, duración, velocidad, solape
  longitudinal y lateral estimados, y área de cobertura.
- **Calidad de las imágenes**: cuántas quedaron en Fix, Float, Single y PPP,
  con su porcentaje sobre las emparejadas y el número de no emparejadas.
- **Precisión formal**: Sigma Norte, Sigma Este y Sigma Altura (media, mediana y
  máximo) tomadas de la solución RTKLIB. Son errores formales, no exactitud
  empírica.
- **Terreno**: por defecto se usa un **DEM de 30 m (teselas terrarium de
  Mapzen/AWS, zoom 13)**. La ondulación local N se estima desde la base del
  vuelo o desde el geoide del job, y se aplica para comparar alturas
  elipsoidales con el DEM ortométrico. También puedes indicar una elevación
  manual como alternativa; esa elevación debe ser **elipsoidal** (no msnm). El
  API valida que sea finita y esté entre -500 y 9000 m.
  - Si no hay DEM ni elevación, **no se calculan AGL ni GSD**; solo se muestra
    el rango de altura relativa del vuelo.
- El área se dibuja como polígono en el mapa.
- Los parámetros de cámara (modelo, focal 35 mm y resolución) se leen del EXIF
  de la primera imagen al procesar el lote. Los lotes antiguos sin cámara no
  calculan GSD ni cobertura.
- **Informe PDF**: se genera bajo demanda con el equipo, la calidad por imagen,
  la precisión formal, las métricas de vuelo, el croquis a escala (grilla E/N,
  barra de escala y norte), la base con su línea de base y las advertencias.
  Queda como entregable independiente y aparece en **Entregas → Informes de
  vuelo**.

**Base**

- Lista los trabajos **estáticos** o **PPP estático**.
- Asocia el `MARKER NAME` del RINEX con la ficha de la estación y compara
  ΔE, ΔN y Δh contra las coordenadas CTM12 y la altura de la ficha.
- Semáforo: ≤ 3 cm correcto, ≤ 10 cm revisar, > 10 cm fuera de tolerancia.

**Transformación**

- Convierte coordenadas WGS84 → CTM12 (EPSG:9377) y a la inversa.
- Formato de entrada: una línea por punto, `latitud, longitud[, altura]` o
  `este, norte`.
- Devuelve E/N y, si indicas altura, la altura ortométrica con QGeoidCOL2023.
- Permite descargar el resultado en CSV.

---

## 7. Entregas

### 7.1 Qué es una entrega

Cuando una solicitud de efemérides o un trabajo PPK queda `ready`, el backend
genera un ZIP, lo sube a MinIO y crea un registro en `delivery_packages`.

Ruta: `/deliveries`

### 7.2 Contenido del ZIP

Para un trabajo PPK:

```text
solution.pos
report.json
trajectory.geojson
manifest.json
```

Para una solicitud de efemérides:

- los productos descargados;
- los insumos auxiliares seleccionados;
- `manifest.json`.

`manifest.json` incluye tipo de recurso, identificador, fecha de generación y
lista de archivos con tamaños.

### 7.3 Retención y URLs

- Retención por defecto: **48 horas**.
- Vida de la URL firmada: **48 horas**.
- Al expirar, el paquete pasa a `expired` y el objeto se elimina por la tarea
  `delivery.cleanup`.
- Si el paquete expiró y se vuelve a solicitar la entrega, se renueva el mismo
  registro con un objeto nuevo.

### 7.4 Listado y descarga

- Tabla con archivo, recurso, tamaño, estado, expiración y descarga.
- Filtro por recurso mediante `?resource=<id>`.
- La descarga usa una URL firmada de MinIO.
- Sección **Resultados de geotagging**: TXT, CSV, reporte e imágenes EXIF de los
  lotes listos.
- Sección **Informes de vuelo**: PDF de calidad generados desde `/quality`.
  No forma parte del ZIP; es un entregable independiente.
- Si `SMTP_HOST` está configurado, se envía un correo al propietario con el
  enlace.

### 7.5 Errores frecuentes de entrega

| Situación | Código | Qué hacer |
|---|---|---|
| Paquete no encontrado | `delivery.not_found` | Revisa el ID o la sesión. |
| Recurso aún no listo | `delivery.source_not_found` | Espera a que el trabajo esté `ready`. |
| Paquete expirado | estado `expired` | Solicita una nueva entrega. |
| MinIO caído | `domain.object_storage_unavailable` | Revisa el túnel y reintenta. |
| URL no disponible | `download_url: null` | Reintenta más tarde o revisa MinIO. |

---

## 8. Administración

Ruta: `/admin` (solo administradores).

### 8.1 Usuarios

- Lista de usuarios registrados.
- Correo, nombre, rol, estado y fecha de registro.
- Los roles son `user` y `admin`.
- Los usuarios inactivos no pueden iniciar sesión.

La API ofrece además:

```http
GET   /api/v1/admin/users
PATCH /api/v1/admin/users/{user_id}/role
POST  /api/v1/admin/users/{user_id}/activate
POST  /api/v1/admin/users/{user_id}/deactivate
```

> **Limitación actual:** la interfaz muestra usuarios y auditoría en modo
> lectura. Los cambios de rol/estado se hacen por API.

### 8.2 Auditoría de autenticación

- Eventos de login, registro, refresh, logout, OAuth y cambios privilegiados.
- Acción, resultado, actor, recurso, IP hash, user agent, request ID y detalles.
- La lectura está restringida a administradores.

### 8.3 Auditoría de workers

Las tareas que procesan un recurso o fallan registran en
`worker_audit_events`:

- nombre de tarea;
- `task_id`;
- cola;
- recurso;
- estado;
- duración;
- error.

Los despachos de outbox y sondeos vacíos no generan fila. La auditoría se purga
automáticamente después de `WORKER_AUDIT_RETENTION_DAYS` (30 días por defecto).
Esta tabla no está expuesta aún en la interfaz; se consulta por base de datos o
herramientas de operación.

---

## 9. Límites y cuotas

Los valores son configurables por el operador. Los predeterminados son:

| Límite | Valor por defecto |
|---|---|
| Solicitudes de efemérides de invitado | 3 por día |
| Archivos RINEX de invitado | 1 por día |
| Rango máximo de efemérides | 31 días |
| Tamaño máximo de subida RINEX | 200 MB |
| Tamaño máximo normalizado RINEX | 200 MB |
| Retención de geotagging | 168 h |
| Retención de entregas | 48 h |
| Vida de URL firmada | 48 h |
| Timeout PPK | 1 800 s |
| Timeout PPP y Estático | 3 600 s |
| Límite duro de tarea de postproceso | 7 200 s |
| Duración mínima recomendada PPP | 60 min |
| Salida máxima PPK | 50 MB |
| Épocas máximas PPK | 200 000 |
| Distancia máxima CORS | 20 km |
| Radio de búsqueda de navegación | 2000 km |
| Muestreo preferido IGAC | 15 s |
| Rate limit global | 600 requests/minuto |
| Login | 5 intentos / 15 min |
| Registro | 5 / hora |
| Refresh | 30 / minuto |
| Access token | 15 min |
| Refresh token | 30 días |
| Cookie de invitado | 30 días |
| Chunk de subida de geotagging | 16 MiB |
| Subidas abandonadas de geotagging | 24 h |
| Zoom del DEM de calidad | 13 (~19 m/px) |
| Teselas DEM máximas por análisis | 64 |
| Caché DEM | `data/dem-cache` |
| Outbox dispatch | cada 30 s |
| Sondeo de efemérides en espera | cada 15 min |
| Limpieza de entregas | cada hora |
| Limpieza de auditoría de workers | diario 04:00 UTC |
| Retención de auditoría de workers | 30 días |
| Backup de PostgreSQL | diario 03:00 UTC si está habilitado |

> **Importante:** estos valores pueden cambiar según el `.env` del backend. La
> interfaz muestra la cuota que reporta la API en cada momento.

### 9.0 Nombre de la plataforma

El nombre visible se configura con `APP_NAME` en el `.env` del backend y con
`VITE_APP_NAME` / `VITE_APP_NAME_ACCENT` en el frontend. El acento
(`VITE_APP_NAME_ACCENT`, por defecto `FIX`) se pinta en naranja; el resto en
crema. Los identificadores técnicos (`geohub`, base de datos, bucket, cookies)
no cambian.

### 9.1 Cuándo un procesamiento supera el tiempo

- PPK dispone de 30 minutos de RTKLIB; PPP y estático, 60 minutos.
- Al superarlos, el trabajo se marca `failed` con el motivo y **no se
  reintenta**, para no duplicar horas de cómputo.
- El límite duro de Celery es de 2 horas: el trabajo se corta y se cierra.
- Si el corte fue por una caída del worker y no por timeout, el trabajo se
  retoma automáticamente al superar 130 minutos sin señales de actividad.

---

## 10. Estados y flujos asíncronos

### 10.1 Flujo de efemérides

```text
Usuario crea solicitud
  → API guarda solicitud + evento outbox
  → Beat publica evento en RabbitMQ
  → Worker resuelve fuentes
  → Descarga / descomprime / checksum / MinIO
  → Solicitud ready
  → Evento delivery.package
  → Worker genera ZIP y URL firmada
```

### 10.2 Flujo PPK

```text
Usuario encola PPK
  → API guarda job + evento outbox
  → Worker descarga rover/base de MinIO
  → Ejecuta rnx2rtkp
  → Parsea .pos
  → Calcula CTM12 / geoide
  → Guarda .pos / report.json / trajectory.geojson
  → Job ready
  → Evento delivery.package
```

### 10.3 Qué hacer si algo queda pendiente

1. Revisa el estado en la interfaz.
2. Comprueba `/health/queue` (RabbitMQ).
3. Revisa que worker y beat estén activos:
   ```bash
   docker ps --format '{{.Names}} {{.Status}}' | grep geohub-backend
   ```
4. Revisa los logs:
   ```bash
   docker logs -f geohub-backend-worker-1
   docker logs -f geohub-backend-beat-1
   ```
5. Revisa `worker_audit_events` para el recurso.
6. Si el producto aún no está publicado, espera el sondeo automático.
7. Si el error persiste, comparte el `request_id`, el ID del recurso y el
   detalle técnico con el administrador.

---

## 11. Errores y solución de problemas

### 11.1 Códigos HTTP

| HTTP | Significado general |
|---|---|
| `200` | Operación correcta. |
| `201` | Recurso creado. |
| `204` | Operación correcta sin contenido. |
| `400` | Petición mal formada. |
| `401` | Sesión ausente, inválida o expirada. La app renueva el token automáticamente y repite la operación. |
| `403` | Sin permisos o CSRF inválido. |
| `404` | Recurso no encontrado o no visible. Si aparece en una acción sobre tus archivos, recarga y vuelve a iniciar sesión. |
| `409` | Conflicto de estado (por ejemplo, trabajo aún no listo). |
| `413` | Archivo demasiado grande. |
| `415` | Formato no soportado. |
| `422` | Validación de datos fallida. |
| `429` | Cuota o rate limit excedido. |
| `500` | Error interno. |
| `502` | Falla de una fuente externa o motor RTKLIB. |
| `503` | Servicio no disponible (MinIO, RabbitMQ, OAuth). |
| `504` | Timeout de procesamiento. |

### 11.2 Errores de autenticación

| Código | Causa | Solución |
|---|---|---|
| `auth.invalid_credentials` | Correo o contraseña incorrectos. | Revisa los datos. |
| `auth.email_exists` | El correo ya está registrado. | Inicia sesión o usa otro correo. |
| `auth.rate_limited` | Demasiados intentos. | Espera la ventana. |
| `auth.invalid_access_token` | Access token inválido o expirado. | La app intenta refresh; si falla, vuelve a iniciar sesión. |
| `auth.invalid_refresh_token` | Refresh inválido o ausente. | Inicia sesión de nuevo. |
| `auth.refresh_reuse` | Reutilización de refresh revocado. | Por seguridad se revoca la familia; inicia sesión. |
| `auth.csrf_failed` | Falta o no coincide el token CSRF. | Recarga la página; verifica cookies/proxy. |
| `auth.oauth_not_configured` | Proveedor OAuth no configurado. | Usa correo/contraseña o configura OAuth. |
| `auth.forbidden` | Rol insuficiente. | Usa una cuenta administradora. |
| `auth.inactive_user` | Cuenta desactivada. | Contacta al administrador. |

### 11.3 Errores de red y navegador

- **No pudimos conectar con el servidor:** API caída, URL incorrecta o túnel
  cerrado.
- **La solicitud expiró:** CSRF desincronizado o cookies de otro origen.
  Recarga y verifica que uses el proxy de Vite.
- **401/403 en bucle:** revisa cookies, `CORS_ORIGINS`, `COOKIE_DOMAIN`,
  `COOKIE_SAMESITE` y que el frontend esté en `localhost:5173`.
- **Mapa sin teselas:** falta conexión a OpenStreetMap o la CSP bloquea el
  origen.
- **Descarga no disponible:** URL firmada expirada o MinIO caído.

### 11.4 Diagnóstico con request ID

Cada respuesta incluye `X-Request-ID`. Úsalo para buscar en logs:

```bash
docker logs geohub-backend-api-1 2>&1 | grep '<request-id>'
```

En errores 500, la respuesta incluye `request_id`. Compártelo con el
administrador; no compartas cookies ni tokens.

---

## 12. Productos, formatos y sistemas de referencia

### 12.1 Productos de efemérides

| Producto | Descripción |
|---|---|
| Órbitas/relojes | Productos principales según el tipo seleccionado. |
| CLK | Correcciones de reloj. |
| ERP | Parámetros de orientación terrestre. |
| ATX | Calibración de antenas. |
| IONEX | Mapas de contenido total de electrones. |

Los nombres y prefijos dependen de la fuente y del tipo de producto.

### 12.2 RINEX

- RINEX 2, 3 y 4 de observación.
- Hatanaka y compresión normalizados a un RINEX legible.
- Se conserva el original y el normalizado.

### 12.3 Salidas PPK

| Archivo | Contenido |
|---|---|
| `solution.pos` | Solución RTKLIB en formato `.pos`. |
| `report.json` | Métricas, conteos, media, CTM12, geoide y metadatos. |
| `trajectory.geojson` | `FeatureCollection` con `LineString`, `qualities` y `timestamps`. |
| `manifest.json` | Índice del paquete ZIP. |

### 12.4 Sistemas de referencia

- **WGS84 / EPSG:4326**: coordenadas geográficas del mapa y trayectoria.
- **MAGNA-SIRGAS Origen Nacional CTM12 / EPSG:9377**: coordenadas proyectadas
  oficiales de Colombia.
- **QGeoidCOL2023**: modelo cuasi-geoidal del IGAC para ondulación y altura
  ortométrica.
  - Licencia CC BY 4.0.
  - Resolución IGAC 0515 de 2026.
  - Cobertura geográfica de Colombia.

### 12.5 Checksums y almacenamiento

- Cada producto RINEX y cada paquete ZIP tiene SHA-256.
- Rutas de almacenamiento:
  ```text
  ephemeris/{request_id}/{filename}
  rinex/{file_id}/original/{filename}
  rinex/{file_id}/{normalized_filename}
  postprocessing/{job_id}/solution.pos
  postprocessing/{job_id}/report.json
  postprocessing/{job_id}/trajectory.geojson
  delivery/{resource_type}/{resource_id}/{filename}
  backups/postgres/geohub-YYYYMMDDTHHMMSSZ.dump
  ```

---

## 13. Consideraciones técnicas e información importante

### 13.1 Fechas y zonas horarias

- Todas las ventanas de efemérides se interpretan en **UTC**.
- El frontend envía días completos: `00:00:00Z` a `23:59:59Z`.
- La fecha final debe estar en el pasado; no se pueden pedir días futuros.
- La latencia del producto se calcula desde el cierre de la ventana.

### 13.2 Procesamiento asíncrono

- Las solicitudes y los trabajos no se ejecutan en la petición HTTP.
- La API responde rápido y el worker procesa en segundo plano.
- Los estados `requested`, `waiting_publication`, `queued`, `running`,
  `resolving` y `downloading` son temporales.
- Los reintentos son automáticos para fallos transitorios.
- Si un producto no está publicado, el sondeo lo reintenta más tarde.

### 13.3 Dependencias externas

El correcto funcionamiento depende de:

- PostgreSQL/PostGIS;
- RabbitMQ;
- MinIO/S3;
- acceso a las fuentes de efemérides (CDDIS, BKG, IGN, ESA, GFZ, CODE);
- RTKLIB instalado en la imagen del worker;
- conectividad para las teselas de OpenStreetMap.

Si MinIO o RabbitMQ fallan, el frontend puede mostrar errores 503 o los trabajos
pueden quedar pendientes. Revisa `/health/queue`, `/health/ready` y los logs.

### 13.4 Sesiones y mismo origen

- En desarrollo, el frontend y la API deben compartir origen mediante el proxy
  de Vite.
- El frontend usa `credentials: 'include'`, CSRF y refresh coordinado.
- Si despliegas frontend y API en orígenes distintos, configura:
  - `CORS_ORIGINS`;
  - `COOKIE_DOMAIN`;
  - `COOKIE_SAMESITE=none`;
  - `COOKIE_SECURE=true`;
  - HTTPS.
- No uses `localStorage` para tokens.

### 13.5 Ownership y datos

- No hay multitenancia: cada recurso pertenece a un `owner_id` de usuario o de
  invitado.
- Un invitado no puede ver recursos de otro invitado.
- Al crear una cuenta, el historial de invitado no se migra automáticamente.
- La retención de entregas es limitada; descarga lo que necesites conservar.

### 13.6 Privacidad y observabilidad

- Sentry se inicializa solo si hay DSN.
- El scrubber elimina cuerpos, cookies, cabeceras, query strings, correos y
  tokens.
- El usuario en Sentry se reduce a ID y rol.
- Los logs incluyen `request_id`; no incluyen contraseñas ni tokens.
- `/metrics` debe restringirse por red o proxy en producción.

### 13.7 Backups

- La tarea `maintenance.backup` no está habilitada por defecto.
- Para activarla, define `BACKUP_ENABLED=true` y revisa la retención.
- El backup se sube a `BACKUP_STORAGE_PREFIX` en MinIO/S3.
- Un backup no se considera restaurable hasta probar la restauración en una
  base vacía.
- Consulta [`../infra/runbook.md`](../infra/runbook.md) para el procedimiento.

### 13.8 Rendimiento

- Usa ventanas cortas y evita pedir productos innecesarios.
- Las solicitudes de invitado tienen cuota para proteger el servicio.
- El rate limit global es una primera barrera; en varias réplicas se necesita
  un limitador compartido.
- Las teselas del mapa y las descargas dependen de la red.

---

## 14. Limitaciones conocidas

Esta sección describe lo que **no** está implementado o tiene alcance parcial.

### 14.1 Cuentas y autenticación

- No hay recuperación de contraseña.
- No hay flujo de verificación de correo.
- No hay MFA.
- No hay gestión de sesiones activas desde la interfaz.
- No hay eliminación de cuenta desde la interfaz.

### 14.2 Efemérides

- No se pueden cancelar solicitudes desde la interfaz (el estado existe en el
  dominio, pero no hay endpoint público de cancelación).
- No hay filtros avanzados en el historial (solo paginación).
- No hay descarga directa de productos individuales desde la interfaz; se usan
  las entregas ZIP o la API.
- La disponibilidad depende de las fuentes externas.
- No hay suscripciones recurrentes ni programación de solicitudes.

### 14.3 RINEX

- Se aceptan observación y navegación; no se permite subir varios archivos en una sola operación.
- No hay carga por lote ni múltiples archivos en una sola operación.
- No hay descarga del RINEX original/normalizado desde la interfaz; solo
  metadatos y uso interno para PPK.
- No hay edición de metadatos ni re-normalización manual.
- La cuota de invitado es limitada.

### 14.4 PPK

- No hay procesamiento en tiempo real; es postproceso.
- No hay selección manual de estación CORS: la automática elige la más cercana
  dentro de `CORS_MAX_DISTANCE_KM` (20 km por defecto).
- No hay edición de opciones RTKLIB por el usuario: el motor y sus máscaras son
  fijos y se muestran en el modal antes de encolar.
- El control de base requiere un trabajo estático o PPP estático sobre el RINEX
  de la estación; no se genera automáticamente.
- No hay carga de coordenadas de base manual.
- No hay trabajos por lote.
- No hay comparación de múltiples soluciones.
- El informe PDF se genera bajo demanda desde **QA de vuelo**.
- El geoide QGeoidCOL2023 cubre Colombia; fuera de cobertura puede fallar.

### 14.5 Entregas

- La retención es limitada (48 h por defecto).
- Las URLs firmadas expiran.
- No hay historial ilimitado de paquetes.
- El correo depende de SMTP configurado.

### 14.6 Administración

- La interfaz de administración es de lectura.
- Los cambios de rol/estado se hacen por API.
- La auditoría de workers no está expuesta en la interfaz.

### 14.7 Operación

- El rate limit es en memoria.
- No hay CI/CD configurado en el repositorio.
- No hay despliegue con TLS/reverse proxy incluido; se documenta.
- Los túneles de MinIO/RabbitMQ/PostGIS son externos y pueden ser inestables.
- Sentry requiere configuración de DSN.
- Los backups están deshabilitados por defecto.

### 14.8 Producto

- No hay pagos, suscripciones ni donaciones implementadas.
- No hay multi-idioma; la interfaz está en español.
- No hay modo offline.
- No hay aplicación móvil.

---

## 15. Preguntas frecuentes

### ¿Por qué mi solicitud está en `waiting_publication`?

Porque el producto aún no está publicado por la fuente. El sistema la
reintentará cuando venza la fecha estimada. Puedes activar el aviso por correo.

### ¿Por qué no puedo crear más solicitudes?

Probablemente agotaste la cuota diaria de invitado. Espera la ventana o crea una
cuenta.

### ¿Por qué mi PPK falló?

Revisa el detalle técnico. Causas comunes:

- rover y base no cubren la misma ventana;
- falta posición aproximada para CORS;
- no hay CORS cercana;
- el RINEX está corrupto;
- RTKLIB excedió el timeout;
- MinIO no estaba disponible.

### ¿Por qué no veo el mapa?

- El trabajo aún no está `ready`.
- No hay trayectoria generada.
- MinIO no está disponible.
- No hay conexión a OpenStreetMap.
- La CSP bloquea el origen de tiles (revisa la configuración de build).

### ¿Dónde está mi ZIP?

En `/deliveries`. Si el trabajo está `ready`, aparece el enlace de descarga. Si
expiró, solicita una nueva entrega.

### ¿Cuánto tiempo se guardan los archivos?

Las entregas 48 h por defecto. Los objetos de efemérides/RINEX/PPK dependen de la
retención configurada por el operador. Descarga lo que necesites conservar.

### ¿Puedo usar mi propia base?

Sí. En el formulario PPK elige **RINEX propio** y selecciona el archivo base.

### ¿Qué significa Fix / Float / Single?

- **Fix**: solución con ambigüedades resueltas; máxima confianza.
- **Float**: ambigüedades no resueltas; menor confianza.
- **Single**: solución autónoma; menor precisión.

### ¿Qué es CTM12?

Es la proyección oficial de Colombia **MAGNA-SIRGAS Origen Nacional**
(EPSG:9377). El reporte incluye Este y Norte.

### ¿Qué es la altura ortométrica?

Es la altura sobre el nivel medio del mar. Se calcula restando la ondulación
geoidal a la altura elipsoidal usando QGeoidCOL2023.

### ¿Por qué me pide iniciar sesión otra vez?

La sesión expiró y el refresh falló. Puede ocurrir si cambiaste de red, borraste
cookies o el backend reinició la clave. Inicia sesión de nuevo.

### ¿Cómo me hago administrador?

Un administrador existente debe cambiar tu rol, o se puede usar el script
`promote_admin` con acceso al backend.

### ¿Puedo usar la API sin la interfaz?

Sí. La documentación interactiva está en `http://127.0.0.1:8000/docs`.

---

## 16. Glosario

| Término | Significado |
|---|---|
| **GNSS** | Sistemas globales de navegación por satélite. |
| **Efemérides** | Información de órbitas y relojes de satélites. |
| **RINEX** | Formato estándar de observaciones GNSS. |
| **Hatanaka** | Compresión diferencial de RINEX. |
| **PPK** | Post-Processed Kinematic. |
| **RTKLIB** | Suite de software GNSS usada para PPK. |
| **CORS** | Estación de referencia de operación continua. |
| **Fix** | Ambigüedades resueltas. |
| **Float** | Ambigüedades no resueltas. |
| **Single** | Solución autónoma. |
| **WGS84** | Sistema geodésico global. |
| **MAGNA-SIRGAS** | Marco geodésico oficial de Colombia. |
| **CTM12** | Proyección MAGNA-SIRGAS Origen Nacional. |
| **Geoide** | Superficie equipotencial usada para altura ortométrica. |
| **QGeoidCOL2023** | Modelo cuasi-geoidal de Colombia. |
| **CLK** | Producto de relojes. |
| **ERP** | Parámetros de orientación terrestre. |
| **ATX** | Calibración de antenas. |
| **IONEX** | Mapas ionosféricos. |
| **Outbox** | Patrón transaccional para publicar eventos. |
| **Celery** | Sistema de colas de tareas. |
| **MinIO** | Almacenamiento de objetos compatible con S3. |
| **URL firmada** | Enlace temporal con permisos de descarga. |
| **SHA-256** | Hash criptográfico de integridad. |

---

## 17. Protección antiabuso

- El registro y el login pueden exigir **Cloudflare Turnstile** si el
  administrador configura `TURNSTILE_ENABLED`. El widget es invisible en login
  y visible en registro.
- Los invitados tienen una cuota diaria por sesión y otra por IP. Si se agota,
  crea una cuenta o espera a la siguiente ventana.
- `/metrics` y `/docs` deben restringirse en producción (`METRICS_TOKEN` y
  `API_DOCS_ENABLED=false` o proxy de red).

## 18. Soporte y operación

### 17.1 Documentación relacionada

- `README.md` del backend: configuración, API y migraciones.
- `README.md` del frontend: desarrollo, sesión, CSP y E2E.
- [`../infra/runbook.md`](../infra/runbook.md): operación, logs, backups, restauración y fallos comunes.
- `ROADMAP.md`: estado de las fases.
- `http://127.0.0.1:8000/docs`: OpenAPI interactivo.

### 17.2 Antes de pedir soporte

Incluye:

- `request_id` de la respuesta;
- ID del recurso (solicitud, archivo o trabajo);
- hora aproximada;
- acción realizada;
- mensaje visible;
- captura o detalle técnico si está disponible.

No compartas:

- contraseñas;
- cookies;
- tokens;
- DSNs;
- credenciales de MinIO/RabbitMQ/PostgreSQL.

### 17.3 Health checks

```bash
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/health/ready
curl http://127.0.0.1:8000/health/queue
curl http://127.0.0.1:8000/metrics
```

### 17.4 Comandos útiles

```bash
# Servicios
docker compose -f geohub-backend/docker-compose.dev.yml up -d --build

# Logs
docker logs -f geohub-backend-api-1
docker logs -f geohub-backend-worker-1
docker logs -f geohub-backend-beat-1

# Frontend
cd geohub/geohub-frontend
pnpm dev

# Tests backend
docker run --rm --network host --env-file geohub-backend/.env \
  -v "$PWD/geohub-backend":/app -w /app geohub-backend:test \
  pytest -q

# Tests frontend
cd geohub/geohub-frontend
pnpm lint
pnpm build
pnpm test
pnpm test:e2e
```

---

## 19. Anexo: API rápida

Todas las rutas usan el prefijo `/api/v1`.

### Autenticación

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/auth/register` | Registrar usuario. |
| POST | `/auth/login` | Iniciar sesión (form-urlencoded). La sesión web usa cookies HttpOnly. |
| POST | `/auth/token` | Emitir un access token para clientes máquina (sin cookies). |
| POST | `/auth/refresh` | Rotar sesión. |
| POST | `/auth/logout` | Cerrar sesión. |
| GET | `/auth/me` | Usuario actual. |
| GET | `/auth/oauth/{provider}/authorize` | URL de OAuth. |
| GET | `/auth/oauth/{provider}/callback` | Callback OAuth. |

### Efemérides

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/ephemeris/requests` | Crear solicitud. |
| GET | `/ephemeris/requests` | Listar solicitudes. |
| GET | `/ephemeris/requests/{id}` | Consultar solicitud. |
| GET | `/ephemeris/requests/{id}/products` | Listar productos. |
| POST | `/ephemeris/requests/{id}/resolve` | Resolver manualmente (síncrono). |
| GET | `/ephemeris/quota` | Cuota de invitado. |
| GET | `/ephemeris/sources` | Estado de fuentes. |

### RINEX

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/rinex/files` | Subir RINEX (multipart). |
| GET | `/rinex/files` | Listar archivos. |
| GET | `/rinex/files/{id}` | Consultar archivo. |
| GET | `/rinex/quota` | Cuota de invitado. |

### Postprocesamiento

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/postprocessing/jobs` | Encolar trabajo PPK. |
| GET | `/postprocessing/jobs` | Listar trabajos. |
| GET | `/postprocessing/jobs/{id}` | Consultar trabajo. |
| GET | `/postprocessing/jobs/{id}/trajectory` | GeoJSON de trayectoria. |
| GET | `/postprocessing/jobs/{id}/points` | Puntos GeoJSON/KML (`format`, `quality`). |

### Estaciones

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/stations` | Catálogo con `search`, `network`, `active`, `limit`, `offset`. |
| GET | `/stations/summary` | Totales por estado y red. |
| GET | `/stations/{id}` | Detalle de una estación. |

### Geotagging

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/geotagging/batches` | Abrir el lote y la sesión de subida. |
| PUT | `/geotagging/batches/{id}/mrk` | Subir el MRK del vuelo. |
| PUT | `/geotagging/batches/{id}/archive/parts/{n}` | Subir una parte del ZIP. |
| POST | `/geotagging/batches/{id}/images` | Subir imágenes sueltas (ZIP en servidor). |
| POST | `/geotagging/batches/{id}/complete` | Validar y encolar el lote. |
| POST | `/geotagging/batches/{id}/cancel` | Cancelar una subida en curso. |
| POST | `/geotagging/batches/{id}/retry` | Reintentar un lote fallido. |
| GET | `/geotagging/batches` | Listar lotes; filtro `processing_job_id`. |
| GET | `/geotagging/batches/{id}` | Estado, progreso y partes recibidas. |
| GET | `/geotagging/batches/{id}/geotags` | Geoetiquetas `txt` o `csv`. |
| GET | `/geotagging/batches/{id}/report` | Reporte JSON. |
| GET | `/geotagging/batches/{id}/images` | URL firmada del ZIP con EXIF. |

### Calidad

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/quality/flights` | Lotes listos con disponibilidad de cámara. |
| GET | `/quality/flights/{id}` | Métricas de vuelo; acepta `ground_height_m`. |
| GET | `/quality/flights/{id}/report.pdf` | Informe PDF de calidad. |
| GET | `/quality/base/{job_id}` | Control de la estación base. |
| POST | `/quality/transform` | Transformación WGS84 ↔ CTM12. |

### Entregas

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/delivery/packages` | Listar paquetes. |
| GET | `/delivery/packages/{id}` | Paquete + URL firmada. |

### Administración

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/admin/users` | Listar usuarios. |
| PATCH | `/admin/users/{id}/role` | Cambiar rol. |
| POST | `/admin/users/{id}/activate` | Activar. |
| POST | `/admin/users/{id}/deactivate` | Desactivar. |
| GET | `/admin/audit-events` | Auditoría de autenticación. |

### Salud

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Liveness. |
| GET | `/health/ready` | PostgreSQL/PostGIS. |
| GET | `/health/queue` | RabbitMQ. |
| GET | `/metrics` | Métricas Prometheus. |

---

> **Fin del manual.** Para cambios de configuración, límites o credenciales,
> consulta al operador del backend. La plataforma es una herramienta de apoyo;
> valida siempre los resultados geodésicos antes de usarlos en producción.
