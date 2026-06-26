# Casos de Prueba - Scheduling

## Alcance

Estos casos documentan las pruebas unitarias de valor del módulo `scheduling`. El foco está en disponibilidad, reservas, cancelaciones, reprogramación, solapamientos, resultados de citas y efectos sobre el estado de la práctica.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino reglas funcionales del flujo de agenda.

## Unitarias

### CU-U-SC-01: Administración publica y mantiene disponibilidad futura

- **Tipo de prueba:** Unitaria
- **Dominio:** Scheduling
- **Contexto:** Roles administrativos publican bloques de disponibilidad para entrevistas y presentaciones.
- **Objetivo:** Validar creación, edición, eliminación y permisos de disponibilidad.
- **Escenario:** Un encargado crea bloques, edita uno propio, elimina uno futuro y un estudiante intenta crear disponibilidad.
- **Variantes cubiertas:**
  - Rango horario genera bloques esperados.
  - Estudiante no puede crear disponibilidad.
  - Edición propia futura actualiza datos y duración.
  - Solapamiento de dueño se rechaza.
  - Disponibilidad propia futura se elimina.
- **Resultado esperado:** Solo roles administrativos administran disponibilidad válida y sin solapamientos.
- **Valor de negocio:** Protege la agenda disponible para estudiantes.
- **Pruebas automatizadas:**
  - `tests/modules/scheduling/test_scheduling_service.py::test_create_availability_generates_expected_blocks`
  - `tests/modules/scheduling/test_scheduling_service.py::test_create_availability_rejects_non_admin_role`
  - `tests/modules/scheduling/test_scheduling_service.py::test_admin_can_update_own_future_availability`
  - `tests/modules/scheduling/test_scheduling_service.py::test_update_availability_rejects_owner_overlap`
  - `tests/modules/scheduling/test_scheduling_service.py::test_admin_can_delete_own_future_availability`

### CU-U-SC-02: Reserva de cita valida práctica y duplicados

- **Tipo de prueba:** Unitaria
- **Dominio:** Scheduling
- **Contexto:** Un estudiante reserva horarios para una práctica propia.
- **Objetivo:** Validar asignación de cita y rechazo de condiciones inválidas.
- **Escenario:** El estudiante reserva un bloque disponible y luego se prueban duplicados o práctica anulada.
- **Variantes cubiertas:**
  - Reserva asigna estudiante, práctica y timestamp.
  - Práctica con cita activa para el mismo tipo se rechaza.
  - Práctica anulada se rechaza.
- **Resultado esperado:** Solo prácticas válidas y sin cita activa pueden reservar.
- **Valor de negocio:** Evita duplicidad y citas asociadas a trámites anulados.
- **Pruebas automatizadas:**
  - `tests/modules/scheduling/test_scheduling_service.py::test_reserve_slot_assigns_student_and_internship`
  - `tests/modules/scheduling/test_scheduling_service.py::test_reserve_slot_rejects_duplicate_appointment_for_internship`
  - `tests/modules/scheduling/test_scheduling_service.py::test_reserve_slot_rejects_cancelled_internship`

### CU-U-SC-03: Cancelación y reprogramación respetan actor y tipo de cita

- **Tipo de prueba:** Unitaria
- **Dominio:** Scheduling
- **Contexto:** Estudiantes y administración pueden cancelar citas bajo reglas distintas; estudiantes pueden reprogramar manteniendo tipo de cita.
- **Objetivo:** Validar motivos, propiedad y consistencia de propósito al mover citas.
- **Escenario:** Administración cancela sin motivo, estudiante cancela, estudiante reprograma y se intenta mover a otro propósito.
- **Variantes cubiertas:**
  - Administración requiere motivo.
  - Estudiante cancela cita propia sin motivo.
  - Reprogramación cancela cita anterior y agenda nuevo bloque.
  - Nuevo bloque con propósito distinto se rechaza.
- **Resultado esperado:** Cancelación y reprogramación conservan trazabilidad y coherencia funcional.
- **Valor de negocio:** Evita cambios de agenda ambiguos o inconsistentes.
- **Pruebas automatizadas:**
  - `tests/modules/scheduling/test_scheduling_service.py::test_admin_cancel_requires_reason`
  - `tests/modules/scheduling/test_scheduling_service.py::test_student_can_cancel_own_appointment_without_reason`
  - `tests/modules/scheduling/test_scheduling_service.py::test_student_can_reschedule_own_appointment`
  - `tests/modules/scheduling/test_scheduling_service.py::test_reschedule_rejects_slot_with_different_purpose`

### CU-U-SC-04: Resultado de cita actualiza avance y cierre de práctica

- **Tipo de prueba:** Unitaria
- **Dominio:** Scheduling / Internships
- **Contexto:** La entrevista inicial y presentación final afectan el avance de la práctica.
- **Objetivo:** Validar efectos de resultados sobre `completion_status`, `final_result` y bloqueo de nuevos registros.
- **Escenario:** Se registra resultado de entrevista inicial y presentación final.
- **Variantes cubiertas:**
  - Entrevista inicial completada marca práctica `in_progress`.
  - Presentación final con prerequisitos faltantes se rechaza.
  - Presentación final reprobada finaliza práctica y libera bloqueo de nuevo registro.
- **Resultado esperado:** Los resultados de agenda sincronizan correctamente el estado de la práctica.
- **Valor de negocio:** Protege el cierre académico y la posibilidad de repetir práctica cuando corresponde.
- **Pruebas automatizadas:**
  - `tests/modules/scheduling/test_scheduling_service.py::test_register_initial_interview_outcome_marks_in_progress`
  - `tests/modules/scheduling/test_scheduling_service.py::test_register_final_presentation_outcome_requires_prerequisites`
  - `tests/modules/scheduling/test_scheduling_service.py::test_register_final_presentation_outcome_finalizes_failed_internship`
  - `tests/modules/scheduling/test_scheduling_service.py::test_register_final_presentation_outcome_sends_notification_on_approval`

### CU-U-SC-05: Solicitudes de agenda validan configuración, propósito y duplicidad

- **Tipo de prueba:** Unitaria
- **Dominio:** Scheduling
- **Contexto:** Los estudiantes pueden solicitar agenda para consulta general o presentación final según configuración y prerequisitos.
- **Objetivo:** Validar creación de solicitudes y bloqueos funcionales antes de que administración asigne horario.
- **Escenario:** Se crean solicitudes con consultas generales habilitadas/deshabilitadas, presentación final con prerequisitos y duplicidad.
- **Variantes cubiertas:**
  - Consulta general deshabilitada se rechaza.
  - Consulta general habilitada se crea correctamente.
  - Presentación final exige práctica asociada.
  - Presentación final exige evaluaciones requeridas.
  - Presentación final válida crea solicitud.
  - Solicitud duplicada se rechaza.
  - Coordinador objetivo debe ser válido.
- **Resultado esperado:** Solo solicitudes coherentes con configuración y prerequisitos quedan pendientes.
- **Valor de negocio:** Evita cola administrativa con solicitudes inválidas o duplicadas.
- **Pruebas automatizadas:**
  - `tests/modules/scheduling/test_scheduling_service.py::test_create_scheduling_request_general_consultation_disabled`
  - `tests/modules/scheduling/test_scheduling_service.py::test_create_scheduling_request_general_consultation_enabled`
  - `tests/modules/scheduling/test_scheduling_service.py::test_create_scheduling_request_final_presentation_requires_internship`
  - `tests/modules/scheduling/test_scheduling_service.py::test_create_scheduling_request_final_presentation_missing_evaluations`
  - `tests/modules/scheduling/test_scheduling_service.py::test_create_scheduling_request_final_presentation_success`
  - `tests/modules/scheduling/test_scheduling_service.py::test_create_scheduling_request_duplicate_prevention`
  - `tests/modules/scheduling/test_scheduling_service.py::test_create_scheduling_request_requires_valid_target_coordinator`

### CU-U-SC-06: Administración responde solicitudes respetando permisos y solapamientos

- **Tipo de prueba:** Unitaria
- **Dominio:** Scheduling
- **Contexto:** La administración convierte solicitudes pendientes en citas o las rechaza/cancela.
- **Objetivo:** Validar permisos, asignación, solapamientos y trazabilidad de resolución.
- **Escenario:** Roles administrativos responden solicitudes con éxito, rechazo, cancelación o conflictos de horario.
- **Variantes cubiertas:**
  - Respuesta requiere rol administrativo.
  - Respuesta exitosa agenda la cita.
  - Solapamientos se rechazan.
  - Rechazo de solicitud registra resultado.
  - Cancelación de solicitud registra resultado.
  - Listado de pendientes filtra por coordinador objetivo.
  - Respuesta como director conserva rol resolutor.
- **Resultado esperado:** La resolución de solicitudes mantiene permisos, disponibilidad y trazabilidad.
- **Valor de negocio:** Protege asignación justa y consistente de agenda administrativa.
- **Pruebas automatizadas:**
  - `tests/modules/scheduling/test_scheduling_service.py::test_respond_to_request_requires_admin`
  - `tests/modules/scheduling/test_scheduling_service.py::test_respond_to_request_success`
  - `tests/modules/scheduling/test_scheduling_service.py::test_respond_to_request_overlap_validations`
  - `tests/modules/scheduling/test_scheduling_service.py::test_reject_request`
  - `tests/modules/scheduling/test_scheduling_service.py::test_cancel_request`
  - `tests/modules/scheduling/test_scheduling_service.py::test_list_pending_requests_filters_by_target_coordinator`
  - `tests/modules/scheduling/test_scheduling_service.py::test_respond_as_director_sets_resolved_by_role_director`

### CU-U-SC-07: Configuración de agenda expone opciones correctas por actor

- **Tipo de prueba:** Unitaria
- **Dominio:** Scheduling
- **Contexto:** Frontend necesita saber si hay consultas generales, coordinadores activos y bloqueo de postulaciones.
- **Objetivo:** Validar lectura y modificación de configuración operacional.
- **Escenario:** Se consulta configuración para estudiante y administración, y dirección cambia bloqueo de postulaciones.
- **Variantes cubiertas:**
  - Consulta y toggle de consultas generales.
  - Solo dirección puede cambiar bloqueo de postulaciones.
  - Dirección puede cambiar bloqueo.
  - Configuración estudiantil incluye coordinadores activos.
  - Configuración administrativa incluye bloqueo de postulaciones.
- **Resultado esperado:** La configuración expuesta coincide con permisos y estado operacional.
- **Valor de negocio:** Evita habilitar acciones de agenda no disponibles para el actor.
- **Pruebas automatizadas:**
  - `tests/modules/scheduling/test_scheduling_service.py::test_get_and_toggle_general_consultations_config`
  - `tests/modules/scheduling/test_scheduling_service.py::test_only_director_can_toggle_internship_applications_disabled`
  - `tests/modules/scheduling/test_scheduling_service.py::test_director_can_toggle_internship_applications_disabled`
  - `tests/modules/scheduling/test_scheduling_service.py::test_student_config_includes_active_coordinators`
  - `tests/modules/scheduling/test_scheduling_service.py::test_admin_config_includes_internship_applications_disabled`

### CU-U-SC-08: Agendamiento directo administrativo respeta rol y estado de práctica

- **Tipo de prueba:** Unitaria
- **Dominio:** Scheduling
- **Contexto:** Administración puede agendar directamente sin solicitud previa en casos operativos.
- **Objetivo:** Validar autorización y bloqueo para prácticas finalizadas.
- **Escenario:** Se agenda directamente con rol válido, estudiante intenta agendar y práctica finalizada se rechaza.
- **Variantes cubiertas:**
  - Agendamiento directo exitoso.
  - Estudiante no puede agendar directamente.
  - Práctica finalizada bloquea agendamiento directo.
- **Resultado esperado:** Solo administración agenda directamente prácticas vigentes.
- **Valor de negocio:** Evita saltos de flujo por actores no autorizados o trámites cerrados.
- **Pruebas automatizadas:**
  - `tests/modules/scheduling/test_scheduling_service.py::test_schedule_direct_appointment_success`
  - `tests/modules/scheduling/test_scheduling_service.py::test_schedule_direct_appointment_forbidden_for_student`
  - `tests/modules/scheduling/test_scheduling_service.py::test_schedule_direct_appointment_fails_if_finalized`

### CU-U-SC-09: Confirmación y documentación de cita respetan propietario

- **Tipo de prueba:** Unitaria
- **Dominio:** Scheduling
- **Contexto:** Estudiantes confirman asistencia y pueden asociar documentos a la cita.
- **Objetivo:** Validar propiedad de confirmación y actualización documental.
- **Escenario:** Estudiante confirma cita propia, otro estudiante intenta confirmar y se actualiza documento asociado.
- **Variantes cubiertas:**
  - Confirmación de cita propia exitosa.
  - Confirmación por otro estudiante se rechaza.
  - Actualización de documento de cita exitosa.
- **Resultado esperado:** La cita solo puede confirmarse por el estudiante correspondiente y conserva documento asociado.
- **Valor de negocio:** Protege asistencia y evidencia documental de presentaciones.
- **Pruebas automatizadas:**
  - `tests/modules/scheduling/test_scheduling_service.py::test_confirm_appointment_success`
  - `tests/modules/scheduling/test_scheduling_service.py::test_confirm_appointment_fails_for_other_student`
  - `tests/modules/scheduling/test_scheduling_service.py::test_update_appointment_document_success`

### CU-U-SC-10: Notificaciones de agenda son efecto secundario opcional

- **Tipo de prueba:** Unitaria
- **Dominio:** Scheduling / Notifications
- **Contexto:** Algunas operaciones de agenda notifican al estudiante, pero la acción principal no debe depender de mensajería.
- **Objetivo:** Confirmar degradación segura cuando no hay servicio de notificaciones.
- **Escenario:** Se ejecuta operación de agenda sin servicio de notificaciones configurado.
- **Variantes cubiertas:**
  - Notificación no se despacha cuando el servicio es `None`.
- **Resultado esperado:** La operación principal continúa sin error de mensajería.
- **Valor de negocio:** Evita que un fallo de notificaciones bloquee la agenda.
- **Pruebas automatizadas:**
  - `tests/modules/scheduling/test_scheduling_service.py::test_notification_not_dispatched_when_service_is_none`
