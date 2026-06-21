# Casos de Prueba - Admin

## Alcance

Estos casos documentan las pruebas de valor del módulo `admin`. El foco está en consultas administrativas, filtros del dashboard, detalle de prácticas, gestión de requisitos académicos, registro institucional de seguro escolar, permisos y traducción de errores del controller.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos de negocio verificables con una o más pruebas automatizadas.

## Integración

### CU-I-AD-01: Roles autorizados para seguro escolar

- Tipo de prueba: Integración
- Dominio: Admin
- Contexto: La gestión de seguro escolar queda bajo responsabilidad de `Director de carrera`.
- Objetivo: Validar que la dependencia de roles permite a `Director de carrera` y rechaza a `Encargado de practica` y estudiantes.
- Escenario: Usuarios con distintos roles intentan pasar la dependencia de autorización.
- Variantes cubiertas:
  - `Director de carrera` autorizado.
  - `Encargado de practica` rechazado con `403`.
  - `Estudiante` rechazado con `403`.
- Resultado esperado: Solo Dirección de carrera pasa la validación.
- Valor de negocio: Evita que estudiantes o coordinación modifiquen una validación que corresponde a Dirección.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_router.py::test_school_insurance_director_role_is_authorized`
  - `tests/modules/admin/test_admin_router.py::test_school_insurance_coordinator_role_is_rejected`
  - `tests/modules/admin/test_admin_router.py::test_school_insurance_student_role_is_rejected`

### CU-I-AD-02: Detalle de práctica inexistente se traduce a 404

- Tipo de prueba: Integración
- Dominio: Admin
- Contexto: El service devuelve `None` cuando una práctica no existe; el controller debe traducirlo a HTTP.
- Objetivo: Validar que el endpoint administrativo no devuelve una respuesta vacía para recursos inexistentes.
- Escenario: Se solicita detalle de una práctica inexistente.
- Variantes cubiertas:
  - Service retorna `None`.
- Resultado esperado: El controller responde `404 Not Found`.
- Valor de negocio: Da una señal clara al frontend y evita mostrar detalles inexistentes.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_router.py::test_get_internship_detail_returns_404_when_missing`

### CU-I-AD-03: Transición inválida se traduce a 400

- Tipo de prueba: Integración
- Dominio: Admin
- Contexto: El service expresa transiciones inválidas como `ValueError`; el contrato HTTP debe exponerlas como error de solicitud.
- Objetivo: Validar la traducción controller-service para cambios inválidos de requisito académico.
- Escenario: Se intenta actualizar un requisito con una transición no permitida.
- Variantes cubiertas:
  - Service lanza `ValueError`.
- Resultado esperado: El controller responde `400 Bad Request`.
- Valor de negocio: Permite al frontend distinguir error de reglas de negocio frente a recurso inexistente.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_router.py::test_update_student_requirement_returns_400_for_invalid_transition`

### CU-I-AD-04: Requisito académico inexistente se traduce a 404

- Tipo de prueba: Integración
- Dominio: Admin
- Contexto: Actualizar un requisito inexistente debe informarse como recurso no encontrado.
- Objetivo: Validar que el controller traduce `None` del service a `404`.
- Escenario: Se intenta actualizar un requisito académico que no pertenece al estudiante o no existe.
- Variantes cubiertas:
  - Service retorna `None`.
- Resultado esperado: El controller responde `404 Not Found`.
- Valor de negocio: Evita que el frontend interprete como éxito una actualización que no ocurrió.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_router.py::test_update_student_requirement_returns_404_when_missing`

### CU-I-AD-05: Seguro escolar de usuario no estudiante se traduce a 404

- Tipo de prueba: Integración
- Dominio: Admin
- Contexto: El service retorna `None` si el usuario no existe o no es estudiante.
- Objetivo: Validar que el endpoint de seguro escolar mantiene un contrato HTTP consistente para usuarios no aplicables.
- Escenario: Un rol autorizado intenta registrar seguro escolar para un usuario que no es estudiante.
- Variantes cubiertas:
  - Service retorna `None`.
- Resultado esperado: El controller responde `404 Not Found`.
- Valor de negocio: Evita crear prerrequisitos institucionales para usuarios incorrectos.
- Pruebas automatizadas:
  - `tests/modules/admin/test_admin_router.py::test_school_insurance_returns_404_for_non_student`
