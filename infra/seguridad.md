# Seguridad

- Cookies HttpOnly para access/refresh/guest.
- CSRF obligatorio en métodos mutables.
- Rate limits de login, registro, refresh y API.
- Argon2id para contraseñas.
- Salida de RTKLIB en subproceso sin shell, con timeout y entorno mínimo.
- ExifTool se ejecuta sin shell, con timeout y lista de argumentos.
- Los ZIP de imágenes se validan por extensión, firma y tamaño descomprimido
  antes de extraer; se bloquean rutas absolutas y `..`.
- URLs firmadas de MinIO con expiración.
- Cabeceras de seguridad, CSP y HSTS en producción.
- Scrubbing de Sentry y logs.

## Checklist

- [ ] `SECRET_KEY` de al menos 32 caracteres y fuera del repositorio.
- [ ] `COOKIE_SECURE=true` y HTTPS.
- [ ] `TRUSTED_HOSTS` y `CORS_ORIGINS` restringidos.
- [ ] `/docs` y `/metrics` no expuestos públicamente.
- [ ] Credenciales CDDIS/SMTP/MinIO solo por entorno.
- [ ] Backups cifrados y con retención.
- [ ] `pip-audit` y `pnpm audit` revisados.

## Controles de sesión

- `access_token` solo viaja en cookie `HttpOnly`; `/auth/token` es la vía para
  clientes máquina.
- El `state` OAuth se ata al navegador con una cookie `HttpOnly` y `SameSite=Lax`.
- Un refresh rotado se acepta durante `refresh_rotation_grace_seconds` solo si
  coinciden UA/IP; fuera de la ventana se revoca la familia.
- Logout borra access, refresh, CSRF y la cookie de invitado.
- Las cuentas locales no verificadas no se vinculan automáticamente a OAuth.

## Antiabuso y verificación

- Turnstile se valida en el servidor (`siteverify`) con acción y hostname; el
  token no se registra y el secreto solo vive en el backend.
- Registro y login rechazan peticiones de navegador cross-site aunque no usen
  el token CSRF (comprobación de `Origin`/`Sec-Fetch-Site`).
- La cuota de invitado se cuenta por cookie **y** por IP con una tabla atómica.
- Los correos no verificados no se vinculan automáticamente a OAuth.
- `/metrics` requiere token en producción (o loopback) y `/docs` puede
  deshabilitarse con `API_DOCS_ENABLED=false`.
