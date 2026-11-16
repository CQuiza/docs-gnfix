# Referencia de la API

La referencia generada está en:

- Swagger UI: `http://127.0.0.1:8000/docs`
- OpenAPI JSON: `http://127.0.0.1:8000/openapi.json`

## Grupos

| Grupo | Prefijo | Uso |
|---|---|---|
| Salud | `/health`, `/health/ready`, `/health/queue` | Estado de API, base y cola |
| Auth | `/api/v1/auth` | Registro, login, refresh, logout, OAuth |
| Efemérides | `/api/v1/ephemeris` | Solicitudes, productos, cuotas |
| Fuentes | `/api/v1/sources` | Salud de fuentes IGS |
| RINEX | `/api/v1/rinex` | Subida, listado, cuota |
| Postprocesamiento | `/api/v1/postprocessing` | Jobs PPK/PPP, trayectoria y puntos |
| Geotagging | `/api/v1/geotagging` | Lotes de imágenes, MRK, geoetiquetas y EXIF |
| Estaciones | `/api/v1/stations` | Catálogo CORS público |
| Entregas | `/api/v1/delivery` | Paquetes ZIP y URLs firmadas |
| Admin | `/api/v1/admin` | Usuarios, auditoría y refresco de catálogo |
| Métricas | `/metrics` | Prometheus |
