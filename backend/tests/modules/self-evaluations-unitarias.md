# Casos de Prueba - Self Evaluations

## Alcance

Estos casos documentan las pruebas unitarias de valor del módulo `self_evaluations`. El foco está en habilitación temporal, borradores, envío, bloqueo post-envío y reapertura administrativa auditada.

## Unitarias

### CU-U-SE-01: Formulario se habilita según últimos días hábiles y estado de práctica

- Tipo de prueba: Unitaria
- Dominio: Self Evaluations
- Contexto: La autoevaluación no debe completarse al inicio de la práctica.
- Objetivo: Validar la ventana de habilitación funcional.
- Escenario: Se consulta o evalúa disponibilidad antes y dentro de los últimos 5 días hábiles.
- Variantes cubiertas:
  - Práctica fresca no habilita formulario.
  - Dentro de últimos 5 días hábiles se habilita.
  - Antes de la ventana se rechaza con razón clara.
- Resultado esperado: El formulario solo se habilita cuando corresponde.
- Valor de negocio: Evita autoevaluaciones prematuras.
- Pruebas automatizadas:
  - `tests/modules/self_evaluations/test_self_evaluation_service.py::test_form_is_not_enabled_for_fresh_internship`
  - `tests/modules/self_evaluations/test_self_evaluation_service.py::test_form_is_enabled_during_last_five_business_days`
  - `tests/modules/self_evaluations/test_self_evaluation_service.py::test_form_is_not_enabled_before_last_five_business_days`

### CU-U-SE-02: Estudiante guarda borrador y envío queda bloqueado

- Tipo de prueba: Unitaria
- Dominio: Self Evaluations
- Contexto: El estudiante puede guardar respuestas parciales, pero una autoevaluación enviada no debe modificarse sin reapertura.
- Objetivo: Validar borrador, propiedad y bloqueo post-envío.
- Escenario: Se guarda borrador, un estudiante ajeno intenta enviar y luego se intenta editar una evaluación enviada.
- Variantes cubiertas:
  - Borrador parcial persiste respuestas.
  - Estudiante no propietario recibe `403`.
  - Evaluación enviada no se edita sin reapertura.
- Resultado esperado: Solo el propietario completa la evaluación y el envío bloquea cambios posteriores.
- Valor de negocio: Protege integridad de evidencias académicas.
- Pruebas automatizadas:
  - `tests/modules/self_evaluations/test_self_evaluation_service.py::test_save_draft_persists_partial_responses`
  - `tests/modules/self_evaluations/test_self_evaluation_service.py::test_submit_requires_owner_and_locks_evaluation`
  - `tests/modules/self_evaluations/test_self_evaluation_service.py::test_submitted_evaluation_cannot_be_edited_without_reopen`

### CU-U-SE-03: Reapertura administrativa conserva trazabilidad

- Tipo de prueba: Unitaria
- Dominio: Self Evaluations
- Contexto: Una evaluación enviada puede necesitar corrección bajo decisión administrativa.
- Objetivo: Validar que la reapertura exige motivo y registra actor.
- Escenario: Dirección reabre una autoevaluación enviada con razón de corrección.
- Variantes cubiertas:
  - Estado cambia a `reopened`.
  - Se registra `reopened_by`.
  - Se conserva motivo de reapertura.
- Resultado esperado: La reapertura queda auditada.
- Valor de negocio: Permite correcciones trazables sin debilitar el bloqueo post-envío.
- Pruebas automatizadas:
  - `tests/modules/self_evaluations/test_self_evaluation_service.py::test_admin_reopen_requires_reason_and_records_actor`
