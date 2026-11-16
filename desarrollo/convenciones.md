# Convenciones

- Código e identificadores en inglés; documentación, errores de dominio y UI en
  español.
- Arquitectura hexagonal; dependencias hacia dentro.
- `ruff check` y `ruff format --check` obligatorios.
- Tests para reglas de dominio, casos de uso y adaptadores críticos.
- Migraciones con `downgrade`.
- Sin secretos en logs ni repositorio.
- Fechas UTC; tiempo GPS modelado como value object.
- Errores tipados con `code` estable para la API.
- Commits pequeños por fase y con documentación actualizada.
