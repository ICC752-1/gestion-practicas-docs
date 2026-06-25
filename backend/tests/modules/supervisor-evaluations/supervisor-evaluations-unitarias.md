# Casos de Prueba - Supervisor Evaluations

## Alcance

Estos casos documentan las pruebas unitarias de valor del módulo `supervisor_evaluations`. El foco está en invitaciones públicas, prerequisitos, token de un solo uso, formulario público mínimo, permisos y asignaciones por correo autenticado.

## Unitarias

### CU-U-SV-01: Invitación de supervisor exige práctica aprobada y autoevaluación enviada

- **Tipo de prueba:** Unitaria
- **Dominio:** Supervisor Evaluations
- **Contexto:** La evaluación del supervisor externo ocurre después de la autoevaluación del estudiante y solo para prácticas aprobadas.
- **Objetivo:** Validar prerequisitos para generar invitación.
- **Escenario:** Administración genera invitación con práctica válida, práctica no aprobada y autoevaluación faltante.
- **Variantes cubiertas:**
  - Modo simulado retorna token demo y URL.
  - Token persistido queda hasheado.
  - Práctica no aprobada se rechaza.
  - Autoevaluación no enviada se rechaza.
- **Resultado esperado:** Solo prácticas aprobadas con autoevaluación enviada generan invitación.
- **Valor de negocio:** Protege la secuencia estudiante-supervisor y evita enlaces prematuros.
- **Pruebas automatizadas:**
  - `tests/modules/supervisor_evaluations/test_supervisor_evaluation_service.py::test_generate_invitation_returns_demo_link_in_simulated_mode`
  - `tests/modules/supervisor_evaluations/test_supervisor_evaluation_service.py::test_generate_invitation_requires_approved_internship`
  - `tests/modules/supervisor_evaluations/test_supervisor_evaluation_service.py::test_generate_invitation_requires_submitted_self_evaluation`

### CU-U-SV-02: Formulario público expone datos mínimos y valida vigencia

- **Tipo de prueba:** Unitaria
- **Dominio:** Supervisor Evaluations
- **Contexto:** El supervisor externo accede con token público sin sesión institucional.
- **Objetivo:** Validar que el formulario expone lo mínimo y rechaza invitaciones inválidas por estado o expiración.
- **Escenario:** Se consulta formulario con token válido, luego la práctica deja de estar aprobada y otra invitación expira.
- **Variantes cubiertas:**
  - Formulario muestra organización, estudiante, supervisor y criterios.
  - Si la práctica deja de estar aprobada se rechaza.
  - Invitación expirada devuelve error funcional.
- **Resultado esperado:** El formulario público no expone más de lo necesario y valida vigencia dinámicamente.
- **Valor de negocio:** Protege privacidad y consistencia del acceso externo.
- **Pruebas automatizadas:**
  - `tests/modules/supervisor_evaluations/test_supervisor_evaluation_service.py::test_public_form_exposes_minimum_information_for_valid_token`
  - `tests/modules/supervisor_evaluations/test_supervisor_evaluation_service.py::test_public_form_rejects_invitation_when_internship_stops_being_approved`
  - `tests/modules/supervisor_evaluations/test_supervisor_evaluation_service.py::test_expired_invitation_is_rejected`

### CU-U-SV-03: Envío público consume token e impide reutilización

- **Tipo de prueba:** Unitaria
- **Dominio:** Supervisor Evaluations
- **Contexto:** La evaluación del supervisor externo debe enviarse una sola vez.
- **Objetivo:** Validar persistencia de evaluación y consumo del token.
- **Escenario:** Supervisor envía evaluación y luego intenta reutilizar el mismo token.
- **Variantes cubiertas:**
  - Evaluación queda creada.
  - Invitación queda marcada como usada.
  - Segundo envío recibe conflicto.
- **Resultado esperado:** El token público es de un solo uso.
- **Valor de negocio:** Evita duplicidad o manipulación posterior de evaluaciones externas.
- **Pruebas automatizadas:**
  - `tests/modules/supervisor_evaluations/test_supervisor_evaluation_service.py::test_submit_evaluation_marks_token_used_and_prevents_reuse`

### CU-U-SV-04: Lectura de evaluaciones y asignaciones respeta permisos

- **Tipo de prueba:** Unitaria
- **Dominio:** Supervisor Evaluations
- **Contexto:** La evaluación del supervisor contiene información del desempeño del estudiante.
- **Objetivo:** Validar permisos de lectura y asignaciones por correo autenticado.
- **Escenario:** Roles no autorizados intentan leer evaluación y supervisor consulta prácticas asociadas a su correo.
- **Variantes cubiertas:**
  - Secretaría y FICA no pueden leer evaluación de supervisor.
  - Supervisor ve asignaciones cuyo email coincide con el autenticado.
- **Resultado esperado:** La lectura queda limitada a actores autorizados.
- **Valor de negocio:** Protege datos de evaluación y evita exposición entre supervisores.
- **Pruebas automatizadas:**
  - `tests/modules/supervisor_evaluations/test_supervisor_evaluation_service.py::test_non_admin_role_cannot_read_supervisor_evaluation`
  - `tests/modules/supervisor_evaluations/test_supervisor_evaluation_service.py::test_supervisor_assignments_match_authenticated_email`
