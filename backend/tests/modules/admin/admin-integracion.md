# Casos de Prueba - Admin

## Alcance

Estos casos documentan pruebas de integración liviana del módulo `admin` centradas en dependencias de autorización usadas por los endpoints administrativos y de reportes.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos de autorización verificables.

## Integración

### CU-I-AD-01: Roles de lectura administrativa permiten decisión académica

- **Tipo de prueba:** Integración
- **Dominio:** Admin
- **Contexto:** Los endpoints de lectura administrativa son consumidos por roles que toman decisiones sobre prácticas.
- **Objetivo:** Validar que `Encargado de practica` y `Director de carrera` pasan la dependencia de roles, mientras roles no decisores son rechazados.
- **Escenario:** Usuarios con distintos roles ejecutan la dependencia `require_roles` configurada para lectura admin.
- **Variantes cubiertas:**
  - `Encargado de practica` autorizado.
  - `Director de carrera` autorizado.
  - Secretaría, estudiante, supervisor, FICA y Superadmin rechazados para este alcance.
- **Resultado esperado:** Solo roles de decisión académica pasan la validación.
- **Valor de negocio:** Evita exposición de vistas administrativas a roles no autorizados.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_router.py::test_admin_read_roles_are_authorized`
  - `tests/modules/admin/test_admin_router.py::test_admin_read_rejects_non_decision_roles`

### CU-I-AD-02: Seguro escolar queda restringido a Dirección de carrera

- **Tipo de prueba:** Integración
- **Dominio:** Admin
- **Contexto:** La validación de seguro escolar por solicitud corresponde a Dirección de carrera.
- **Objetivo:** Validar que solo `Director de carrera` pasa la dependencia de roles para seguro escolar.
- **Escenario:** Usuarios con roles de dirección, coordinación y estudiante intentan pasar la autorización.
- **Variantes cubiertas:**
  - Director autorizado.
  - Encargado de práctica rechazado.
  - Estudiante rechazado.
- **Resultado esperado:** Solo Dirección puede validar seguro escolar administrativo.
- **Valor de negocio:** Evita que estudiantes o coordinación modifiquen una validación institucional sensible.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_router.py::test_school_insurance_director_role_is_authorized`
  - `tests/modules/admin/test_admin_router.py::test_school_insurance_coordinator_role_is_rejected`
  - `tests/modules/admin/test_admin_router.py::test_school_insurance_student_role_is_rejected`

### CU-I-AD-03: Reportes administrativos usan roles propios de análisis

- **Tipo de prueba:** Integración
- **Dominio:** Admin / Reports
- **Contexto:** Los reportes agregados se consumen por FICA, coordinación y Dirección.
- **Objetivo:** Validar autorización para reportes y rechazo de roles no consumidores.
- **Escenario:** Usuarios con roles autorizados y no autorizados ejecutan la dependencia de reportes.
- **Variantes cubiertas:**
  - `FICA`, `Encargado de practica` y `Director de carrera` autorizados.
  - Secretaría, estudiante, supervisor y Superadmin rechazados.
- **Resultado esperado:** Solo roles definidos para reportes pasan la validación.
- **Valor de negocio:** Controla acceso a métricas agregadas de gestión.
- **Pruebas automatizadas:**
  - `tests/modules/admin/test_admin_report_service.py::test_admin_report_roles_are_authorized`
  - `tests/modules/admin/test_admin_report_service.py::test_admin_report_rejects_non_report_roles`
