# Casos de Prueba - Documents

## Alcance

Estos casos documentan las pruebas de valor del módulo `documents`. El foco está en carga documental, validación de archivos, storage privado, permisos de acceso, revisión documental, eliminación lógica, paquete DIRAE, exportación CSV y contratos de modelo.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos verificables que protegen reglas documentales, privacidad y consistencia operativa.

## Unitarias

### CU-U-DO-01: Carga documental válida persiste metadata y archivo privado

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: Un estudiante propietario puede cargar documentos asociados a una práctica vigente.
- Objetivo: Validar que una carga válida persiste metadata, escribe el archivo en storage privado y conserva relación con estudiante, práctica y tipo documental.
- Escenario: El propietario sube un archivo válido para una práctica existente y no terminal.
- Variantes cubiertas:
  - Metadata de documento persistida con estado `uploaded`.
  - Archivo físico escrito bajo carpeta de práctica.
  - Nombre con path se normaliza para evitar escape de storage.
- Resultado esperado: El documento queda creado, con archivo privado y sin exponer paths arbitrarios.
- Valor de negocio: Permite al estudiante adjuntar antecedentes sin comprometer privacidad del filesystem.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_upload_document_validates_and_persists_metadata`
  - `tests/modules/documents/test_document_service.py::test_upload_normalizes_file_name_and_keeps_storage_private`

### CU-U-DO-02: Carga documental rechaza archivo inválido

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: El sistema debe impedir archivos no soportados o sin contenido útil.
- Objetivo: Validar restricciones de extensión, tamaño y contenido vacío.
- Escenario: Un estudiante intenta cargar archivos con extensión inválida, tamaño sobre límite o contenido vacío.
- Variantes cubiertas:
  - Extensión no permitida.
  - Tamaño mayor al máximo configurado.
  - Archivo vacío.
- Resultado esperado: La carga falla con `400 Bad Request`.
- Valor de negocio: Evita documentos inválidos que bloqueen revisión o exportación posterior.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_invalid_extension`
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_size_over_limit`
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_empty_file`

### CU-U-DO-03: Carga documental valida práctica, tipo documental y propietario

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: La carga documental debe asociarse a una práctica existente, tipo documental activo y estudiante propietario.
- Objetivo: Evitar cargas huérfanas, cruzadas o sobre prácticas cerradas.
- Escenario: Se intenta cargar documento con tipo inexistente, práctica inexistente, usuario no propietario o práctica terminal.
- Variantes cubiertas:
  - Tipo documental inexistente devuelve `404`.
  - Práctica inexistente devuelve `404`.
  - Estudiante no propietario devuelve `403`.
  - Práctica terminal devuelve `409`.
- Resultado esperado: Solo el propietario de una práctica vigente puede cargar documentos válidos.
- Valor de negocio: Protege consistencia documental y evita modificaciones fuera del ciclo permitido.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_missing_document_type`
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_missing_internship`
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_non_owner_student`
  - `tests/modules/documents/test_document_service.py::test_upload_rejects_terminal_internship`

### CU-U-DO-04: Carga documental limpia archivo si falla persistencia

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: La escritura física del archivo ocurre antes de persistir metadata. Si la persistencia falla, no debe quedar basura huérfana.
- Objetivo: Validar rollback del archivo escrito cuando falla la creación de metadata.
- Escenario: El archivo se escribe correctamente, pero el repositorio falla al crear el documento.
- Variantes cubiertas:
  - Falla de persistencia posterior a escritura física.
- Resultado esperado: El archivo recién escrito se elimina y no quedan archivos huérfanos.
- Valor de negocio: Mantiene alineados storage físico y metadata documental.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_upload_removes_written_file_when_metadata_persistence_fails`

### CU-U-DO-05: Listado y descarga respetan permisos documentales

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: Los documentos contienen información privada del estudiante y de la práctica.
- Objetivo: Validar que propietario y roles documentales pueden consultar, mientras estudiantes cruzados son rechazados.
- Escenario: Se listan o descargan documentos como propietario, rol documental y estudiante no propietario.
- Variantes cubiertas:
  - Propietario lista documentos.
  - Rol documental lista documentos.
  - Estudiante no propietario no lista.
  - Propietario descarga archivo.
  - Estudiante no propietario no descarga.
- Resultado esperado: Solo propietario o rol documental acceden a documentos.
- Valor de negocio: Protege privacidad documental y datos de estudiantes.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_list_documents_allows_owner_and_admin`
  - `tests/modules/documents/test_document_service.py::test_list_documents_rejects_cross_access`
  - `tests/modules/documents/test_document_service.py::test_download_allows_owner_and_rejects_cross_access`

### CU-U-DO-06: Descarga rechaza archivos inexistentes, eliminados o paths inseguros

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: La descarga resuelve una clave interna de storage contra un directorio privado.
- Objetivo: Evitar exposición de archivos faltantes, eliminados o fuera del storage permitido.
- Escenario: Se prepara descarga de documento con archivo faltante, eliminado o `file_path` inseguro.
- Variantes cubiertas:
  - Archivo físico inexistente devuelve `404`.
  - Documento eliminado devuelve `404`.
  - `file_path` con `..` devuelve `400`.
  - `file_path` absoluto devuelve `400`.
- Resultado esperado: La API no entrega archivos inválidos ni resuelve rutas fuera del storage privado.
- Valor de negocio: Protege privacidad y seguridad del filesystem del servidor.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_download_rejects_missing_file`
  - `tests/modules/documents/test_document_service.py::test_download_rejects_deleted_document`
  - `tests/modules/documents/test_document_service.py::test_download_rejects_unsafe_storage_key`

### CU-U-DO-07: Revisión documental respeta roles, estados y comentario obligatorio

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: Solo roles documentales pueden aprobar u observar documentos. Observar requiere comentario para que el estudiante pueda corregir.
- Objetivo: Validar reglas de revisión documental.
- Escenario: Se intenta observar o aprobar documentos con distintos roles y estados.
- Variantes cubiertas:
  - `observed` sin comentario devuelve `400`.
  - `observed` con comentario actualiza documento.
  - `approved` actualiza documento sin comentario obligatorio.
  - Estudiante no puede revisar.
  - Documento eliminado no puede revisarse.
- Resultado esperado: La revisión solo procede con rol documental, documento vigente y payload válido.
- Valor de negocio: Protege decisiones documentales y asegura observaciones accionables.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_observed_status_requires_comment`
  - `tests/modules/documents/test_document_service.py::test_observed_status_with_comment_updates_document`
  - `tests/modules/documents/test_document_service.py::test_approved_status_updates_document`
  - `tests/modules/documents/test_document_service.py::test_student_cannot_review_document`
  - `tests/modules/documents/test_document_service.py::test_review_rejects_deleted_document`

### CU-U-DO-08: Eliminación lógica respeta propietario, roles y estado aprobado

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: La eliminación documental es lógica y mantiene trazabilidad. El estudiante no debe borrar documentos ya aprobados.
- Objetivo: Validar reglas de eliminación por estado y actor.
- Escenario: Se elimina un documento como propietario, estudiante cruzado o rol documental.
- Variantes cubiertas:
  - Estudiante no puede eliminar documento aprobado.
  - Estudiante no propietario no puede eliminar documento ajeno.
  - Propietario puede eliminar documento `uploaded` u `observed`.
  - Rol documental puede eliminar documento aprobado.
- Resultado esperado: El documento queda `deleted` solo cuando el actor tiene permiso y el estado lo permite.
- Valor de negocio: Evita pérdida lógica indebida de documentos ya aceptados y conserva trazabilidad.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_student_cannot_delete_approved_document`
  - `tests/modules/documents/test_document_service.py::test_student_cannot_delete_document_from_another_student`
  - `tests/modules/documents/test_document_service.py::test_owner_can_delete_non_approved_document`
  - `tests/modules/documents/test_document_service.py::test_admin_can_soft_delete_approved_document`

### CU-U-DO-09: Paquete DIRAE exportable requiere práctica aprobada y documentos requeridos aprobados

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: El paquete documental resume si una práctica puede exportarse a DIRAE.
- Objetivo: Validar condiciones de exportabilidad y razones cuando no se cumplen.
- Escenario: Se construye paquete con práctica aprobada, práctica no aprobada y documentos requeridos faltantes.
- Variantes cubiertas:
  - Práctica aprobada con documentos requeridos aprobados es exportable.
  - Práctica no aprobada no es exportable.
  - Documento requerido faltante no es exportable.
  - Múltiples tipos requeridos deben estar aprobados.
- Resultado esperado: `exportable` y `reasons` reflejan reglas DIRAE.
- Valor de negocio: Evita exportar trámites incompletos o no aprobados.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_package_is_exportable_with_approved_internship_and_docs`
  - `tests/modules/documents/test_document_service.py::test_package_not_exportable_when_internship_is_not_approved`
  - `tests/modules/documents/test_document_service.py::test_package_not_exportable_when_required_document_missing`
  - `tests/modules/documents/test_document_service.py::test_package_requires_all_required_document_types`

### CU-U-DO-10: Paquete DIRAE selecciona documentos vigentes y últimos aprobados

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: Un estudiante puede subir varias versiones de un documento. El paquete debe elegir la versión aprobada vigente más reciente.
- Objetivo: Validar selección documental por estado, eliminación, fecha e ID.
- Escenario: Se construye paquete con documentos aprobados, observados, cargados, eliminados y varias versiones.
- Variantes cubiertas:
  - Documentos `uploaded` u `observed` se ignoran.
  - Documentos `deleted` o con `deleted_at` se ignoran.
  - Se selecciona el último aprobado por `upload_date`.
  - Empate por fecha se resuelve por mayor ID.
  - Documentos opcionales aprobados aparecen en `optional_documents`.
- Resultado esperado: El paquete incluye solo documentos aprobados vigentes y selecciona la versión correcta.
- Valor de negocio: Evita exportar documentos obsoletos, observados o eliminados.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_package_ignores_non_approved_documents`
  - `tests/modules/documents/test_document_service.py::test_package_ignores_deleted_document`
  - `tests/modules/documents/test_document_service.py::test_package_selects_latest_approved_document_by_type`
  - `tests/modules/documents/test_document_service.py::test_package_tiebreaks_latest_approved_document_by_id`
  - `tests/modules/documents/test_document_service.py::test_package_includes_approved_optional_documents`

### CU-U-DO-11: Paquete DIRAE construye datos del estudiante y controla acceso

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: El paquete DIRAE contiene datos personales y académicos del estudiante.
- Objetivo: Validar matrícula derivada, ausencia segura de matrícula y permisos de consulta.
- Escenario: Se consulta paquete como propietario, rol documental y estudiante cruzado, con y sin año de ingreso.
- Variantes cubiertas:
  - Matrícula se construye cuando existe año de admisión.
  - Matrícula queda vacía si no hay año disponible.
  - Propietario y rol documental pueden consultar paquete.
  - Estudiante no propietario recibe `403`.
- Resultado esperado: El paquete expone datos correctos solo a usuarios autorizados.
- Valor de negocio: Protege datos personales y asegura consistencia de exportación.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_package_builds_student_enrollment_when_year_is_available`
  - `tests/modules/documents/test_document_service.py::test_package_keeps_student_enrollment_empty_without_year`
  - `tests/modules/documents/test_document_service.py::test_package_access_allows_owner_and_document_admin`
  - `tests/modules/documents/test_document_service.py::test_package_access_rejects_cross_student`

### CU-U-DO-12: Exportación DIRAE genera CSV y auditoría estructurada

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: Roles documentales pueden exportar paquetes documentales DIRAE en CSV.
- Objetivo: Validar autorización, contenido CSV, auditoría y errores para solicitudes específicas.
- Escenario: Se exportan paquetes con IDs explícitos o sin filtro.
- Variantes cubiertas:
  - Rol autorizado genera CSV con columnas y datos esperados.
  - Usuario no documental recibe `403`.
  - ID solicitado inexistente devuelve `404`.
  - ID solicitado no exportable devuelve `409`.
  - Sin IDs y sin exportables retorna solo encabezado.
  - Sin IDs exporta solo paquetes exportables e ignora no exportables.
- Resultado esperado: El CSV contiene solo paquetes válidos y el evento de auditoría conserva actor, prácticas y documentos aprobados.
- Valor de negocio: Protege la entrega formal de antecedentes a DIRAE.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_authorized`
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_rejects_non_document_admin`
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_returns_404_for_unknown_requested_id`
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_returns_409_for_requested_non_exportable`
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_without_ids_can_return_header_only`
  - `tests/modules/documents/test_document_service.py::test_export_dirae_csv_without_ids_exports_only_exportable_packages`

### CU-U-DO-13: Contrato ORM documental mantiene columnas y enums críticos

- Tipo de prueba: Unitaria
- Dominio: Documents
- Contexto: El proyecto no usa Alembic; los modelos ORM deben mantenerse alineados manualmente con el esquema SQL inicial.
- Objetivo: Detectar cambios accidentales en tablas, columnas, flags y enums documentales.
- Escenario: Se inspeccionan modelos `Document` y `DocumentType`.
- Variantes cubiertas:
  - Modelo `Document` conserva columnas críticas.
  - Enum de estado documental conserva valores de negocio.
  - `DocumentType` conserva flag `is_active`.
- Resultado esperado: El contrato ORM conserva nombres y valores críticos del módulo.
- Valor de negocio: Reduce riesgo de desalineación entre modelos, SQL, API y tests.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_model.py::test_document_model_matches_database_contract`
  - `tests/modules/documents/test_document_model.py::test_document_status_enum_matches_business_contract`
  - `tests/modules/documents/test_document_model.py::test_document_type_model_includes_active_flag`
