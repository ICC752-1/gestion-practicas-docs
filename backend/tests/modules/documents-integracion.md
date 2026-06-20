# Casos de Prueba - Documents

## Alcance

Estos casos documentan las pruebas de valor del módulo `documents`. El foco está en carga documental, validación de archivos, storage privado, permisos de acceso, revisión documental, eliminación lógica, paquete DIRAE, exportación CSV y contratos de modelo.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos verificables que protegen reglas documentales, privacidad y consistencia operativa.

## Integración

### CU-I-DO-01: Descarga HTTP exige autenticación y respeta propietario

- Tipo de prueba: Integración
- Dominio: Documents
- Contexto: La descarga debe pasar por API autenticada, no por archivos públicos.
- Objetivo: Validar comportamiento HTTP de descarga para usuario autenticado y estudiante cruzado.
- Escenario: Se solicita descarga sin token, como propietario y como estudiante no propietario.
- Variantes cubiertas:
  - Sin autenticación devuelve `401`.
  - Propietario recibe archivo y `Content-Disposition` correcto.
  - Estudiante no propietario recibe `403`.
- Resultado esperado: Solo usuario autorizado descarga el archivo real.
- Valor de negocio: Protege privacidad documental en el endpoint más sensible.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_router.py::test_download_document_requires_authentication`
  - `tests/modules/documents/test_document_router.py::test_download_document_returns_file_for_owner`
  - `tests/modules/documents/test_document_router.py::test_download_document_rejects_cross_student`

### CU-I-DO-02: Controller delega operaciones documentales al service y preserva contrato

- Tipo de prueba: Integración
- Dominio: Documents
- Contexto: El controller transforma multipart, responses y payloads hacia el service documental.
- Objetivo: Validar que operaciones HTTP principales retornan contratos esperados y delegan argumentos críticos.
- Escenario: Se listan tipos, se carga archivo, se revisa documento, se elimina, se consulta paquete y se exporta CSV usando service doble.
- Variantes cubiertas:
  - Listado de tipos documentales activos.
  - Upload lee contenido del archivo y retorna metadata.
  - Revisión retorna documento actualizado.
  - Delete retorna `204`.
  - Paquete documental retorna resumen.
  - Exportación retorna CSV con media type y filename.
- Resultado esperado: El controller conserva contratos HTTP y delega correctamente.
- Valor de negocio: Evita romper integración frontend-backend de gestión documental.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_router.py::test_list_document_types_returns_active_types`
  - `tests/modules/documents/test_document_router.py::test_upload_document_reads_file_and_returns_metadata`
  - `tests/modules/documents/test_document_router.py::test_update_document_status_returns_reviewed_document`
  - `tests/modules/documents/test_document_router.py::test_delete_document_returns_204`
  - `tests/modules/documents/test_document_router.py::test_get_document_package_returns_summary`
  - `tests/modules/documents/test_document_router.py::test_export_dirae_document_packages_returns_csv`

### CU-I-DO-03: Roles documentales restringen revisión y exportación

- Tipo de prueba: Integración
- Dominio: Documents
- Contexto: Estudiantes pueden cargar y consultar sus documentos, pero no revisar ni exportar paquetes DIRAE.
- Objetivo: Validar dependencias de roles para acciones documentales administrativas.
- Escenario: Un usuario con rol `Estudiante` intenta revisar documento y exportar paquetes DIRAE.
- Variantes cubiertas:
  - Estudiante intenta actualizar estado documental.
  - Estudiante intenta exportar paquetes DIRAE.
- Resultado esperado: Ambas acciones administrativas devuelven `403`.
- Valor de negocio: Evita que estudiantes aprueben documentos o generen exportaciones institucionales.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_router.py::test_update_document_status_rejects_student_role`
  - `tests/modules/documents/test_document_router.py::test_export_dirae_document_packages_rejects_student_role`

### CU-I-DO-04: Controller propaga errores de servicio

- Tipo de prueba: Integración
- Dominio: Documents
- Contexto: El service concentra reglas y errores de negocio; el controller no debe ocultarlos.
- Objetivo: Validar que errores HTTP del service se propagan al consumidor.
- Escenario: El service lanza `HTTPException` al listar documentos de práctica inexistente.
- Variantes cubiertas:
  - Error `404` del service.
- Resultado esperado: El controller propaga el mismo código de error.
- Valor de negocio: Mantiene contratos de error coherentes entre service y API.
- Pruebas automatizadas:
  - `tests/modules/documents/test_document_router.py::test_router_propagates_service_errors`
