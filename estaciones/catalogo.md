# Catálogo de estaciones

GNFIX usa el catálogo RINEX de la Red Geodésica Nacional del IGAC.

## Fuentes

- `GET https://ccg.igac.gov.co/api/estaciones` — identificador, `t_id`, sigla,
  estado.
- `GET https://redgeodesica-cg.igac.gov.co:8080/api/rastreo/estaciones` —
  coordenadas, municipio, estado y domo.
- `GET https://ccg.igac.gov.co/api/rinex?id=<t_id>` — archivos disponibles.

## Campos

| Campo | Uso |
|---|---|
| `code` | Identificador de estación (`3MAT`, `BOGA`, ...) |
| `name` | Municipio o nombre |
| `network` | IGAC, SGC, EAAB, ... |
| `latitude`, `longitude` | WGS84 |
| `height_m` | Altura elipsoidal; puede ser 0 si no está publicada |
| `external_id` | `t_id` del IGAC |
| `is_active` | Estado del catálogo IGAC |
| `last_synced_at` | Última actualización del catálogo |

## Importación

```bash
cd geohub/geohub-backend
python -m geohub.scripts.import_cors_stations --igac
```

El importador hace upsert por `code`. No borra estaciones existentes.

## Uso en postprocesamiento

- Auto-CORS busca la estación activa más cercana dentro de
  `CORS_MAX_DISTANCE_KM` (20 km).
- La navegación broadcast se obtiene de la estación IGAC más cercana.
- La observación y la navegación se descargan en un solo POST.

Ver [operación](operacion.md).
