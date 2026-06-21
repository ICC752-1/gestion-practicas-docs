# Casos de Prueba - Internships

## Alcance

Estos casos documentan las pruebas de valor del módulo `internships`. El foco está en reglas de aprobación, requisitos institucionales, secuencialidad académica, permisos, trazabilidad, edición administrativa, anulación lógica, contratos de entrada y acceso a información de prácticas.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino un comportamiento de negocio verificable con una o más pruebas automatizadas.

## Unitarias

### CU-U-IN-01: Bloquear aprobación final de práctica estival sin seguro ni excepción

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Una práctica estival es una práctica realizada en `Verano` o `Invierno`. Crear la solicitud no autoriza el inicio ni la aprobación final; la autorización formal requiere seguro escolar o una excepción administrativa.
- Objetivo: Evitar que el sistema apruebe formalmente una práctica estival sin cobertura registrada ni autorización excepcional.
- Escenario: Una práctica estival intenta pasar a `Aprobada`, pero el estudiante no tiene seguro escolar vigente ni excepción administrativa.
- Variantes cubiertas:
  - Aprobación final estival sin seguro escolar ni excepción.
  - Aprobación final estival evaluada desde el flujo integrado de reglas de aprobación.
- Resultado esperado: La aprobación falla con `409 Conflict`; el estado de la práctica no avanza a `Aprobada`.
- Valor de negocio: Protege al estudiante y a la institución frente a una aprobación sin cobertura válida.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_seasonal_without_insurance_raises_409`
  - `tests/modules/internships/test_internship_exception.py::test_approve_seasonal_internship_raises_409_without_insurance_or_exception`

### CU-U-IN-02: Permitir aprobación estival si el seguro fue regularizado antes de aprobar

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Un estudiante puede crear una solicitud de práctica estival cuando aún no tiene seguro escolar. La validación crítica se realiza al momento de aprobar, usando el requisito institucional vigente.
- Objetivo: Confirmar que el sistema revisa el seguro escolar actualizado al momento de la aprobación final.
- Escenario: El estudiante crea una práctica estival sin seguro. Luego un administrador regulariza el seguro escolar y se intenta aprobar la práctica.
- Variantes cubiertas:
  - El valor inicial `has_school_insurance=False` no bloquea si el requisito institucional vigente está completado.
  - La aprobación actualiza la copia de compatibilidad `has_school_insurance` con el valor vigente.
- Resultado esperado: La práctica se aprueba porque el seguro está regularizado al momento de aprobar.
- Valor de negocio: Evita bloquear trámites válidos cuando el estudiante regulariza el requisito antes de la decisión final.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approval_uses_current_insurance_instead_of_creation_snapshot`

### CU-U-IN-03: Permitir avance a revisión de práctica estival sin seguro

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: La creación o revisión inicial de una práctica estival no equivale a autorización final. El seguro escolar se exige cuando la transición termina en `Aprobada`.
- Objetivo: Evitar que el sistema bloquee prematuramente solicitudes estivales que todavía solo están en revisión.
- Escenario: Una práctica estival en `Pendiente` avanza a `En revisión` sin seguro escolar.
- Variantes cubiertas:
  - Encargado de práctica aprueba desde `Pendiente`, generando solo avance a `En revisión`.
- Resultado esperado: La práctica pasa a `En revisión` sin exigir seguro escolar en esa etapa.
- Valor de negocio: Permite iniciar la revisión administrativa sin confundir revisión con aprobación final.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_seasonal_request_can_advance_to_review_without_insurance`

### CU-U-IN-04: Bloquear Práctica I si el estudiante no aprobó la inducción

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: La inducción obligatoria es un prerrequisito inexceptuable para tramitar la aprobación administrativa de `Práctica de Estudio I`.
- Objetivo: Evitar que un estudiante avance en su primera práctica sin completar la inducción.
- Escenario: Se intenta aprobar una `Práctica de Estudio I` sin requisito de inducción completado ni intento aprobado.
- Variantes cubiertas:
  - Práctica I bloqueada por requisito `induction` incompleto.
  - Práctica I bloqueada cuando tampoco existe intento de inducción aprobado.
- Resultado esperado: La aprobación falla con `409 Conflict`.
- Valor de negocio: Asegura el cumplimiento de un prerrequisito formativo obligatorio.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_practice_1_blocked_without_induction`
  - `tests/modules/internships/test_internship_exception.py::test_approve_practice_1_without_induction_raises_409_absolute_block`

### CU-U-IN-05: Permitir Práctica I con inducción aprobada

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Si el estudiante cumple la inducción obligatoria, la Práctica I puede continuar su flujo administrativo normal.
- Objetivo: Confirmar que el requisito de inducción no bloquea prácticas que sí cumplen el prerrequisito.
- Escenario: Se intenta aprobar una `Práctica de Estudio I` con inducción completada.
- Variantes cubiertas:
  - Inducción completada mediante requisito institucional.
  - Inducción aprobada mediante intento previo como respaldo de elegibilidad.
- Resultado esperado: La práctica puede avanzar según la matriz de aprobación correspondiente.
- Valor de negocio: Evita falsos bloqueos a estudiantes que ya cumplieron la inducción.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_practice_1_allowed_with_induction_completed`
  - `tests/modules/internships/test_induction_service.py::TestRegistrationEligibility::test_eligibility_uses_passed_induction_attempt_as_fallback`

### CU-U-IN-06: Bloquear Práctica II sin Práctica I aprobada

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: La `Práctica de Estudio II` depende académicamente de que `Práctica de Estudio I` esté aprobada o de que exista una excepción administrativa de secuencialidad.
- Objetivo: Proteger la secuencia académica del proceso de prácticas.
- Escenario: Se intenta aprobar una `Práctica de Estudio II` sin requisito académico de Práctica I en estado `Aprobada` y sin excepción.
- Variantes cubiertas:
  - No existe requisito académico de Práctica I aprobada.
  - Existe requisito académico de Práctica I, pero no está aprobado.
  - El estado de Práctica I no permite satisfacer secuencialidad.
- Resultado esperado: La aprobación falla con `409 Conflict` y detalle de regla `sequentiality`.
- Valor de negocio: Evita avances académicos fuera de orden.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_practice_2_fails_without_practice_1_requirement`
  - `tests/modules/internships/test_internship_exception.py::test_approve_practice_2_blocked_without_approved_practice_1`
  - `tests/modules/internships/test_internship_exception.py::test_approve_practice_2_none_status_not_crashes`

### CU-U-IN-07: Permitir Práctica II con Práctica I aprobada o excepción

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: La secuencialidad de Práctica II puede cumplirse por requisito académico aprobado o por excepción administrativa justificada.
- Objetivo: Confirmar que el sistema permite avanzar cuando existe una condición válida de secuencialidad.
- Escenario: Se intenta aprobar una `Práctica de Estudio II` con Práctica I aprobada o con excepción `sequentiality` registrada.
- Variantes cubiertas:
  - Práctica II con requisito académico de Práctica I en `Aprobada`.
  - Práctica II con excepción administrativa `sequentiality`.
  - Práctica II no requiere inducción de Práctica I para avanzar.
- Resultado esperado: La práctica avanza según la matriz de aprobación correspondiente.
- Valor de negocio: Permite continuidad del trámite cuando la secuencia académica está satisfecha o formalmente exceptuada.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_practice_2_allowed_with_practice_1_requirement`
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_practice_2_with_sequentiality_exception`
  - `tests/modules/internships/test_internship_exception.py::test_approve_practice_2_allowed_with_approved_practice_1`
  - `tests/modules/internships/test_internship_exception.py::test_approve_practice_2_allowed_with_sequentiality_exception`
  - `tests/modules/internships/test_internship_exception.py::test_approve_practice_2_without_induction_allows_advance`

### CU-U-IN-08: Validar secuencialidad de Tesis

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: La `Tesis` requiere que `Práctica de Estudio II` esté aprobada, salvo que exista una excepción administrativa específica.
- Objetivo: Evitar que una tesis avance sin cumplir el hito académico previo.
- Escenario: Se intenta aprobar una `Tesis` con y sin Práctica II aprobada o excepción.
- Variantes cubiertas:
  - Tesis bloqueada sin Práctica II aprobada.
  - Tesis permitida con Práctica II aprobada.
  - Tesis permitida con excepción `sequentiality_thesis`.
- Resultado esperado: La aprobación falla con `409 Conflict` cuando no se cumple la regla y avanza cuando existe requisito o excepción válida.
- Valor de negocio: Protege la progresión académica hacia tesis.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_thesis_fails_without_practice_2_requirement`
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_thesis_allowed_with_practice_2_requirement`
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_thesis_with_sequentiality_exception`

### CU-U-IN-09: Validar regla de Práctica Controlada y ramo paralelo

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: La `Práctica Controlada` requiere resolver co-requisitos o contar con una excepción administrativa `parallel_course`.
- Objetivo: Evitar que una práctica controlada avance sin excepción mientras los co-requisitos no están modelados como cumplidos.
- Escenario: Se intenta aprobar una `Práctica Controlada` con y sin excepción `parallel_course`.
- Variantes cubiertas:
  - Práctica Controlada bloqueada sin excepción.
  - Práctica Controlada permitida con excepción `parallel_course`.
- Resultado esperado: La aprobación falla con `409 Conflict` sin excepción y avanza con excepción válida.
- Valor de negocio: Evita aprobar una práctica con co-requisitos pendientes sin trazabilidad administrativa.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_controlled_practice_fails_without_parallel_exception`
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_controlled_practice_allowed_with_parallel_exception`

### CU-U-IN-10: Validar aprobación normal versus aprobación directa

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Una práctica en estado `Pendiente` puede pasar a `En revisión` o directamente a `Aprobada` según actor y configuración.
- Objetivo: Confirmar que el sistema aplica correctamente la matriz de transición administrativa.
- Escenario: Una práctica es aprobada desde distintos estados y por distintos actores.
- Variantes cubiertas:
  - Encargado de práctica aprueba desde `Pendiente` y avanza a `En revisión`.
  - Director de carrera aprueba desde `Pendiente` y avanza directo a `Aprobada`.
  - Encargado aprueba con `skip_review=True` y avanza a `Aprobada`.
  - Actor autorizado aprueba desde `En revisión`.
  - Actor autorizado aprueba desde `En revisión DIRAE` solo como cobertura
    legacy; el flujo actual no debe generar ese estado como fase de DIRAE.
- Resultado esperado: Cada transición produce el estado definido por la matriz de negocio.
- Valor de negocio: Preserva la trazabilidad del flujo regular y la facultad de aprobación directa cuando corresponde.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_actions.py::TestApprove::test_encargado_aprueba_desde_pendiente_avanza_a_en_revision`
  - `tests/modules/internships/test_internship_actions.py::TestApprove::test_director_aprueba_desde_pendiente_directo_a_aprobada`
  - `tests/modules/internships/test_internship_actions.py::TestApprove::test_skip_review_se_conserva_por_compatibilidad`
  - `tests/modules/internships/test_internship_actions.py::TestApprove::test_etapa2_en_revision_a_aprobada`
  - `tests/modules/internships/test_internship_actions.py::TestApprove::test_etapa2_desde_en_revision_dirae`

### CU-U-IN-11: Impedir acciones sobre estados terminales

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Los estados terminales son estados finales: `Aprobada`, `Rechazada` y `Reprobada`. Una práctica cerrada no debe modificarse por acciones normales.
- Objetivo: Evitar inconsistencias como aprobar dos veces, rechazar una práctica aprobada, derivar una práctica cerrada o registrar excepciones sobre un trámite cerrado.
- Escenario: Se intenta operar sobre prácticas en estado terminal.
- Variantes cubiertas:
  - Aprobar práctica ya `Aprobada`.
  - Rechazar práctica ya `Rechazada`.
  - Rechazar práctica ya `Aprobada`.
  - Derivar práctica `Aprobada`.
  - Derivar práctica `Rechazada`.
  - Editar una práctica en estado terminal.
  - Registrar excepción sobre una práctica en estado terminal.
- Resultado esperado: El sistema responde `409 Conflict` y no modifica la práctica.
- Valor de negocio: Protege la integridad del historial y evita reabrir trámites cerrados sin flujo formal.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_actions.py::TestApprove::test_ya_aprobada_lanza_409`
  - `tests/modules/internships/test_internship_actions.py::TestReject::test_ya_rechazada_lanza_409`
  - `tests/modules/internships/test_internship_actions.py::TestReject::test_ya_aprobada_lanza_409`
  - `tests/modules/internships/test_internship_actions.py::TestDerive::test_aprobada_lanza_409`
  - `tests/modules/internships/test_internship_actions.py::TestDerive::test_rechazada_lanza_409`
  - `tests/modules/internships/test_admin_edit_cancel_internships.py::TestAdminUpdate::test_admin_update_rejects_terminal_status`

### CU-U-IN-12: Rechazo y derivación deben exigir comentario

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Rechazar o derivar una práctica afecta directamente el proceso del estudiante y debe quedar justificado.
- Objetivo: Asegurar que esas acciones no puedan ejecutarse sin una razón trazable.
- Escenario: Un actor intenta rechazar o derivar una práctica sin entregar comentario válido.
- Variantes cubiertas:
  - Rechazo sin comentario.
  - Rechazo con comentario en blanco.
  - Derivación sin comentario.
  - Derivación con comentario en blanco.
- Resultado esperado: El sistema responde `400 Bad Request` y no cambia el estado de la práctica.
- Valor de negocio: Evita decisiones administrativas sin motivo registrado.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_actions.py::TestReject::test_sin_comentario_lanza_400`
  - `tests/modules/internships/test_internship_actions.py::TestReject::test_comentario_en_blanco_lanza_400`
  - `tests/modules/internships/test_internship_actions.py::TestDerive::test_sin_comentario_lanza_400`
  - `tests/modules/internships/test_internship_actions.py::TestDerive::test_comentario_en_blanco_lanza_400`

### CU-U-IN-13: Validar excepciones administrativas

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Las excepciones administrativas permiten avanzar pese a una regla incumplida, pero solo para reglas admitidas y por roles autorizados.
- Objetivo: Garantizar que las excepciones sean trazables, idempotentes y restringidas.
- Escenario: Un actor intenta registrar excepciones administrativas válidas, inválidas, duplicadas o sin permisos.
- Variantes cubiertas:
  - Registro exitoso de excepción `school_insurance`.
  - Idempotencia al registrar la misma regla para la misma práctica.
  - Rechazo de reglas no exceptuables.
  - Rechazo de excepción por rol no autorizado.
  - Registro de excepción `sequentiality`.
- Resultado esperado: Las excepciones válidas se registran o reutilizan; los intentos inválidos fallan con `400` o `403`.
- Valor de negocio: Evita bypasses informales y conserva trazabilidad de decisiones excepcionales.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_exception.py::test_grant_exception_success_and_idempotency`
  - `tests/modules/internships/test_internship_exception.py::test_grant_exception_rejects_invalid_rules`
  - `tests/modules/internships/test_internship_exception.py::test_grant_exception_requires_privileged_role`
  - `tests/modules/internships/test_internship_exception.py::test_grant_sequentiality_exception_success`
  - `tests/modules/internships/test_internship_exception.py::test_grant_sequentiality_exception_rejects_invalid_sequentiality_rule`

### CU-U-IN-14: Elegibilidad de registro informa bloqueos sin impedir creación

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: La elegibilidad orienta al frontend sobre requisitos pendientes, pero no crea, aprueba ni rechaza prácticas.
- Objetivo: Confirmar que el diagnóstico comunica bloqueos relevantes sin convertirlos en bloqueo de creación.
- Escenario: Se consulta elegibilidad con combinaciones de seguro, inducción, Práctica I aprobada y excepciones.
- Variantes cubiertas:
  - Falta seguro e inducción para práctica estival de tipo I.
  - Todos los requisitos relevantes están cumplidos.
  - Inducción cumplida mediante intento aprobado.
  - Práctica semestral no se bloquea por falta de seguro.
  - Secuencialidad detecta ausencia de Práctica I aprobada.
  - Secuencialidad detecta excepción registrada.
- Resultado esperado: La respuesta indica flags y `blocked` según el contexto consultado.
- Valor de negocio: Permite advertencias tempranas al estudiante sin impedir solicitudes que aún pueden ser revisadas.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestRegistrationEligibility::test_eligibility_returns_blocked_when_no_insurance_and_no_induction`
  - `tests/modules/internships/test_induction_service.py::TestRegistrationEligibility::test_eligibility_returns_not_blocked_when_all_met`
  - `tests/modules/internships/test_induction_service.py::TestRegistrationEligibility::test_eligibility_uses_passed_induction_attempt_as_fallback`
  - `tests/modules/internships/test_induction_service.py::TestRegistrationEligibility::test_semester_does_not_block_when_school_insurance_is_missing`
  - `tests/modules/internships/test_internship_exception.py::test_registration_eligibility_sequentiality_blocked`
  - `tests/modules/internships/test_internship_exception.py::test_registration_eligibility_has_approved_practice_1`
  - `tests/modules/internships/test_internship_exception.py::test_registration_eligibility_has_sequentiality_exception`
  - `tests/modules/internships/test_internship_exception.py::test_registration_eligibility_school_insurance_exception_filtered`

### CU-U-IN-15: Crear Práctica II no exige Práctica I aprobada

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: La secuencialidad se valida al aprobar, no al crear la solicitud de Práctica II.
- Objetivo: Permitir que el estudiante registre la solicitud aunque la aprobación final dependa de requisitos posteriores.
- Escenario: Se crea una solicitud de `Práctica de Estudio II` con y sin Práctica I aprobada.
- Variantes cubiertas:
  - Creación de Práctica II sin Práctica I aprobada.
  - Creación de Práctica II con una Práctica I activa pero no aprobada.
- Resultado esperado: La solicitud se crea en estado inicial y no se bloquea por secuencialidad.
- Valor de negocio: Evita bloquear solicitudes tempranas que aún pueden regularizarse antes de la aprobación.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_exception.py::test_create_practice_2_allowed_without_approved_practice_1`
  - `tests/modules/internships/test_internship_exception.py::test_create_practice_2_allowed_with_active_practice_1`

### CU-U-IN-16: Aprobación sincroniza requisito académico

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Al aprobar una práctica, el sistema actualiza el requisito académico asociado para que futuras reglas de secuencialidad usen una fuente consultable.
- Objetivo: Confirmar que una aprobación final deja reflejado el avance académico del estudiante.
- Escenario: Director aprueba Práctica I o Práctica II y el sistema sincroniza `StudentInternshipRequirement`.
- Variantes cubiertas:
  - Aprobación de Práctica I sincroniza requisito académico de Práctica I.
  - Aprobación de Práctica II sincroniza requisito académico de Práctica II.
- Resultado esperado: El requisito académico correspondiente queda en estado `Aprobada`.
- Valor de negocio: Mantiene consistencia entre el estado administrativo de la práctica y las reglas académicas posteriores.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_practice_1_syncs_academic_requirement`
  - `tests/modules/internships/test_induction_service.py::TestIntegratedRules::test_approve_practice_2_syncs_academic_requirement`

### CU-U-IN-17: Creación calcula seguro escolar desde requisito institucional

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: El estudiante no envía `has_school_insurance` en el payload de creación; el backend calcula ese valor desde el requisito institucional.
- Objetivo: Evitar que el frontend declare manualmente una cobertura que debe provenir del backend.
- Escenario: Se crea una práctica con requisito de seguro completado, inexistente o no completado.
- Variantes cubiertas:
  - Requisito `school_insurance` completado produce `has_school_insurance=True`.
  - Sin requisito produce `has_school_insurance=False`.
  - Requisito existente pero incompleto produce `has_school_insurance=False`.
- Resultado esperado: El valor persistido se calcula desde la fuente institucional.
- Valor de negocio: Mantiene la autoridad del backend sobre requisitos institucionales.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestSchoolInsuranceComputation::test_create_sets_insurance_true_when_student_has_requirement`
  - `tests/modules/internships/test_induction_service.py::TestSchoolInsuranceComputation::test_create_sets_insurance_false_when_student_lacks_requirement`
  - `tests/modules/internships/test_induction_service.py::TestSchoolInsuranceComputation::test_create_sets_insurance_false_when_requirement_not_completed`
  - `tests/modules/internships/test_internship_service.py::test_create_internship_assigns_authenticated_user_id`

### CU-U-IN-18: Dashboard normaliza estados y calcula estadísticas

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: El dashboard de revisión usa estados normalizados como `submitted`, `in_review`, `approved` y `rejected`.
- Objetivo: Confirmar que estados internos y legacy se transforman correctamente para el dashboard.
- Escenario: Se listan o cuentan prácticas con estado nulo, `Pendiente`, `En revisión`, `Aprobada`, `Rechazada` y `Reprobada`.
- Variantes cubiertas:
  - Estado nulo se muestra como `submitted` / `Pendiente`.
  - `Reprobada` se normaliza como `rejected`.
  - Filtro por estado normalizado devuelve solo coincidencias.
  - Estadísticas agregadas cuentan cada grupo normalizado.
- Resultado esperado: El dashboard recibe estados y conteos consistentes.
- Valor de negocio: Evita métricas y filtros incorrectos en la revisión administrativa.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_service.py::test_list_dashboard_internships_maps_null_status_as_submitted`
  - `tests/modules/internships/test_internship_service.py::test_list_dashboard_internships_filters_by_normalized_status`
  - `tests/modules/internships/test_internship_service.py::test_get_dashboard_stats_counts_normalized_statuses`

### CU-U-IN-19: Edición administrativa exige motivo, rol y campos válidos

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: La edición administrativa corrige datos de una práctica sin cambiar su estado principal, por lo que debe ser trazable y acotada.
- Objetivo: Evitar ediciones sin motivo, por roles no autorizados o sobre campos no permitidos.
- Escenario: Se intenta editar una práctica con combinaciones válidas e inválidas.
- Variantes cubiertas:
  - Edición válida actualiza campos permitidos y registra motivo limpio.
  - Campo prohibido es rechazado por schema.
  - Rol incorrecto no puede editar.
  - Estado terminal no permite edición.
  - Motivo en blanco es rechazado.
  - Payload sin campos editables es rechazado.
- Resultado esperado: Solo la edición válida se aplica; los intentos inválidos fallan con `400`, `403` o `409`.
- Valor de negocio: Protege la trazabilidad y evita correcciones administrativas indebidas.
- Pruebas automatizadas:
  - `tests/modules/internships/test_admin_edit_cancel_internships.py::TestAdminUpdate::test_admin_update_valid_updates_allowed_fields`
  - `tests/modules/internships/test_admin_edit_cancel_internships.py::TestAdminUpdate::test_admin_update_rejects_forbidden_field`
  - `tests/modules/internships/test_admin_edit_cancel_internships.py::TestAdminUpdate::test_admin_update_rejects_wrong_role`
  - `tests/modules/internships/test_admin_edit_cancel_internships.py::TestAdminUpdate::test_admin_update_rejects_terminal_status`
  - `tests/modules/internships/test_admin_edit_cancel_internships.py::TestAdminUpdate::test_admin_update_rejects_blank_reason`
  - `tests/modules/internships/test_admin_edit_cancel_internships.py::TestAdminUpdate::test_admin_update_rejects_without_editable_fields`

### CU-U-IN-20: Anulación lógica conserva trazabilidad

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Anular una práctica no debe eliminarla físicamente; debe conservar datos y registrar quién anuló y por qué.
- Objetivo: Confirmar que la anulación se registra como cambio lógico y bloquea anulaciones repetidas.
- Escenario: Un actor autorizado anula una práctica o intenta anular una práctica inexistente/ya anulada.
- Variantes cubiertas:
  - Anulación válida marca la práctica como anulada.
  - Anulación de práctica inexistente retorna `404`.
  - Anulación de práctica ya anulada retorna `409`.
- Resultado esperado: La anulación válida registra `is_cancelled`, `cancelled_by` y `cancellation_reason`; los casos inválidos no llaman al repositorio de persistencia.
- Valor de negocio: Mantiene evidencia histórica sin borrar el trámite.
- Pruebas automatizadas:
  - `tests/modules/internships/test_admin_edit_cancel_internships.py::TestCancelInternship::test_cancel_valid_marks_internship_cancelled`
  - `tests/modules/internships/test_admin_edit_cancel_internships.py::TestCancelInternship::test_cancel_missing_internship_returns_404`
  - `tests/modules/internships/test_admin_edit_cancel_internships.py::TestCancelInternship::test_cancel_rejects_already_cancelled_internship`

### CU-U-IN-21: Contrato de creación de práctica valida payload

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: El schema de creación es el contrato entre frontend y backend para registrar prácticas.
- Objetivo: Evitar solicitudes inválidas o campos controlados por backend en el payload.
- Escenario: Se construye `InternshipCreateRequest` con datos válidos e inválidos.
- Variantes cubiertas:
  - Payload válido es aceptado.
  - Fecha de término anterior a inicio es rechazada.
  - Modalidad inválida es rechazada.
  - Modalidad `Híbrido` es aceptada.
  - Monto negativo es rechazado.
  - Campos opcionales pueden omitirse.
  - Campos requeridos en blanco son rechazados.
  - Email de supervisor inválido es rechazado.
  - `has_school_insurance` enviado por cliente es rechazado.
- Resultado esperado: El schema acepta solo contratos válidos y rechaza datos inválidos con `ValidationError`.
- Valor de negocio: Protege la calidad de datos de solicitudes de práctica.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_schema.py::test_internship_create_request_accepts_valid_payload`
  - `tests/modules/internships/test_internship_schema.py::test_internship_create_request_rejects_end_date_before_start_date`
  - `tests/modules/internships/test_internship_schema.py::test_internship_create_request_rejects_invalid_modality`
  - `tests/modules/internships/test_internship_schema.py::test_internship_create_request_accepts_hybrid_modality_with_accent`
  - `tests/modules/internships/test_internship_schema.py::test_internship_create_request_rejects_negative_amount`
  - `tests/modules/internships/test_internship_schema.py::test_internship_create_request_allows_optional_fields_to_be_omitted`
  - `tests/modules/internships/test_internship_schema.py::test_internship_create_request_rejects_blank_required_text`
  - `tests/modules/internships/test_internship_schema.py::test_internship_create_request_rejects_invalid_supervisor_email`
  - `tests/modules/internships/test_internship_schema.py::test_register_semester_ok`
  - `tests/modules/internships/test_internship_schema.py::test_register_summer_no_insurance`
  - `tests/modules/internships/test_internship_schema.py::test_register_summer_with_insurance`

### CU-U-IN-22: Contrato de excepción valida regla y motivo

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Las solicitudes de excepción administrativa deben declarar una regla admitida y una justificación no vacía.
- Objetivo: Evitar excepciones sin motivo o sobre reglas no soportadas.
- Escenario: Se construye `InternshipExceptionRequest` con reglas válidas, regla inválida y motivo vacío.
- Variantes cubiertas:
  - Reglas `school_insurance`, `sequentiality`, `sequentiality_thesis` y `parallel_course` son aceptadas.
  - Regla no soportada es rechazada.
  - Motivo vacío, espacios o nulo es rechazado.
- Resultado esperado: El schema acepta solo reglas exceptuables con motivo válido.
- Valor de negocio: Evita registrar excepciones administrativas ambiguas o no autorizadas por contrato.
- Pruebas automatizadas:
  - `tests/modules/internships/test_induction_service.py::TestInternshipExceptionRequestSchema::test_accepts_school_insurance`
  - `tests/modules/internships/test_induction_service.py::TestInternshipExceptionRequestSchema::test_accepts_sequentiality`
  - `tests/modules/internships/test_induction_service.py::TestInternshipExceptionRequestSchema::test_accepts_sequentiality_thesis`
  - `tests/modules/internships/test_induction_service.py::TestInternshipExceptionRequestSchema::test_accepts_parallel_course`
  - `tests/modules/internships/test_induction_service.py::TestInternshipExceptionRequestSchema::test_rejects_invalid_rule`
  - `tests/modules/internships/test_induction_service.py::TestInternshipExceptionRequestSchema::test_rejects_blank_reason`
  - `tests/modules/internships/test_internship_schema.py::test_exception_request_rejects_blank_reason`

### CU-U-IN-23: Permisos de lectura permiten propietario o rol privilegiado

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: El detalle, tracking y excepciones de una práctica deben ser visibles para el propietario o para roles administrativos autorizados.
- Objetivo: Evitar acceso cruzado de estudiantes a prácticas ajenas.
- Escenario: Se evalúa acceso de lectura para propietario, rol privilegiado y estudiante no propietario.
- Variantes cubiertas:
  - Propietario puede leer sin rol privilegiado.
  - Rol privilegiado puede leer práctica ajena.
  - Estudiante no propietario sin rol privilegiado no puede leer.
  - Helper de roles detecta presencia o ausencia de roles permitidos.
- Resultado esperado: Solo propietario o rol privilegiado obtiene permiso de lectura.
- Valor de negocio: Protege información sensible de prácticas y estudiantes.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_permissions.py::test_has_any_role_returns_true_when_user_has_allowed_role`
  - `tests/modules/internships/test_internship_permissions.py::test_has_any_role_returns_false_when_user_lacks_allowed_roles`
  - `tests/modules/internships/test_internship_permissions.py::test_can_read_internship_allows_owner_without_privileged_role`
  - `tests/modules/internships/test_internship_permissions.py::test_can_read_internship_allows_privileged_role_for_non_owner`
  - `tests/modules/internships/test_internship_permissions.py::test_can_read_internship_rejects_non_owner_without_privileged_role`

### CU-U-IN-24: Contrato ORM mantiene columnas y enums críticos

- Tipo de prueba: Unitaria
- Dominio: Internships
- Contexto: Los modelos ORM deben mantenerse alineados manualmente con el esquema SQL inicial definido en `init.sql`.
- Objetivo: Detectar cambios accidentales en columnas, enums o nombres físicos usados por el módulo.
- Escenario: Se inspeccionan modelos `Internship` e `InternshipStatusHistory`.
- Variantes cubiertas:
  - `upload_date` usa timestamp naive compatible con base de datos.
  - Columnas snapshot de supervisor existen en `Internship`.
  - Enum de modalidad coincide con contrato de base de datos.
  - Modelo de historial conserva tabla y columnas esperadas.
- Resultado esperado: El contrato ORM conserva nombres y valores críticos.
- Valor de negocio: Reduce riesgo de desalineación entre modelos, SQL y tests al mantener el esquema desde `init.sql`.
- Pruebas automatizadas:
  - `tests/modules/internships/test_internship_model.py::test_upload_date_default_matches_database_timezone_naive_timestamp`
  - `tests/modules/internships/test_internship_model.py::test_internship_model_includes_supervisor_snapshot_columns`
  - `tests/modules/internships/test_internship_model.py::test_internship_modality_enum_matches_database_contract`
  - `tests/modules/internships/test_internship_model.py::test_internship_status_history_model_matches_database_contract`
