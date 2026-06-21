# Casos de Prueba - Notifications

## Alcance

Estos casos documentan las pruebas de valor del módulo `notifications`. El foco está en persistencia de notificaciones, modos de despacho `simulated` y `real`, reintentos operativos, permisos de consulta, endpoint legacy de envío SMTP directo, contratos de eventos y seguridad del contenido HTML generado.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos verificables que protegen reglas operativas o contratos relevantes del sistema.

## Integración

### CU-I-NO-01: Usuario autenticado lista sus notificaciones

- Tipo de prueba: Integración
- Dominio: Notifications
- Contexto: El listado de notificaciones es una bandeja personal asociada al usuario autenticado.
- Objetivo: Validar que el controller consulta notificaciones usando `current_user.id` y respeta paginación.
- Escenario: Un usuario autenticado solicita su listado con `limit` y `offset`.
- Variantes cubiertas:
  - Consulta delegada al service con usuario autenticado.
  - Parámetros de paginación se propagan.
- Resultado esperado: Se retornan solo las notificaciones del usuario consultado por el service.
- Valor de negocio: Protege la separación de bandejas entre usuarios.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_controller.py::TestListNotifications::test_list_returns_notifications_for_authenticated_user`

### CU-I-NO-02: Usuario solo consulta detalle de notificaciones propias

- Tipo de prueba: Integración
- Dominio: Notifications
- Contexto: El detalle de una notificación puede contener contenido administrativo y payload del trámite.
- Objetivo: Validar control de acceso sobre notificaciones individuales.
- Escenario: Se consulta una notificación propia, una ajena y una inexistente.
- Variantes cubiertas:
  - Usuario destinatario accede al detalle.
  - Usuario no destinatario recibe `403 Forbidden`.
  - Notificación inexistente devuelve `404 Not Found`.
- Resultado esperado: Solo el destinatario puede ver el detalle de su notificación.
- Valor de negocio: Evita exposición de información administrativa entre usuarios.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_controller.py::TestGetNotification::test_get_detail_of_own_notification`
  - `tests/modules/notifications/test_notification_controller.py::TestGetNotification::test_get_detail_of_other_users_notification_raises_403`
  - `tests/modules/notifications/test_notification_controller.py::TestGetNotification::test_get_detail_of_nonexistent_notification_raises_404`

### CU-I-NO-03: Controller de reintento traduce resultado operacional

- Tipo de prueba: Integración
- Dominio: Notifications
- Contexto: El endpoint de reintento debe exponer de forma clara si el reenvío funcionó, falló o no era elegible.
- Objetivo: Validar la traducción controller-service para reintentos manuales.
- Escenario: Un rol autorizado reintenta una notificación y el service retorna éxito, fallo o `None`.
- Variantes cubiertas:
  - Reintento exitoso responde `success=True`.
  - Reintento ejecutado pero fallido responde `success=False` con estado `failed`.
  - Notificación inexistente o no elegible responde `404`.
- Resultado esperado: La respuesta HTTP comunica correctamente el resultado operacional.
- Valor de negocio: Da feedback confiable a roles administrativos que recuperan envíos fallidos.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_controller.py::TestRetryNotification::test_retry_returns_response_on_success`
  - `tests/modules/notifications/test_notification_controller.py::TestRetryNotification::test_retry_returns_failed_response_when_smtp_retry_fails`
  - `tests/modules/notifications/test_notification_controller.py::TestRetryNotification::test_retry_returns_404_when_notification_not_found`

### CU-I-NO-04: Endpoint legacy de envío directo traduce errores SMTP

- Tipo de prueba: Integración
- Dominio: Notifications
- Contexto: `POST /notifications/send-email` es un flujo legacy de SMTP directo que no persiste notificaciones, pero sigue expuesto a roles administrativos.
- Objetivo: Validar que el controller responde correctamente ante éxito, falta de mailer y fallos inesperados.
- Escenario: Un rol autorizado solicita envío directo con un payload válido.
- Variantes cubiertas:
  - Envío directo exitoso devuelve respuesta positiva.
  - Mailer no configurado devuelve `400 Bad Request`.
  - Error SMTP inesperado devuelve `500 Internal Server Error`.
- Resultado esperado: El endpoint no oculta fallos de configuración ni errores operativos.
- Valor de negocio: Permite diagnosticar correctamente un flujo legacy sin confundirlo con notificaciones persistentes.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_controller.py::TestSendNotification::test_send_notification_returns_success_on_smtp_send`
  - `tests/modules/notifications/test_notification_controller.py::TestSendNotification::test_send_notification_returns_400_when_mailer_is_not_configured`
  - `tests/modules/notifications/test_notification_controller.py::TestSendNotification::test_send_notification_returns_500_on_unexpected_smtp_error`

### CU-I-NO-05: Eventos de solicitud y preparación de expediente se persisten en modo simulado

- Tipo de prueba: Integración
- Dominio: Notifications / Internships
- Contexto: Las acciones sobre solicitudes de práctica y preparación local del expediente para DIRAE generan notificaciones para comunicación interna y al estudiante.
- Objetivo: Validar que el flujo real de servicios produce y persiste eventos esperados sin SMTP real.
- Escenario: Se crea una solicitud de práctica y luego se aprueba, rechaza y prepara su expediente local usando servicios de dominio con repositorio de notificaciones en memoria.
- Variantes cubiertas:
  - Solicitud de práctica creada genera evento `internship_created` como `custom`.
  - Aprobación de solicitud genera `internship_approved`.
  - Rechazo de solicitud conserva motivo en payload.
  - Preparación del expediente local conserva motivo en payload.
- Resultado esperado: Todas las notificaciones se persisten con estado `simulated` y payload funcional correcto.
- Valor de negocio: Protege la comunicación asociada al ciclo administrativo de solicitudes y a la preparación local del expediente que luego se tramita externamente.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_event_integration.py::test_internship_lifecycle_events_are_persisted_in_simulated_mode`

### CU-I-NO-06: Eventos documentales se persisten en modo simulado

- Tipo de prueba: Integración
- Dominio: Notifications / Documents
- Contexto: La carga y revisión documental deben generar notificaciones para roles revisores y estudiantes.
- Objetivo: Validar que el servicio documental emite eventos persistentes al cargar y revisar documentos.
- Escenario: Un estudiante carga un documento; luego un revisor lo observa y después lo aprueba.
- Variantes cubiertas:
  - Carga documental genera evento `document_uploaded`.
  - Observación genera `document_status_changed` con comentario.
  - Aprobación genera `document_status_changed` sin comentario.
- Resultado esperado: Las notificaciones se persisten como `simulated` con payload documental correcto.
- Valor de negocio: Protege trazabilidad y comunicación de cambios documentales relevantes.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_event_integration.py::test_document_events_are_persisted_in_simulated_mode`
