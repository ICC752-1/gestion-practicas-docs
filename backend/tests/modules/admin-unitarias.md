# Casos de Prueba - Admin

## Alcance

Estos casos documentan las pruebas de valor del módulo `admin`. El foco está en consultas administrativas, filtros del dashboard, detalle de prácticas, gestión de requisitos académicos, registro institucional de seguro escolar, permisos y traducción de errores del controller.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos de negocio verificables con una o más pruebas automatizadas.

## Unitarias

### CU-U-AD-01: Resumen administrativo conserva totales y estados

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: El dashboard administrativo muestra totales globales y desglose de prácticas por estado.
- Objetivo: Confirmar que el service conserva los conteos entregados por el repositorio y traduce estados faltantes a una etiqueta legible.
- Escenario: El repositorio entrega conteo de estudiantes, conteo de prácticas y agrupación por estado.
- Variantes cubiertas:
  - Resumen con estados explícitos.
  - Resumen con estado nulo traducido a `Sin estado`.
- Resultado esperado: El resumen mantiene totales correctos y no expone estado nulo al contrato HTTP.
- Valor de negocio: Protege indicadores usados por coordinación para monitorear el proceso de prácticas.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_get_summary_returns_summary`
  - `tests/modules/admin/test_admin_service.py::test_get_summary_maps_missing_status`

### CU-U-AD-02: Listado administrativo de estudiantes conserva datos relevantes

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: Coordinación necesita consultar estudiantes registrados y su estado de cuenta desde una vista administrativa.
- Objetivo: Validar que el service mapea identificador, correo, nombre, RUT y estado activo sin perder datos.
- Escenario: El repositorio devuelve estudiantes con información básica.
- Variantes cubiertas:
  - Estudiante activo con datos personales completos.
- Resultado esperado: El listado administrativo conserva los campos relevantes del estudiante.
- Valor de negocio: Evita mostrar información incompleta o incorrecta en vistas administrativas.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_get_students_maps_students`

### CU-U-AD-03: Listado administrativo de prácticas mapea estudiante y estado

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: El dashboard de coordinación consume prácticas con datos del estudiante y estado actual.
- Objetivo: Confirmar que el service construye items administrativos con relaciones anidadas cuando existen y tolera estado ausente.
- Escenario: El repositorio devuelve prácticas con estudiante y estado, y también prácticas sin estado.
- Variantes cubiertas:
  - Práctica con estudiante y estado relacionado.
  - Práctica sin estado relacionado.
- Resultado esperado: El listado conserva estudiante y estado cuando existen; si no hay estado, retorna `status=None` sin romper el contrato.
- Valor de negocio: Evita fallos del dashboard ante datos incompletos o prácticas heredadas sin estado.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_get_internships_maps_related_data`
  - `tests/modules/admin/test_admin_service.py::test_get_internships_allows_none_status`

### CU-U-AD-04: Filtros normalizados del dashboard agrupan estados funcionales

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: El frontend consulta `/admin/internships?status=...` con filtros normalizados, no con nombres internos de estados.
- Objetivo: Validar la correspondencia entre filtros del dashboard y estados funcionales de prácticas.
- Escenario: Se filtra una lista de prácticas por `submitted`, `in_review` y `approved`.
- Variantes cubiertas:
  - `submitted` incluye `Pendiente` y prácticas sin estado.
  - `in_review` incluye `En revisión`; `En revisión DIRAE` solo se acepta como
    compatibilidad con registros legacy, no como estado funcional nuevo.
  - `approved` incluye `Aprobada`.
- Resultado esperado: Cada filtro retorna solo las prácticas esperadas.
- Valor de negocio: Protege las bandejas de trabajo del coordinador y evita mezclar trámites en estados incorrectos.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_get_internships_filters_by_dashboard_status`

### CU-U-AD-05: Detalle administrativo de práctica existente o inexistente

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: Coordinación consulta el detalle de una práctica para tomar decisiones o revisar antecedentes.
- Objetivo: Validar que el service retorna detalle completo si existe y `None` si no existe, dejando al controller traducir el caso a HTTP.
- Escenario: Se solicita una práctica existente y luego una inexistente.
- Variantes cubiertas:
  - Práctica existente con estudiante y estado.
  - Práctica inexistente.
- Resultado esperado: El detalle existe con datos completos o se retorna `None` sin inventar una respuesta vacía.
- Valor de negocio: Evita pantallas administrativas con datos falsos o incompletos.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_get_internship_detail_returns_detail`
  - `tests/modules/admin/test_admin_service.py::test_get_internship_detail_returns_none`

### CU-U-AD-06: Listado de requisitos académicos del estudiante

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: Coordinación revisa requisitos académicos asociados a prácticas de un estudiante.
- Objetivo: Confirmar que el service retorna requisitos académicos con tipo, estado y trazabilidad básica.
- Escenario: El repositorio devuelve requisitos de práctica para un estudiante.
- Variantes cubiertas:
  - Requisito académico existente con estado `Habilitada`.
- Resultado esperado: El requisito se mapea al contrato administrativo esperado.
- Valor de negocio: Permite revisar y gestionar el avance académico-administrativo del estudiante.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_get_student_internship_requirements_maps_requirements`

### CU-U-AD-07: Transiciones válidas de requisitos académicos

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: Los requisitos académicos de práctica tienen un ciclo de estados controlado por administración.
- Objetivo: Validar que el service permite solo avances funcionalmente correctos.
- Escenario: Se actualiza un requisito académico usando transiciones permitidas.
- Variantes cubiertas:
  - `Pendiente` a `Habilitada`.
  - `Habilitada` a `En revisión`.
  - `En revisión` a `Aprobada`.
  - `En revisión` a `Rechazada`.
  - `Rechazada` a `Habilitada`.
- Resultado esperado: La transición se acepta, se actualiza el estado y se registra trazabilidad de actualización.
- Valor de negocio: Protege el flujo académico y evita saltos de estado no autorizados.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_update_student_internship_requirement_accepts_valid_transition`

### CU-U-AD-08: Transiciones inválidas de requisitos académicos

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: Saltar estados puede habilitar indebidamente trámites académicos o registrar aprobaciones sin revisión.
- Objetivo: Confirmar que transiciones no permitidas son rechazadas.
- Escenario: Se intenta actualizar un requisito académico con saltos inválidos.
- Variantes cubiertas:
  - `Pendiente` a `Aprobada`.
  - `Aprobada` a `Rechazada`.
  - `Habilitada` a `Aprobada`.
- Resultado esperado: El service lanza `ValueError` y no acepta la transición.
- Valor de negocio: Evita alteraciones incorrectas del avance académico del estudiante.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_update_student_internship_requirement_rejects_invalid_transition`

### CU-U-AD-09: Actualización de requisito académico registra trazabilidad

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: Los cambios administrativos deben conservar quién actualizó y cuándo se actualizó el requisito.
- Objetivo: Validar que la actualización registra `status_updated_at` y `status_updated_by`, incluyendo el caso de mismo estado.
- Escenario: Un administrador actualiza un requisito académico con transición válida o mismo estado.
- Variantes cubiertas:
  - Transición válida registra usuario y fecha.
  - Actualización con mismo estado se acepta y registra usuario.
  - Requisito inexistente devuelve `None`.
- Resultado esperado: Las actualizaciones válidas conservan trazabilidad; el requisito inexistente no se modifica.
- Valor de negocio: Aporta auditoría funcional sobre cambios administrativos sensibles.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_update_student_internship_requirement_accepts_valid_transition`
  - `tests/modules/admin/test_admin_service.py::test_update_student_internship_requirement_allows_same_status`
  - `tests/modules/admin/test_admin_service.py::test_update_student_internship_requirement_returns_none_when_missing`

### CU-U-AD-10: Actualización de requisito académico emite notificación sin bloquear flujo

- Tipo de prueba: Unitaria
- Dominio: Admin / Notifications
- Contexto: Los cambios de requisitos deben notificar al estudiante, pero un fallo de notificaciones no debe impedir la actualización principal.
- Objetivo: Validar que el service despacha `requirement_status_changed` y tolera errores del servicio de notificaciones.
- Escenario: Se actualiza un requisito académico con servicio de notificaciones exitoso y luego con servicio que falla.
- Variantes cubiertas:
  - Notificación incluye destinatario, requisito, estado nuevo y estado anterior.
  - Fallo de notificación no revierte ni bloquea la actualización.
- Resultado esperado: La actualización se completa y la notificación se intenta cuando hay servicio configurado.
- Valor de negocio: Mantiene comunicación al estudiante sin convertir mensajería en punto único de falla.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_update_student_internship_requirement_dispatches_notification`
  - `tests/modules/admin/test_admin_service.py::test_update_student_internship_requirement_ignores_notification_failure`

### CU-U-AD-11: Listado de requisitos institucionales del estudiante

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: Los prerrequisitos institucionales, como seguro escolar, condicionan aprobaciones finales en otros módulos.
- Objetivo: Validar que solo se retornan requisitos institucionales para usuarios que son estudiantes.
- Escenario: Se consulta la lista de requisitos institucionales de un estudiante.
- Variantes cubiertas:
  - Estudiante existente con requisito `school_insurance` completado.
- Resultado esperado: El service retorna requisitos institucionales del estudiante con estado de cumplimiento.
- Valor de negocio: Permite a roles administrativos verificar condiciones institucionales antes de aprobar trámites.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_get_student_registration_requirements_returns_institutional_data`

### CU-U-AD-12: Seguro escolar se crea o actualiza correctamente

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: El seguro escolar es un prerrequisito institucional usado para aprobar prácticas estivales.
- Objetivo: Confirmar que el service crea el requisito si no existe y limpia la fecha de completitud cuando se revoca.
- Escenario: Un administrador marca seguro escolar como completado y luego se prueba revocación de un requisito existente.
- Variantes cubiertas:
  - Requisito faltante se crea como `school_insurance` completado.
  - Al revocar, `is_completed=false` y `completed_at=None`.
  - Se registra `updated_by`.
- Resultado esperado: El requisito refleja correctamente la cobertura vigente.
- Valor de negocio: Protege una regla institucional crítica para prácticas de temporada.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_update_school_insurance_creates_missing_requirement`
  - `tests/modules/admin/test_admin_service.py::test_update_school_insurance_clears_completion_when_revoked`

### CU-U-AD-13: Seguro escolar rechaza usuarios que no son estudiantes

- Tipo de prueba: Unitaria
- Dominio: Admin
- Contexto: El seguro escolar institucional solo aplica a estudiantes.
- Objetivo: Evitar crear o modificar requisitos institucionales para usuarios con otros roles.
- Escenario: Se intenta actualizar seguro escolar para un usuario con rol administrativo.
- Variantes cubiertas:
  - Usuario existente sin rol `Estudiante`.
- Resultado esperado: El service retorna `None` y no crea requisito institucional.
- Valor de negocio: Evita datos institucionales inválidos asociados a usuarios no estudiantes.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_service.py::test_update_school_insurance_returns_none_for_non_student`
