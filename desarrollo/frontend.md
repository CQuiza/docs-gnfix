# Frontend

Ver también:

- [`geohub-frontend/README.md`](../../geohub-frontend/README.md)
- [Manual de usuario](../usuario/manual-usuario.md)

## Stack

React + TypeScript + Vite + Tailwind. Leaflet para mapas. Vitest + Testing
Library y Playwright para E2E.

## Comandos

```bash
cd geohub/geohub-frontend
pnpm lint
pnpm build
pnpm test
pnpm test:e2e
```

## Identidad

- Carbón `#0E1013`, titanio `#262930`, crema `#F4F0EA`, naranja `#FF5500`.
- Orbitron para títulos/marca; JetBrains Mono para datos y RINEX.
- UI en español.

## Rutas

| Ruta | Página |
|---|---|
| `/home` | Landing pública (producto, FAQ y contacto) |
| `/` | Panel (redirige a `/home` sin sesión) |
| `/ephemeris` | Historial de efemérides |
| `/ephemeris/new` | Nueva solicitud |
| `/rinex` | Archivos RINEX |
| `/postprocessing` | Trabajos PPK/PPP |
| `/postprocessing/:jobId` | Detalle, trayectoria, puntos y geotagging |
| `/geotagging` | Historial centralizado de lotes (polling permanente) |
| `/activity` | Procesos activos y recientes (polling permanente) |
| `/quality` | QA de vuelo, control de base y transformación |
| `/stations` | Catálogo de estaciones |
| `/deliveries` | Entregas |
| `/manual` | Manual |
| `/admin` | Administración |

## Subida por partes de geotagging

- `GeotaggingPanel` crea el lote (`POST /geotagging/batches`) y después sube el
  MRK y el ZIP en partes de 16 MiB con `XMLHttpRequest`.
- El progreso se deriva de `uploaded_bytes`/`total_bytes` del servidor, no del
  XHR, así que sobrevive a recargas.
- Si el lote queda en `uploading`, la tarjeta muestra **Subida incompleta** y
  habilita **Reanudar subida** cuando se vuelve a seleccionar el mismo ZIP.
- `beforeunload` avisa mientras hay una subida activa.
- `/geotagging` y `/activity` refrescan siempre (cada 4–5 s) mientras la
  pestaña está visible y al volver a ella.

## Selección de RINEX

- `/rinex` muestra **Archivo procesado** con los metadatos del archivo
  seleccionado.
- La tabla **Archivos cargados** es interactiva: clic o Enter sobre una fila
  actualiza la tarjeta. No está limitada al último archivo subido.
- Tras una subida, el archivo recién procesado queda seleccionado.

## Sesión y errores

- Un visitante sin sesión es `anonymous`: `/` lo lleva a `/home` y las rutas
  profundas a `/login`. `guest` solo se activa con **Explorar como invitado**
  y se conserva en `sessionStorage` durante la pestaña.
- La landing lee `VITE_CONTACT_EMAIL` para los enlaces `mailto:`.
- `apiFetch` renueva el access token ante un 401 y repite la petición.
- La API responde 401 cuando existe cookie de refresh sin access; así no se
  degrada la sesión a invitado.
- `toUserMessage` usa el mensaje de dominio del API para `rinex.*`,
  `postprocessing.*`, `geotagging.*`, `stations.*`, `delivery.*` y `quality.*`.

## Marca configurable

- `VITE_APP_NAME` y `VITE_APP_NAME_ACCENT` definen el nombre y la parte que se
  pinta en naranja.
- `<BrandName />` renderiza el prefijo en crema y el acento en `text-brand-500`;
  se usa en el logo, login y layout.
