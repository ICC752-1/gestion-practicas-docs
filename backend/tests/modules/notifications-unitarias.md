# Casos de Prueba - Notifications

## Alcance

Estos casos documentan las pruebas unitarias de valor del módulo `notifications`. El foco está en persistencia de notificaciones, modos de despacho `simulated` y `real`, reintentos operativos, contratos de eventos, seguridad del contenido HTML y efectos secundarios no bloqueantes.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos verificables que protegen reglas operativas o contratos relevantes.

## Unitarias

### CU-U-NO-01: Persistir notificaciones simuladas sin invocar SMTP

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: En desarrollo y pruebas, `NOTIFICATION_MODE=simulated` debe registrar notificaciones sin enviar correos reales.
- Objetivo: Confirmar que el sistema persiste la notificación como `simulated` y no usa transporte SMTP.
- Escenario: Se crea una notificación con el servicio configurado en modo simulado.
- Variantes cubiertas:
  - La notificación queda con estado `simulated`.
  - El mailer SMTP no se invoca.
- Resultado esperado: La notificación se persiste y no se realiza envío SMTP.
- Valor de negocio: Permite probar flujos con notificaciones sin riesgo de enviar correos reales.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestCreateSimulatedNotification::test_simulated_mode_persists_with_simulated_status`
  - `tests/modules/notifications/test_notification_service.py::TestCreateSimulatedNotification::test_simulated_mode_does_not_invoke_smtp`

### CU-U-NO-02: Enviar notificaciones reales y registrar resultado de entrega

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: En modo `real`, una notificación persistente debe intentar envío SMTP y actualizar estado según resultado.
- Objetivo: Validar el ciclo `pending` -> `sent` o `failed`.
- Escenario: El servicio crea una notificación en modo real y el transporte SMTP responde con éxito o error.
- Variantes cubiertas:
  - SMTP exitoso invoca el mailer.
  - Error SMTP actualiza la notificación como `failed`.
- Resultado esperado: El estado final refleja el resultado real del intento de envío.
- Valor de negocio: Permite operar notificaciones reales con trazabilidad de entrega y fallos.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestCreateRealNotification::test_real_mode_persists_and_sends_smtp`
  - `tests/modules/notifications/test_notification_service.py::TestCreateRealNotification::test_real_mode_smtp_failure_marks_as_failed`

### CU-U-NO-03: Evitar envío real cuando la configuración SMTP no está lista

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: El modo `real` requiere credenciales SMTP válidas.
- Objetivo: Verificar que credenciales por defecto no intentan correos reales y que el envío directo falla sin mailer.
- Escenario: El servicio se inicializa en modo real con credenciales de prueba o intenta envío directo sin mailer.
- Variantes cubiertas:
  - Modo `real` con credenciales por defecto persiste como `simulated`.
  - `send_email` falla con `RuntimeError` si no existe mailer SMTP.
- Resultado esperado: No se intenta SMTP real y el error es explícito cuando se solicita envío directo.
- Valor de negocio: Reduce riesgo de configuraciones ambiguas.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestCreateRealNotification::test_real_mode_falls_back_to_simulated_with_default_credentials`
  - `tests/modules/notifications/test_notification_service.py::TestInvalidRecipient::test_send_email_raises_runtime_error_in_simulated_mode`

### CU-U-NO-04: Rechazar envío SMTP persistente sin destinatario de correo

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: Una notificación persistente puede tener destinatario interno, pero SMTP necesita correo válido.
- Objetivo: Evitar intentos de envío sin destinatario de email.
- Escenario: Se intenta despachar por SMTP una notificación sin `recipient_email`.
- Variantes cubiertas:
  - Notificación persistente sin correo de destinatario.
- Resultado esperado: El envío SMTP falla con error de destinatario inválido.
- Valor de negocio: Evita errores silenciosos y facilita diagnóstico operativo.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestInvalidRecipient::test_send_smtp_raises_on_no_recipient_email`

### CU-U-NO-05: Helpers de eventos conservan contrato de ruteo y payload

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: Los módulos productores construyen notificaciones mediante helpers con payload consumido por frontend y auditoría.
- Objetivo: Validar que eventos críticos mantienen destinatario, `event_type` y metadata funcional.
- Escenario: Se generan notificaciones para aprobación, rechazo, derivación y cambio de requisito.
- Variantes cubiertas:
  - Aprobación conserva `internship_id`.
  - Rechazo conserva motivo.
  - Derivación conserva motivo.
  - Cambio de requisito conserva identificador, tipo y estados.
- Resultado esperado: Las notificaciones mantienen contratos estables de ruteo y payload.
- Valor de negocio: Evita romper consumidores internos que interpretan notificaciones por tipo y metadata.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestPayloadStorage::test_internship_event_helpers_keep_routing_and_payload_contract`
  - `tests/modules/notifications/test_notification_service.py::TestPayloadStorage::test_requirement_status_changed_notification_keeps_payload_contract`

### CU-U-NO-06: Contenido HTML omite datos vacíos y escapa valores dinámicos

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: Las notificaciones se renderizan como HTML y pueden incluir datos ingresados por usuarios o administradores.
- Objetivo: Evitar filas vacías e inyección HTML en correos transaccionales.
- Escenario: Se construyen notificaciones con motivo ausente y valores dinámicos con HTML.
- Variantes cubiertas:
  - Rechazo sin motivo no muestra fila `Motivo`.
  - Organización y motivo con tags HTML se escapan.
- Resultado esperado: El contenido no incluye filas vacías ni HTML dinámico sin escapar.
- Valor de negocio: Mejora seguridad y calidad de correos enviados a usuarios.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestPayloadStorage::test_event_helpers_escape_dynamic_html_values`
  - `tests/modules/notifications/test_notification_service.py::TestPayloadStorage::test_rejected_notification_without_reason`

### CU-U-NO-07: Productores toleran notificaciones como efecto secundario opcional

- Tipo de prueba: Unitaria
- Dominio: Notifications / Internships
- Contexto: Las notificaciones no deben bloquear el flujo principal si el servicio no está inyectado.
- Objetivo: Confirmar que el productor continúa normalmente cuando `notification_service=None`.
- Escenario: El servicio de prácticas aprueba una práctica sin servicio de notificaciones configurado.
- Variantes cubiertas:
  - Aprobación retorna resultado sin intentar notificar.
- Resultado esperado: El flujo principal no depende de mensajería.
- Valor de negocio: Evita que un problema de mensajería detenga acciones administrativas críticas.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestNotificationFromExternalService::test_internship_service_without_notification_service_skips_gracefully`

### CU-U-NO-08: Reintento respeta configuración y estados elegibles

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: El reintento manual solo tiene sentido con mailer configurado y notificación reintentable.
- Objetivo: Validar que el servicio no reintenta notificaciones inexistentes, ya enviadas o sin mailer.
- Escenario: Se solicita reintento en condiciones no elegibles.
- Variantes cubiertas:
  - Mailer no configurado.
  - Notificación inexistente.
  - Notificación ya enviada.
- Resultado esperado: El servicio retorna `None` y no ejecuta reintento.
- Valor de negocio: Evita operaciones inconsistentes.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_returns_none_when_mailer_not_configured`
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_returns_none_for_nonexistent_notification`
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_returns_none_for_already_sent_notification`

### CU-U-NO-09: Reintento actualiza estado según resultado SMTP

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: Las notificaciones `failed` o `pending` pueden reintentarse manualmente.
- Objetivo: Proteger la lógica operacional de recuperación de notificaciones.
- Escenario: Se reintentan notificaciones elegibles con SMTP exitoso o fallido.
- Variantes cubiertas:
  - Reintento exitoso desde `failed`.
  - Reintento exitoso desde `pending`.
  - Reintento con fallo SMTP conserva estado `failed`.
- Resultado esperado: Éxito marca `sent`; fallo marca o mantiene `failed`.
- Valor de negocio: Permite recuperación manual confiable sin perder trazabilidad.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_successful_for_failed_notification`
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_successful_for_pending_notification`
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_smtp_failure_keeps_notification_failed`
