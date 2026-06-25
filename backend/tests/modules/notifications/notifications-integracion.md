# Casos de Prueba - Notifications

## Alcance

Estos casos documentan las pruebas de integración liviana del módulo `notifications`. El foco está en contratos del controller para bandeja personal, detalle privado, reintento y eventos generados por otros servicios.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos verificables que protegen privacidad y comunicación del sistema.

## Integración

### CU-I-NO-01: Usuario autenticado lista su bandeja de notificaciones

- **Tipo de prueba:** Integración
- **Dominio:** Notifications
- **Contexto:** El listado de notificaciones es una bandeja personal asociada al usuario autenticado.
- **Objetivo:** Validar que el controller consulta notificaciones usando `current_user.id`, paginación y contadores.
- **Escenario:** Un usuario autenticado solicita su listado.
- **Variantes cubiertas:**
  - Se retornan items propios.
  - Se informa total y contador de no leídas.
- **Resultado esperado:** La bandeja se construye desde el usuario autenticado.
- **Valor de negocio:** Protege separación de bandejas entre usuarios.
- **Pruebas automatizadas:**
  - `tests/modules/notifications/test_notification_controller.py::TestListNotifications::test_list_returns_notifications_for_authenticated_user`

### CU-I-NO-02: Usuario solo consulta detalle de notificaciones propias

- **Tipo de prueba:** Integración
- **Dominio:** Notifications
- **Contexto:** El detalle de una notificación puede contener contenido administrativo y payload del trámite.
- **Objetivo:** Validar control de acceso sobre notificaciones individuales.
- **Escenario:** Se consulta una notificación propia, una ajena y una inexistente.
- **Variantes cubiertas:**
  - Usuario destinatario accede al detalle.
  - Usuario no destinatario recibe `403`.
  - Notificación inexistente devuelve `404`.
- **Resultado esperado:** Solo el destinatario puede ver el detalle de su notificación.
- **Valor de negocio:** Evita exposición de información administrativa entre usuarios.
- **Pruebas automatizadas:**
  - `tests/modules/notifications/test_notification_controller.py::TestGetNotification::test_get_detail_of_own_notification`
  - `tests/modules/notifications/test_notification_controller.py::TestGetNotification::test_get_detail_of_other_users_notification_raises_403`
  - `tests/modules/notifications/test_notification_controller.py::TestGetNotification::test_get_detail_of_nonexistent_notification_raises_404`

### CU-I-NO-03: Controller de reintento traduce resultado operacional

- **Tipo de prueba:** Integración
- **Dominio:** Notifications
- **Contexto:** El endpoint de reintento debe exponer de forma clara si el reenvío funcionó o no era elegible.
- **Objetivo:** Validar la traducción controller-service para reintentos manuales.
- **Escenario:** Un rol autorizado reintenta una notificación y el service retorna éxito o `None`.
- **Variantes cubiertas:**
  - Reintento exitoso responde `success=True`.
  - Notificación inexistente o no elegible responde `404`.
- **Resultado esperado:** La respuesta HTTP comunica correctamente el resultado operacional.
- **Valor de negocio:** Da feedback confiable a roles administrativos que recuperan envíos fallidos.
- **Pruebas automatizadas:**
  - `tests/modules/notifications/test_notification_controller.py::TestRetryNotification::test_retry_returns_response_on_success`
  - `tests/modules/notifications/test_notification_controller.py::TestRetryNotification::test_retry_returns_404_when_notification_not_found`

### CU-I-NO-04: Eventos de solicitud y expediente se persisten en modo simulado

- **Tipo de prueba:** Integración
- **Dominio:** Notifications / Internships
- **Contexto:** Las acciones sobre solicitudes y expediente DIRAE generan notificaciones.
- **Objetivo:** Validar que el flujo real de servicios produce y persiste eventos esperados sin SMTP real.
- **Escenario:** Se crea una solicitud, se aprueba, se rechaza y se deriva expediente con repositorio de notificaciones en memoria.
- **Variantes cubiertas:**
  - Solicitud creada genera evento `internship_created`.
  - Aprobación genera `internship_approved`.
  - Rechazo conserva motivo en payload.
  - Derivación conserva motivo en payload.
- **Resultado esperado:** Todas las notificaciones se persisten con estado `simulated` y payload funcional correcto.
- **Valor de negocio:** Protege comunicación asociada al ciclo administrativo de prácticas.
- **Pruebas automatizadas:**
  - `tests/modules/notifications/test_notification_event_integration.py::test_internship_lifecycle_events_are_persisted_in_simulated_mode`

### CU-I-NO-05: Eventos documentales se persisten en modo simulado

- **Tipo de prueba:** Integración
- **Dominio:** Notifications / Documents
- **Contexto:** La carga y revisión documental deben generar notificaciones para roles revisores y estudiantes.
- **Objetivo:** Validar que el servicio documental emite eventos persistentes al cargar y revisar documentos.
- **Escenario:** Un estudiante carga un documento; luego un revisor lo observa y después lo aprueba.
- **Variantes cubiertas:**
  - Carga documental genera evento `document_uploaded`.
  - Observación genera `document_status_changed` con comentario.
  - Aprobación genera `document_status_changed` sin comentario.
- **Resultado esperado:** Las notificaciones se persisten como `simulated` con payload documental correcto.
- **Valor de negocio:** Protege trazabilidad y comunicación de cambios documentales relevantes.
- **Pruebas automatizadas:**
  - `tests/modules/notifications/test_notification_event_integration.py::test_document_events_are_persisted_in_simulated_mode`
