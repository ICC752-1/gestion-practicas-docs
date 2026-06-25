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
