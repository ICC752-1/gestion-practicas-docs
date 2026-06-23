# Casos de Prueba - Data Portability

## Alcance

Estos casos documentan las pruebas unitarias de valor del módulo `data_portability`. El foco está en exportación de datos personales del estudiante, minimización de campos sensibles y autorización por rol.

## Unitarias

### CU-U-DP-01: Exportación JSON minimiza campos sensibles

- Tipo de prueba: Unitaria
- Dominio: Data Portability
- Contexto: El estudiante puede exportar sus datos personales, pero no deben exponerse secretos internos ni rutas del servidor.
- Objetivo: Validar que la exportación JSON omite campos sensibles y registra auditoría completada.
- Escenario: Un estudiante solicita exportación JSON sin documentos físicos.
- Variantes cubiertas:
  - No aparece `password_hash`.
  - No aparecen tokens.
  - No se expone ruta interna de storage.
  - El correo del titular sí aparece.
  - La solicitud queda marcada como `completed`.
- Resultado esperado: El JSON contiene datos portables y minimizados.
- Valor de negocio: Cumple portabilidad sin filtrar secretos ni detalles de infraestructura.
- Pruebas automatizadas:
  - `tests/modules/data_portability/test_data_portability_service.py::test_export_json_minimizes_sensitive_user_fields`

### CU-U-DP-02: Exportación requiere rol estudiante

- Tipo de prueba: Unitaria
- Dominio: Data Portability
- Contexto: La portabilidad de datos personales se expone al titular estudiante.
- Objetivo: Validar que roles administrativos no usen este flujo para exportar datos como titulares.
- Escenario: Un usuario con rol `Director de carrera` intenta exportar datos personales por este endpoint.
- Variantes cubiertas:
  - Actor sin rol `Estudiante` recibe `403`.
- Resultado esperado: Solo estudiantes pueden usar la exportación personal.
- Valor de negocio: Evita abuso administrativo de un flujo de titularidad personal.
- Pruebas automatizadas:
  - `tests/modules/data_portability/test_data_portability_service.py::test_export_requires_student_role`
