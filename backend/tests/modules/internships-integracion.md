# Casos de Prueba - Internships

## Alcance

Estos casos documentan las pruebas de valor del módulo `internships`. El foco está en reglas de aprobación, requisitos institucionales, secuencialidad académica, permisos, trazabilidad, edición administrativa, anulación lógica, contratos de entrada y acceso a información de prácticas.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino un comportamiento de negocio verificable con una o más pruebas automatizadas.

## Integración

### CU-I-IN-01: Tracking permite propietario y roles privilegiados

- Tipo de prueba: Integración
- Dominio: Internships
- Contexto: El tracking de una práctica expone historial administrativo y debe restringirse al estudiante propietario o a roles de revisión.
- Objetivo: Validar permisos de lectura al consultar tracking desde el controlador.
- Escenario: Se consulta el tracking de una práctica como propietario, rol privilegiado, estudiante no propietario y práctica inexistente.
- Variantes cubiertas:
  - Propietario consulta tracking.
  - Rol privilegiado consulta tracking ajeno.
  - Estudiante no propietario intenta consultar tracking ajeno.
  - Se consulta tracking de práctica inexistente.
- Resultado esperado: Propietario y rol privilegiado acceden; acceso cruzado devuelve `403`; práctica inexistente devuelve `404`.
- Valor de negocio: Protege trazabilidad administrativa y datos de estudiantes.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_router.py::test_get_internship_tracking_allows_owner`
  - `tests/modules/internships/test_internship_router.py::test_get_internship_tracking_allows_privileged_role`
  - `tests/modules/internships/test_internship_router.py::test_get_internship_tracking_rejects_forbidden_user`
  - `tests/modules/internships/test_internship_router.py::test_get_internship_tracking_returns_not_found`

### CU-I-IN-02: Dashboard rechaza rol estudiante

- Tipo de prueba: Integración
- Dominio: Internships
- Contexto: El dashboard de prácticas es una vista administrativa para revisión, no una vista de estudiante.
- Objetivo: Confirmar que un estudiante no puede acceder a los permisos de lectura del dashboard.
- Escenario: Un usuario con rol `Estudiante` intenta pasar la dependencia de roles del dashboard.
- Variantes cubiertas:
  - Estudiante intenta acceder a roles de lectura de dashboard.
- Resultado esperado: La validación de roles devuelve `403 Forbidden`.
- Valor de negocio: Evita exposición de información administrativa a estudiantes.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_router.py::test_dashboard_internships_rejects_student_role`
