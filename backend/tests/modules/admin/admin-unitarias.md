# Casos de Prueba - Admin

## Alcance

Estos casos documentan las pruebas unitarias de valor del módulo `admin`. El foco está en consultas administrativas, filtros del dashboard, reportes agregados, alcance por rol/carrera, seguro escolar por solicitud, requisito institucional histórico y contratos de respuesta usados por el frontend.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos de negocio verificables.

## Unitarias

### CU-U-AD-01: Resumen administrativo conserva totales y estados

- **Tipo de prueba:** Unitaria
- **Dominio:** Admin
- **Contexto:** El dashboard administrativo muestra totales globales y desglose de prácticas por estado.
- **Objetivo:** Confirmar que el service conserva conteos y traduce estados faltantes a una etiqueta legible.
- **Escenario:** El repositorio entrega conteos y agrupaciones por estado, incluyendo estado nulo.
- **Variantes cubiertas:**
  - Resumen con estados explícitos.
  - Resumen con estado nulo traducido a `Sin estado`.
- **Resultado esperado:** El resumen mantiene totales correctos y no expone estados nulos al contrato HTTP.
- **Valor de negocio:** Protege indicadores usados por coordinación para monitorear prácticas.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_service.py::test_get_summary_returns_summary`
  - `tests/modules/admin/test_admin_service.py::test_get_summary_maps_missing_status`

### CU-U-AD-02: Listados administrativos conservan datos relevantes

- **Tipo de prueba:** Unitaria
- **Dominio:** Admin
- **Contexto:** Coordinación consulta estudiantes y prácticas desde vistas administrativas.
- **Objetivo:** Validar que los listados conservan datos relevantes y toleran relaciones incompletas.
- **Escenario:** El repositorio devuelve estudiantes y prácticas con estudiante, estado, estado ausente o anulación.
- **Variantes cubiertas:**
  - Estudiante activo con datos personales completos.
  - Práctica con estudiante y estado relacionado.
  - Práctica anulada conserva marca `is_cancelled`.
  - Práctica sin estado retorna `status=None`.
- **Resultado esperado:** Los listados administrativos mantienen los campos necesarios para el dashboard.
- **Valor de negocio:** Evita pantallas administrativas con datos incompletos o inconsistentes.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_service.py::test_get_students_maps_students`
  - `tests/modules/admin/test_admin_service.py::test_get_internships_maps_related_data`
  - `tests/modules/admin/test_admin_service.py::test_get_internships_marks_cancelled_practices`
  - `tests/modules/admin/test_admin_service.py::test_get_internships_allows_none_status`

### CU-U-AD-03: Filtros normalizados del dashboard agrupan estados funcionales

- **Tipo de prueba:** Unitaria
- **Dominio:** Admin
- **Contexto:** El frontend consulta `/admin/internships?status=...` con filtros normalizados.
- **Objetivo:** Validar la correspondencia entre filtros del dashboard y estados funcionales de prácticas.
- **Escenario:** Se filtra una lista de prácticas por `submitted`, `in_review` y `approved`.
- **Variantes cubiertas:**
  - `submitted` incluye `Pendiente` y prácticas sin estado.
  - `in_review` incluye `En revisión` y registros legacy `En revisión DIRAE`.
  - `approved` incluye `Aprobada`.
  - Prácticas anuladas quedan fuera de los filtros.
- **Resultado esperado:** Cada filtro retorna solo las prácticas esperadas.
- **Valor de negocio:** Protege bandejas de trabajo y evita mezclar trámites en estados incorrectos.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_service.py::test_get_internships_filters_by_dashboard_status`

### CU-U-AD-04: Detalle administrativo de práctica existente o inexistente

- **Tipo de prueba:** Unitaria
- **Dominio:** Admin
- **Contexto:** Coordinación abre el detalle de una práctica para revisar antecedentes.
- **Objetivo:** Validar que el service retorna detalle completo si existe y `None` si no existe.
- **Escenario:** Se solicita una práctica existente y luego una inexistente.
- **Variantes cubiertas:**
  - Práctica existente con estudiante, estado y datos de anulación.
  - Práctica inexistente.
- **Resultado esperado:** El detalle existe con datos completos o se retorna `None` sin inventar respuesta vacía.
- **Valor de negocio:** Evita mostrar antecedentes falsos o incompletos.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_service.py::test_get_internship_detail_returns_detail`
  - `tests/modules/admin/test_admin_service.py::test_get_internship_detail_returns_none`

### CU-U-AD-05: Actualización de requisito académico registra trazabilidad

- **Tipo de prueba:** Unitaria
- **Dominio:** Admin
- **Contexto:** Los cambios administrativos deben conservar quién actualizó y cuándo se actualizó el requisito.
- **Objetivo:** Validar que la actualización registra `status_updated_at` sin zona horaria y `status_updated_by`.
- **Escenario:** Un administrador actualiza un requisito académico con transición válida.
- **Variantes cubiertas:**
  - Estado actualizado.
  - Usuario actualizador registrado.
  - Timestamp compatible con columnas `timestamp without time zone`.
- **Resultado esperado:** La actualización conserva trazabilidad funcional.
- **Valor de negocio:** Aporta auditoría sobre cambios administrativos sensibles.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_service.py::test_update_requirement_status_uses_naive_timestamp`

### CU-U-AD-06: Reportes agregados respetan alcance por rol y carrera

- **Tipo de prueba:** Unitaria
- **Dominio:** Admin / Reports
- **Contexto:** Los reportes agregados pueden ser transversales para FICA o acotados por carrera para Dirección.
- **Objetivo:** Validar que el service aplica alcance efectivo y rechaza consultas fuera de la carrera del actor.
- **Escenario:** FICA, Director de carrera y filtros de carrera solicitan dashboard.
- **Variantes cubiertas:**
  - FICA obtiene alcance transversal.
  - Director fuerza su `career_code` propio.
  - Director no puede consultar otra carrera.
- **Resultado esperado:** Los filtros efectivos y `scope` reflejan el rol del actor.
- **Valor de negocio:** Evita exposición agregada fuera del ámbito autorizado.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_report_service.py::test_fica_report_uses_cross_career_scope`
  - `tests/modules/admin/test_admin_report_service.py::test_director_scope_forces_own_career_code`
  - `tests/modules/admin/test_admin_report_service.py::test_director_cannot_request_other_career_code`

### CU-U-AD-07: Exportación CSV de reportes omite campos personales

- **Tipo de prueba:** Unitaria
- **Dominio:** Admin / Reports
- **Contexto:** La exportación de reportes debe ser agregada y no exponer datos personales de estudiantes.
- **Objetivo:** Confirmar que el CSV contiene métricas agregadas y no incluye RUT ni correo.
- **Escenario:** FICA exporta el reporte agregado en CSV.
- **Variantes cubiertas:**
  - Contenido agregado con organizaciones y totales.
  - Ausencia de campos personales evidentes.
- **Resultado esperado:** El CSV es útil para análisis y no expone identificadores personales.
- **Valor de negocio:** Protege privacidad en reportes administrativos.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_report_service.py::test_report_csv_is_aggregate_without_personal_fields`

### CU-U-AD-08: Seguro escolar por solicitud respeta estado y anulación

- **Tipo de prueba:** Unitaria
- **Dominio:** Admin
- **Contexto:** El seguro escolar que condiciona aprobaciones fuera de periodo regular se valida por solicitud concreta.
- **Objetivo:** Confirmar que la validación actualiza la solicitud y rechaza prácticas anuladas.
- **Escenario:** Dirección valida una solicitud vigente y luego intenta validar una solicitud anulada.
- **Variantes cubiertas:**
  - Solicitud queda `validated`, con usuario validador y notas.
  - Solicitud anulada lanza error y no persiste cambios.
- **Resultado esperado:** Solo solicitudes vigentes pueden recibir validación de seguro escolar.
- **Valor de negocio:** Evita modificar trámites anulados y protege una regla institucional crítica.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_service.py::test_update_internship_school_insurance_validates_request`
  - `tests/modules/admin/test_admin_service.py::test_update_internship_school_insurance_rejects_cancelled_practice`

### CU-U-AD-09: Requisito institucional histórico de seguro escolar aplica solo a estudiantes

- **Tipo de prueba:** Unitaria
- **Dominio:** Admin
- **Contexto:** El requisito institucional de seguro escolar del estudiante sigue existiendo como dato de apoyo histórico.
- **Objetivo:** Validar consulta, creación, revocación y rechazo para usuarios no estudiantes.
- **Escenario:** Dirección consulta requisitos, crea requisito faltante, revoca uno existente e intenta actualizar un usuario no estudiante.
- **Variantes cubiertas:**
  - Consulta retorna requisito `school_insurance` completado.
  - Requisito faltante se crea como completado con trazabilidad.
  - Revocación limpia `completed_at`.
  - Usuario sin rol `Estudiante` retorna `None`.
- **Resultado esperado:** El requisito institucional conserva trazabilidad y solo aplica a estudiantes.
- **Valor de negocio:** Evita datos institucionales inválidos y mantiene apoyo diagnóstico para aprobación.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_service.py::test_get_student_registration_requirements_returns_institutional_data`
  - `tests/modules/admin/test_admin_service.py::test_update_school_insurance_creates_missing_requirement`
  - `tests/modules/admin/test_admin_service.py::test_update_school_insurance_clears_completion_when_revoked`
  - `tests/modules/admin/test_admin_service.py::test_update_school_insurance_returns_none_for_non_student`
