# GNFIX — Política de Privacidad y Cookies

> **Última actualización:** 2026-11-18  
> **Responsable:** operador de la plataforma GNFIX  
> **Contacto:** crtquiza@gmail.com

GNFIX aplica un principio de **minimización de datos**: solo tratamos la información indispensable para autenticarte, procesar tus archivos y entregarte resultados. Esta política complementa los [Términos y Condiciones](/legal/terminos).

## 1. Datos que tratamos

- **Cuenta:** correo electrónico (obligatorio), nombre visible (opcional), rol, estado de la cuenta y fecha de registro. Si usas Google o GitHub, recibimos el correo y nombre público asociados a esa cuenta.
- **Seguridad y auditoría:** hash de la dirección IP, agente de navegador, identificadores de sesión, eventos de inicio de sesión y acciones privilegiadas. No almacenamos la IP en claro.
- **Archivos:** encabezados RINEX y metadatos EXIF (coordenadas aproximadas, parámetros de cámara) que el sistema lee para buscar estaciones CORS, calcular métricas y generar entregables.
- **Comunicaciones:** correos de verificación, restablecimiento de contraseña y avisos de disponibilidad que solicitas.
- **Errores:** si ocurre un fallo, enviamos a Sentry información técnica depurada (sin cuerpos de petición, cookies, tokens ni correos completos).

## 2. Finalidades y base legal

Tratamos los datos para:

1. Autenticar tu cuenta y mantener tu historial de trabajos.
2. Procesar tus archivos y generar los entregables solicitados.
3. Enviarte los enlaces o avisos que pides.
4. Aplicar cuotas y proteger la infraestructura frente a abuso o ataques.
5. Cumplir obligaciones legales y resolver reclamaciones.

La base es tu **consentimiento** al registrarte y aceptar esta política, y el **interés legítimo** en mantener la seguridad del servicio. Puedes retirar el consentimiento solicitando la eliminación de tu cuenta.

**No vendemos, alquilamos ni cedemos tu correo ni tus archivos a terceros, ni hacemos perfilado publicitario.**

## 3. Cookies

GNFIX no usa cookies de rastreo, analítica publicitaria ni publicidad de terceros. Solo emplea cookies técnicas necesarias:

- **`gnfix_access`** (`HttpOnly`, unos 15 minutos): mantiene la sesión sin exponer el token al JavaScript.
- **`gnfix_refresh`** (`HttpOnly`, unos 30 días): permite renovar la sesión.
- **`gnfix_csrf`** (legible por la aplicación): protege formularios y peticiones contra CSRF.
- **`gnfix_guest`** (hasta 30 días): asocia temporalmente tus solicitudes si usas el modo invitado.

Puedes borrarlas desde el navegador, pero perderás la sesión y el modo invitado.

## 4. Servicios de terceros

Interactuamos con proveedores estrictamente necesarios:

- **Google / GitHub:** solo si eliges OAuth; recibimos tu correo y nombre público.
- **Proveedor SMTP:** entrega los correos de verificación, restablecimiento y avisos; ve el contenido de esos mensajes.
- **Sentry:** monitoreo de errores con depuración de datos personales; puede implicar transferencia a Estados Unidos.
- **Almacenamiento S3/MinIO y hosting:** guardan archivos y resultados bajo tus permisos.
- **IGS, CDDIS, BKG, IGN, ESA, GFZ, CODE e IGAC:** consultas de efemérides, estaciones CORS y productos; no reciben tu correo ni tus archivos.
- **OpenStreetMap y AWS Elevation:** teselas de mapa y elevación para los informes; no reciben datos personales.

## 5. Retención

- Cuenta y datos asociados: mientras la cuenta esté activa o hasta que solicites su eliminación.
- Tokens de verificación de correo: 24 horas.
- Tokens de restablecimiento de contraseña: 60 minutos.
- Sesiones de refresh: hasta 30 días o hasta cerrar sesión.
- Paquetes de entrega y enlaces firmados: 48 horas.
- Lotes de geotagging: 168 horas (7 días).
- Registros de auditoría técnica: 30 días, salvo obligación legal de conservarlos más tiempo.

## 6. Tus derechos

Como titular puedes solicitar acceso, corrección, actualización, eliminación y revocatoria del consentimiento, así como conocer el uso de tus datos. Escríbenos a **crtquiza@gmail.com** indicando tu cuenta y la solicitud. También puedes presentar una reclamación ante la Superintendencia de Industria y Comercio (SIC) de Colombia.

## 7. Seguridad

Usamos HTTPS, cookies `HttpOnly`, protección CSRF, hash de contraseñas con Argon2id, hash de IP y acceso restringido a la infraestructura. Ningún sistema es infalible; si detectas un incidente, contáctanos de inmediato.

## 8. Menores y cambios

El servicio está dirigido a profesionales y personas con capacidad legal; no está pensado para menores de edad. Si cambiamos esta política de forma sustancial, publicaremos la nueva versión y podremos pedirte una nueva aceptación al iniciar sesión.

Para cualquier duda sobre privacidad escribe a **crtquiza@gmail.com**.
