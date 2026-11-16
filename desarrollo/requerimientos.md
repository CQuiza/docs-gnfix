# Requerimientos y Funcionalidades del Sistema (GNFIX)

## 1. Requerimientos y Funcionalidades del Frontend (UI/UX)

El frontend estará orientado a la usabilidad inmediata sin fricción operativa (módulos limpios, reactivos y enfocados en el flujo geomático).

### A. Módulo de Autenticación y Gestión de Usuarios (`Auth & User Profile`)
* **Acceso Híbrido:** Permite navegación/uso anónimo (modo invitado con límites) y registro mediante formulario o proveedores OAuth (Google/GitHub).
* **Dashboard de Usuario:**
  * Historial de solicitudes (descargas de efemérides y ejecuciones de postprocesamiento).
  * Consumo de cuotas (contador de créditos/proyectos procesados en la capa gratuita).
  * Sección de donaciones / soporte al proyecto (vías Stripe / Open Collective / Buy Me a Coffee).

### B. Módulo de Solicitud Manual y Asistida de Efemérides (`Ephemeris Manager UI`)
* **Ingreso Directo de Fechas:** Calendario con selector de rango de fechas (UTC) y hora de inicio/fin.
* **Filtros Geodésicos:**
  * **Constelaciones:** Casillas de selección para GPS, GLONASS, Galileo, BeiDou.
  * **Nivel de Precisión:** Selección entre Efemérides Finales, Rápidas (*Rapid*) y Ultra-Rápidas (*Ultra-Rapid*).
  * **Insumos Adicionales:** Casillas opcionales para solicitar Relojes (`.clk`), Parámetros de Orientación Terrestre (`.erp`), Calibración de Antenas (`.atx`) y Datos Ionosféricos (`.IONEX`).
* **Delivery & Alerta:** Opción de envío por correo electrónico o generación de enlace efímero, junto con la casilla *"Avisarme cuando esté disponible"* si la efeméride precisa final aún no se publica.

### C. Módulo Drag & Drop (Lectura Automática de RINEX / Logs)
* **Zona de Carga (File Uploader):** Componente para arrastrar archivos RINEX (`.obs`, `.24o`, `.crx`) o logs de drones/móviles.
* **Extracción Dinámica en Pantalla:** Tras procesar el encabezado en backend, despliega un resumen automático:
  * Ventana temporal calculada.
  * Constelaciones detectadas en el rastreo.
  * Coordenadas aproximadas de la base/estación.

### D. Módulo de Orquestación y Postprocesamiento (`PPK Execution UI`)
* **Presets por Software:** Menú desplegable para empaquetar resultados según el software de destino (*Preset Agisoft Metashape*, *Preset Pix4D*, *Preset RTKLIB/RTKPOST*, *Preset Emlid Studio*).
* **Selector de Referencia Local (Enfocado en Colombia y Suramérica):**
  * Opción para aplicar transformación al marco **MAGNA-SIRGAS (Origen Nacional CTM12 / Res. 370 IGAC)**.
  * Opción para aplicar la ondulación con el modelo **QGeoidCOL2023 / Geocol**.
* **Visor Geográfico Interactivo:** Mapa en tiempo real (Leaflet / Mapbox) para previsualizar el punto o la trayectoria procesada con códigos de colores según el tipo de solución (`Fix` = Verde, `Float` = Amarillo, `Single` = Rojo).

---

## 2. Requerimientos y Funcionalidades del Backend (API & Engines)

El backend aplicará arquitectura **Hexagonal + DDD + Vertical Slicing** estructurada en los bounded contexts/módulos principales.

### A. Módulo de Dominio Geodésico (`Ephemeris & Rinex Domain Module`)
* **Motor de Conversión Temporal:** Algoritmo puro de conversión: $\text{UTC Date/Time} \rightarrow \text{GPS Week, Day of Week (0-6), Day of Year (DOY)}$.
* **Parser de Encabezados RINEX:** Lectura y extracción asíncrona de metadatos de archivos de observación v2.x, v3.x y v4.x (fechas, épocas, constelaciones, intervalo, coordenadas iniciales).
* **Lógica de Fallback Multi-Fuente:** Conectores resilientes con reintentos para repositorios internacionales:
  1. NASA CDDIS (Conexión mediante protocolo HTTPS con OAuth / Earthdata Login).
  2. BKG (Servidor de la Agencia Federal Alemana de Cartografía).
  3. CODE / AIUB (Universidad de Berna).
  4. ESA / GFZ / IGN.
  * *Estrategia Fallback:* Si la fuente principal falla o no ha publicado el producto, consulta automáticamente la fuente secundaria informando el origen final del insumo.

### B. Módulo de Ejecución de Cálculo y Procesamiento (`PostProcessing Module`)
* **Integración Headless de RTKLIB (`rnx2rtkp`):**
  * Adaptador CLI que invoca de forma aislada y mediante subprocesos seguros la herramienta `rnx2rtkp` en el contenedor Docker.
  * Generación de archivos de configuración `.conf` dinámicos según los requerimientos del postproceso.
* **Motor de Transposición de Coordenadas y Geoides:**
  * Transformación de coordenadas ITRF/WGS84 a **MAGNA-SIRGAS (Origen Nacional CTM12)**.
  * Cálculo de altura ortométrica mediante interpolación con **QGeoidCOL2023**.
* **Integración con Redes CORS (SIRGAS-CON / MAGNA-ECO):**
  * Auto-Fetch de la estación CORS oficial más cercana a las coordenadas aproximadas del RINEX cargado y descarga automática de su archivo de observación `.obs`.

### C. Módulo de Tareas Asíncronas y Persistencia (`Infrastructure & Workers`)
* **Cola de Procesamiento (Celery + RabbitMQ / Redpanda):**
  * Orquestación asíncrona de descargas masivas, descompresión de formatos comprimidos (`.Z`, `.gz`, `.bz2`, `.crx` $\rightarrow$ `.rnx`) y ejecuciones de `rnx2rtkp`.
  * Sistema de reintentos con backoff exponencial cuando los servidores externos están caídos o sobrecargados.
* **Almacenamiento Efímero en MinIO (Object Storage S3):**
  * Empaquetado final en estructuras `.zip` personalizadas.
  * Generación de URLs firmadas temporales con ciclo de vida de retención (ejemplo: 48 horas).
* **Persistencia PostgreSQL + PostGIS:**
  * Guardado de metadatos de usuarios, registros de descargas, logs de errores y geometría de trayectorias/puntos procesados.
* **Servicio de Notificación SMTP:** Envío automático por correo electrónico cuando los paquetes o los resultados del postprocesamiento están listos.