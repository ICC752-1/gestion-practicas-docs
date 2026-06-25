# Casos de Prueba - Documents

## Alcance

Estos casos documentan pruebas de integración liviana del módulo `documents`. El foco está en contratos HTTP observables, descarga autenticada, transformación de upload, permisos y errores propagados por el controller.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos relevantes de integración interna.

## Integración

### CU-I-DO-01: Descarga HTTP exige autenticación y respeta propietario

- **Tipo de prueba:** Integración
- **Dominio:** Documents
- **Contexto:** La descarga debe pasar por API autenticada, no por archivos públicos.
- **Objetivo:** Validar comportamiento HTTP de descarga para usuario autenticado y estudiante cruzado.
- **Escenario:** Se solicita descarga sin token, como propietario y como estudiante no propietario.
- **Variantes cubiertas:**
  - Sin autenticación devuelve `401`.
  - Propietario recibe archivo y `Content-Disposition` correcto.
  - Estudiante no propietario recibe `403`.
- **Resultado esperado:** Solo usuario autorizado descarga el archivo real.
- **Valor de negocio:** Protege privacidad documental en el endpoint más sensible.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_router.py::test_download_document_requires_authentication`
  - `tests/modules/documents/test_document_router.py::test_download_document_returns_file_for_owner`
  - `tests/modules/documents/test_document_router.py::test_download_document_rejects_cross_student`

### CU-I-DO-02: Controller transforma upload multipart hacia metadata documental

- **Tipo de prueba:** Integración
- **Dominio:** Documents
- **Contexto:** La carga HTTP recibe `UploadFile` y debe entregar bytes reales al service.
- **Objetivo:** Validar que el controller lee el contenido del archivo y retorna metadata pública.
- **Escenario:** Se invoca el endpoint de carga con un archivo simulado.
- **Variantes cubiertas:**
  - El contenido binario llega al service.
  - La respuesta contiene metadata del documento creado.
- **Resultado esperado:** El contrato multipart queda preservado.
- **Valor de negocio:** Evita romper la integración frontend-backend de carga documental.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_router.py::test_upload_document_reads_file_and_returns_metadata`

### CU-I-DO-03: Roles documentales restringen revisión y exportación

- **Tipo de prueba:** Integración
- **Dominio:** Documents
- **Contexto:** Estudiantes y FICA no deben revisar documentos ni exportar paquetes DIRAE.
- **Objetivo:** Validar dependencias de roles para acciones documentales administrativas.
- **Escenario:** Usuarios no documentales intentan revisar documentos y exportar paquetes.
- **Variantes cubiertas:**
  - Roles no documentales reciben `403` al revisar.
  - Roles no documentales reciben `403` al exportar.
- **Resultado esperado:** Solo roles documentales autorizados ejecutan acciones administrativas.
- **Valor de negocio:** Evita aprobaciones documentales o exportaciones institucionales no autorizadas.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_router.py::test_update_document_status_rejects_non_document_admin_roles`
  - `tests/modules/documents/test_document_router.py::test_export_dirae_document_packages_rejects_non_document_admin_roles`

### CU-I-DO-04: Exportación HTTP de paquetes DIRAE conserva contrato CSV

- **Tipo de prueba:** Integración
- **Dominio:** Documents
- **Contexto:** El endpoint de exportación debe entregar un archivo CSV descargable.
- **Objetivo:** Validar media type, encabezado de descarga y contenido inicial del CSV.
- **Escenario:** Un rol autorizado exporta paquetes DIRAE con IDs explícitos.
- **Variantes cubiertas:**
  - `media_type` de CSV.
  - Header `Content-Disposition` con nombre `.csv`.
  - Contenido incluye columnas esperadas.
- **Resultado esperado:** El frontend recibe un archivo CSV descargable.
- **Valor de negocio:** Protege el contrato de exportación institucional.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_router.py::test_export_dirae_document_packages_returns_csv`

### CU-I-DO-05: Controller propaga errores de servicio

- **Tipo de prueba:** Integración
- **Dominio:** Documents
- **Contexto:** El service concentra reglas y errores de negocio; el controller no debe ocultarlos.
- **Objetivo:** Validar que errores HTTP del service se propagan al consumidor.
- **Escenario:** El service lanza `HTTPException` al listar documentos de una práctica inexistente.
- **Variantes cubiertas:**
  - Error `404` del service.
- **Resultado esperado:** El controller propaga el mismo código de error.
- **Valor de negocio:** Mantiene contratos de error coherentes entre service y API.
- **Pruebas automatizadas:**
  - `tests/modules/documents/test_document_router.py::test_router_propagates_service_errors`
