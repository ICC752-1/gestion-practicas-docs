# Casos de Prueba - Documents

## Alcance

Estos casos documentan las pruebas unitarias de valor del módulo `documents`. El foco está en carga documental, validación de archivos, permisos de acceso, revisión, eliminación lógica, documentos sensibles, paquete DIRAE, exportación CSV y contratos ORM.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos verificables que protegen reglas documentales, privacidad y consistencia operativa.

## Unitarias

### CU-U-DO-01: Carga documental válida persiste metadata y archivo privado

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** Un estudiante propietario puede cargar documentos asociados a una práctica vigente.
- **Objetivo:** Validar que una carga válida persiste metadata y escribe el archivo bajo storage privado.
- **Escenario:** El propietario sube un archivo válido para una práctica existente y no terminal.
- **Variantes cubiertas:**
  - Metadata persistida con estado `uploaded`.
  - Archivo físico escrito bajo carpeta de práctica.
- **Resultado esperado:** El documento queda creado con archivo privado y relación correcta con estudiante, práctica y tipo documental.
- **Valor de negocio:** Permite adjuntar antecedentes sin comprometer privacidad del filesystem.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_upload_document_validates_and_persists_metadata`

### CU-U-DO-02: Carga documental rechaza archivo inválido

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** El sistema debe impedir archivos no soportados o que exceden límites.
- **Objetivo:** Validar restricciones de extensión y tamaño.
- **Escenario:** Un estudiante intenta cargar archivos con extensión inválida o tamaño superior al máximo.
- **Variantes cubiertas:**
  - Extensión no permitida.
  - Tamaño mayor al máximo configurado.
- **Resultado esperado:** La carga falla con `400 Bad Request`.
- **Valor de negocio:** Evita documentos inválidos que bloqueen revisión o exportación posterior.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_invalid_extension`
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_size_over_limit`

### CU-U-DO-03: Carga documental valida práctica, tipo, propietario y estado

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** La carga documental debe asociarse a práctica existente, tipo válido y usuario autorizado.
- **Objetivo:** Evitar cargas huérfanas, cruzadas o fuera del ciclo permitido.
- **Escenario:** Se intenta cargar documento con tipo inexistente, práctica inexistente, usuario no propietario o práctica terminal.
- **Variantes cubiertas:**
  - Tipo documental inexistente devuelve `404`.
  - Práctica inexistente devuelve `404`.
  - Estudiante no propietario devuelve `403`.
  - Práctica aprobada sin observación pendiente devuelve `409`.
  - Práctica aprobada con documento observado permite corrección.
  - Diapositivas de presentación pueden cargarse después de aprobación.
- **Resultado esperado:** Solo actores autorizados pueden cargar documentos en estados funcionales válidos.
- **Valor de negocio:** Protege consistencia documental y permite corregir observaciones sin reabrir todo el trámite.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_missing_document_type`
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_missing_internship`
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_non_owner_student`
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_terminal_internship`
  - `tests/modules/documents/test_document_service.py::test_student_can_upload_correction_for_observed_document_after_approval`
  - `tests/modules/documents/test_document_service.py::test_student_cannot_upload_new_document_after_approval_without_observation`
  - `tests/modules/documents/test_document_service.py::test_student_can_upload_presentation_slides_after_approval`

### CU-U-DO-04: Secretaría solo carga documentos administrativos no sensibles

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** Secretaría puede apoyar documentación administrativa, pero no intervenir documentos académicos o sensibles.
- **Objetivo:** Validar restricciones de carga documental por categoría y sensibilidad.
- **Escenario:** Secretaría intenta cargar documentos administrativos, académicos y sensibles.
- **Variantes cubiertas:**
  - Documento administrativo no sensible permitido.
  - Documento académico rechazado.
  - Documento administrativo sensible rechazado.
- **Resultado esperado:** Secretaría solo puede cargar documentos administrativos no sensibles.
- **Valor de negocio:** Protege documentos académicos y antecedentes sensibles.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_secretary_can_upload_non_sensitive_administrative_document`
  - `tests/modules/documents/test_document_service.py::test_secretary_cannot_upload_academic_document`
  - `tests/modules/documents/test_document_service.py::test_secretary_cannot_upload_sensitive_administrative_document`

### CU-U-DO-05: Listado y descarga respetan permisos y sensibilidad

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** Los documentos contienen información privada del estudiante y de la práctica.
- **Objetivo:** Validar que propietario y roles documentales autorizados acceden, mientras accesos cruzados, FICA y documentos sensibles quedan restringidos.
- **Escenario:** Se listan o descargan documentos con distintos roles y documentos sensibles.
- **Variantes cubiertas:**
  - Propietario y rol documental listan documentos.
  - Secretaría no ve documentos sensibles.
  - Estudiante cruzado y FICA son rechazados.
  - Propietario descarga archivo.
  - Secretaría no descarga documento sensible.
- **Resultado esperado:** Solo usuarios autorizados acceden a documentos permitidos.
- **Valor de negocio:** Protege privacidad documental y datos sensibles.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_list_documents_allows_owner_and_admin`
  - `tests/modules/documents/test_document_service.py::test_list_documents_filters_sensitive_documents_for_secretary`
  - `tests/modules/documents/test_document_service.py::test_list_documents_rejects_cross_access`
  - `tests/modules/documents/test_document_service.py::test_list_documents_rejects_fica_role`
  - `tests/modules/documents/test_document_service.py::test_download_allows_owner_and_rejects_cross_access`
  - `tests/modules/documents/test_document_service.py::test_download_rejects_fica_role`
  - `tests/modules/documents/test_document_service.py::test_download_rejects_sensitive_document_for_secretary`

### CU-U-DO-06: Descarga rechaza archivos inexistentes o eliminados

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** La descarga resuelve una clave interna de storage contra archivos privados.
- **Objetivo:** Evitar exposición de documentos faltantes o eliminados.
- **Escenario:** Se prepara descarga de un documento sin archivo físico o marcado como eliminado.
- **Variantes cubiertas:**
  - Archivo físico inexistente devuelve `404`.
  - Documento eliminado devuelve `404`.
- **Resultado esperado:** La API no entrega archivos inválidos.
- **Valor de negocio:** Protege integridad y privacidad del storage documental.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_download_rejects_missing_file`
  - `tests/modules/documents/test_document_service.py::test_download_rejects_deleted_document`

### CU-U-DO-07: Revisión documental respeta comentario obligatorio y trazabilidad

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** Observar un documento requiere comentario accionable para el estudiante.
- **Objetivo:** Validar reglas de revisión y trazabilidad del revisor.
- **Escenario:** Se observa o aprueba un documento con rol documental.
- **Variantes cubiertas:**
  - `observed` sin comentario devuelve `400`.
  - `observed` con comentario actualiza documento.
  - `approved` actualiza documento sin comentario obligatorio.
- **Resultado esperado:** La revisión persiste estado, revisor y comentario cuando corresponde.
- **Valor de negocio:** Protege decisiones documentales y asegura observaciones accionables.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_observed_status_requires_comment`
  - `tests/modules/documents/test_document_service.py::test_observed_status_with_comment_updates_document`
  - `tests/modules/documents/test_document_service.py::test_approved_status_updates_document`

### CU-U-DO-08: Eliminación lógica respeta estado y rol

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** La eliminación documental es lógica y mantiene trazabilidad.
- **Objetivo:** Validar que estudiantes no borren documentos aprobados y que roles documentales puedan eliminar con trazabilidad.
- **Escenario:** Se elimina documento aprobado como estudiante y como rol documental.
- **Variantes cubiertas:**
  - Estudiante no puede eliminar documento aprobado.
  - Rol documental puede eliminar documento aprobado.
- **Resultado esperado:** El documento queda `deleted` solo cuando el actor tiene permiso.
- **Valor de negocio:** Evita pérdida indebida de documentos aceptados y conserva trazabilidad.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_student_cannot_delete_approved_document`
  - `tests/modules/documents/test_document_service.py::test_admin_can_soft_delete_approved_document`

### CU-U-DO-09: Paquete DIRAE exportable exige condiciones completas

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** El paquete DIRAE resume si una práctica puede exportarse para trámite externo.
- **Objetivo:** Validar condiciones de exportabilidad y razones cuando no se cumplen.
- **Escenario:** Se construye paquete con práctica aprobada/finalizada/lista y con distintas condiciones faltantes.
- **Variantes cubiertas:**
  - Paquete completo es exportable.
  - Solicitud no aprobada no exporta.
  - Práctica no finalizada no exporta.
  - Expediente local DIRAE no listo no exporta.
  - Documento requerido faltante no exporta.
  - Documento observado pendiente no exporta.
  - Todos los tipos requeridos deben estar aprobados.
- **Resultado esperado:** `exportable` y `reasons` reflejan reglas de preparación del expediente.
- **Valor de negocio:** Evita exportar expedientes incompletos o en estado incorrecto.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_package_is_exportable_with_approved_internship_and_docs`
  - `tests/modules/documents/test_document_service.py::test_package_not_exportable_when_internship_is_not_approved`
  - `tests/modules/documents/test_document_service.py::test_package_not_exportable_when_practice_is_not_finalized`
  - `tests/modules/documents/test_document_service.py::test_package_not_exportable_when_dirae_status_is_not_ready`
  - `tests/modules/documents/test_document_service.py::test_package_not_exportable_when_required_document_missing`
  - `tests/modules/documents/test_document_service.py::test_package_not_exportable_when_observed_documents_are_pending`
  - `tests/modules/documents/test_document_service.py::test_package_requires_all_required_document_types`

### CU-U-DO-10: Paquete DIRAE selecciona documentos vigentes y últimos aprobados

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** Un estudiante puede subir varias versiones de un documento.
- **Objetivo:** Validar selección por estado, eliminación, fecha e ID.
- **Escenario:** Se construye paquete con documentos aprobados, observados, cargados, eliminados y varias versiones.
- **Variantes cubiertas:**
  - Documentos `uploaded` u `observed` se ignoran.
  - Documentos `deleted` se ignoran.
  - Se selecciona el último aprobado por `upload_date`.
  - Empate por fecha se resuelve por mayor ID.
- **Resultado esperado:** El paquete incluye solo documentos aprobados vigentes y selecciona la versión correcta.
- **Valor de negocio:** Evita exportar documentos obsoletos, observados o eliminados.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_package_ignores_non_approved_documents`
  - `tests/modules/documents/test_document_service.py::test_package_ignores_deleted_document`
  - `tests/modules/documents/test_document_service.py::test_package_selects_latest_approved_document_by_type`
  - `tests/modules/documents/test_document_service.py::test_package_tiebreaks_latest_approved_document_by_id`

### CU-U-DO-11: Paquete DIRAE protege datos sensibles y acceso

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** El paquete DIRAE contiene datos personales, académicos y documentos potencialmente sensibles.
- **Objetivo:** Validar matrícula, permisos y restricción de documentos sensibles por rol.
- **Escenario:** Propietario, roles documentales, FICA y Secretaría consultan paquetes con o sin documentos sensibles.
- **Variantes cubiertas:**
  - Matrícula se construye cuando existe año de admisión.
  - Matrícula queda vacía si no hay año disponible.
  - Propietario y rol documental pueden consultar paquete.
  - Estudiante cruzado y FICA son rechazados.
  - Secretaría no ve documentos sensibles.
  - Director sí puede ver documentos sensibles.
- **Resultado esperado:** El paquete expone datos correctos solo a usuarios autorizados.
- **Valor de negocio:** Protege datos personales y documentos sensibles.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_package_builds_student_enrollment_when_year_is_available`
  - `tests/modules/documents/test_document_service.py::test_package_keeps_student_enrollment_empty_without_year`
  - `tests/modules/documents/test_document_service.py::test_package_access_allows_owner_and_document_admin`
  - `tests/modules/documents/test_document_service.py::test_package_access_rejects_cross_student`
  - `tests/modules/documents/test_document_service.py::test_package_access_rejects_fica_role`
  - `tests/modules/documents/test_document_service.py::test_package_filters_sensitive_documents_for_secretary`
  - `tests/modules/documents/test_document_service.py::test_package_allows_sensitive_documents_for_director`

### CU-U-DO-12: Exportación de expediente DIRAE genera CSV y auditoría estructurada

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** Roles documentales exportan paquetes en CSV para trámite externo.
- **Objetivo:** Validar autorización, contenido CSV, auditoría y errores para solicitudes específicas.
- **Escenario:** Se exportan paquetes con IDs explícitos o sin filtro.
- **Variantes cubiertas:**
  - Rol autorizado genera CSV con columnas y datos esperados.
  - Usuario no documental recibe `403`.
  - ID inexistente devuelve `404`.
  - ID no exportable devuelve `409`.
  - Documento sensible restringido para Secretaría bloquea exportación.
  - Sin IDs y sin exportables retorna solo encabezado.
- **Resultado esperado:** El CSV contiene solo paquetes válidos y el evento de auditoría conserva actor, prácticas y documentos aprobados.
- **Valor de negocio:** Protege generación local del archivo institucional para DIRAE.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_authorized`
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_rejects_non_document_admin`
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_returns_404_for_unknown_requested_id`
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_returns_409_for_requested_non_exportable`
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_rejects_sensitive_document_for_secretary`
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_without_ids_can_return_header_only`

### CU-U-DO-13: Contrato ORM documental mantiene columnas y enums críticos

- **Tipo de prueba:** Unitaria
- **Dominio:** Documents
- **Contexto:** Los modelos ORM deben mantenerse alineados manualmente con el esquema SQL inicial.
- **Objetivo:** Detectar cambios accidentales en tablas, columnas, flags y enums documentales.
- **Escenario:** Se inspeccionan modelos `Document` y `DocumentType`.
- **Variantes cubiertas:**
  - Modelo `Document` conserva columnas críticas.
  - Enum de estado documental conserva valores de negocio.
  - `DocumentType` conserva flag `is_active`.
- **Resultado esperado:** El contrato ORM conserva nombres y valores críticos.
- **Valor de negocio:** Reduce riesgo de desalineación entre modelos, SQL y API.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_model.py::test_document_model_matches_database_contract`
  - `tests/modules/documents/test_document_model.py::test_document_status_enum_matches_business_contract`
  - `tests/modules/documents/test_document_model.py::test_document_type_model_includes_active_flag`
