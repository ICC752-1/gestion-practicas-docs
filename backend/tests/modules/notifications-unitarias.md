# Casos de Prueba - Notifications

## Alcance

Estos casos documentan las pruebas de valor del módulo `notifications`. El foco está en persistencia de notificaciones, modos de despacho `simulated` y `real`, reintentos operativos, permisos de consulta, endpoint legacy de envío SMTP directo, contratos de eventos y seguridad del contenido HTML generado.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos verificables que protegen reglas operativas o contratos relevantes del sistema.

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
- Contexto: En modo `real`, una notificación persistente debe intentar envío SMTP y actualizar su estado según el resultado.
- Objetivo: Validar el ciclo `pending` -> `sent` o `failed` para notificaciones persistentes.
- Escenario: El servicio crea una notificación en modo real y el transporte SMTP responde con éxito o error.
- Variantes cubiertas:
  - SMTP exitoso invoca el mailer y actualiza estado de entrega.
  - Error SMTP actualiza la notificación como `failed`.
- Resultado esperado: El estado final refleja el resultado real del intento de envío.
- Valor de negocio: Permite operar notificaciones reales con trazabilidad de entrega y fallos.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestCreateRealNotification::test_real_mode_persists_and_sends_smtp`
  - `tests/modules/notifications/test_notification_service.py::TestCreateRealNotification::test_real_mode_smtp_failure_marks_as_failed`

### CU-U-NO-03: Evitar envío real cuando la configuración SMTP no está lista

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: El modo `real` requiere credenciales SMTP válidas. Si se mantienen credenciales por defecto, el sistema no debe intentar correos reales.
- Objetivo: Verificar que una configuración incompleta cae a persistencia simulada y que el envío directo se rechaza sin mailer.
- Escenario: El servicio se inicializa en modo real con credenciales de prueba o intenta enviar correo directo sin mailer configurado.
- Variantes cubiertas:
  - Modo `real` con credenciales por defecto no configura mailer y persiste como `simulated`.
  - `send_email` falla con `RuntimeError` si no existe mailer SMTP.
- Resultado esperado: No se intenta SMTP real y el error es explícito cuando se solicita envío directo.
- Valor de negocio: Reduce riesgo de configuraciones ambiguas y evita asumir entregas reales que nunca ocurrieron.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestCreateRealNotification::test_real_mode_falls_back_to_simulated_with_default_credentials`
  - `tests/modules/notifications/test_notification_service.py::TestInvalidRecipient::test_send_email_raises_runtime_error_in_simulated_mode`

### CU-U-NO-04: Rechazar envío SMTP persistente sin destinatario de correo

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: Una notificación persistente puede tener destinatario interno, pero el envío SMTP necesita un correo válido.
- Objetivo: Evitar intentos de envío sin destinatario de email.
- Escenario: Se intenta despachar por SMTP una notificación que no tiene `recipient_email`.
- Variantes cubiertas:
  - Notificación persistente sin correo de destinatario.
- Resultado esperado: El envío SMTP falla con error de destinatario inválido.
- Valor de negocio: Evita errores silenciosos y facilita diagnóstico operativo de notificaciones no entregables.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestInvalidRecipient::test_send_smtp_raises_on_no_recipient_email`

### CU-U-NO-05: Helpers de eventos conservan contrato de ruteo y payload

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: Los módulos productores construyen notificaciones mediante helpers. El frontend, auditoría y diagnóstico dependen de `event_type`, destinatario y `payload` estables.
- Objetivo: Validar que los helpers de eventos críticos construyen notificaciones con datos mínimos esperados.
- Escenario: Se generan notificaciones para aprobación, rechazo, preparación de expediente y cambio de requisito.
- Variantes cubiertas:
  - Aprobación de práctica conserva `event_type`, destinatario y `internship_id`.
  - Rechazo de práctica conserva motivo en payload.
  - Preparación local del expediente conserva motivo en payload.
  - Cambio de requisito conserva requisito, estado nuevo y estado anterior.
- Resultado esperado: Las notificaciones generadas mantienen contratos de ruteo y metadata funcional.
- Valor de negocio: Evita romper consumidores internos que interpretan notificaciones por tipo y payload.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestPayloadStorage::test_internship_event_helpers_keep_routing_and_payload_contract`
  - `tests/modules/notifications/test_notification_service.py::TestPayloadStorage::test_requirement_status_changed_notification_keeps_payload_contract`

### CU-U-NO-06: Contenido HTML omite datos vacíos y escapa valores dinámicos

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: Las notificaciones se renderizan como HTML y pueden incluir datos ingresados por usuarios o administradores.
- Objetivo: Evitar contenido engañoso por campos vacíos y reducir riesgo de inyección HTML en correos.
- Escenario: Se construyen notificaciones con motivo ausente y con valores dinámicos que contienen HTML.
- Variantes cubiertas:
  - Rechazo sin motivo no muestra fila de motivo.
  - Organización y motivo con tags HTML se escapan antes de insertarse en contenido.
- Resultado esperado: El contenido no incluye filas vacías ni HTML dinámico sin escapar.
- Valor de negocio: Mejora seguridad y calidad de correos transaccionales enviados a usuarios.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestPayloadStorage::test_rejected_notification_without_reason`
  - `tests/modules/notifications/test_notification_service.py::TestPayloadStorage::test_event_helpers_escape_dynamic_html_values`

### CU-U-NO-07: Productores toleran notificaciones como efecto secundario opcional

- Tipo de prueba: Unitaria
- Dominio: Notifications / Internships
- Contexto: Las notificaciones no deben bloquear el flujo principal de una práctica si el servicio no está inyectado.
- Objetivo: Confirmar que el productor puede emitir eventos cuando existe servicio y continuar normalmente cuando no existe.
- Escenario: El servicio de prácticas aprueba una práctica con y sin `NotificationService` configurado.
- Variantes cubiertas:
  - Aprobación emite notificación cuando hay servicio configurado.
  - Aprobación continúa si `notification_service=None`.
- Resultado esperado: La notificación se crea cuando corresponde, pero el flujo principal no depende de ella.
- Valor de negocio: Evita que un problema de mensajería detenga acciones administrativas críticas.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestNotificationFromExternalService::test_internship_service_dispatches_notification_on_approve`
  - `tests/modules/notifications/test_notification_service.py::TestNotificationFromExternalService::test_internship_service_without_notification_service_skips_gracefully`

### CU-U-NO-08: Reintento respeta configuración y estados elegibles

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: El reintento manual solo tiene sentido si existe mailer SMTP configurado y la notificación está en estado reintentable.
- Objetivo: Validar que el servicio no reintenta notificaciones inexistentes, no elegibles o sin mailer.
- Escenario: Se solicita reintento en condiciones donde no corresponde ejecutar SMTP.
- Variantes cubiertas:
  - Mailer no configurado.
  - Notificación inexistente.
  - Notificación ya enviada.
- Resultado esperado: El servicio retorna `None` y no ejecuta reintento.
- Valor de negocio: Evita operaciones inconsistentes y mantiene clara la semántica de reintento operativo.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_returns_none_when_mailer_not_configured`
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_returns_none_for_nonexistent_notification`
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_returns_none_for_already_sent_notification`

### CU-U-NO-09: Reintento actualiza estado según resultado SMTP

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: Las notificaciones `failed` o `pending` pueden reintentarse manualmente. El estado final debe reflejar el resultado del nuevo intento.
- Objetivo: Proteger la lógica operacional de recuperación de notificaciones fallidas o pendientes.
- Escenario: Se reintentan notificaciones elegibles con SMTP exitoso o fallido.
- Variantes cubiertas:
  - Reintento exitoso desde `failed`.
  - Reintento exitoso desde `pending`.
  - Reintento con fallo SMTP conserva estado `failed`.
- Resultado esperado: Éxito marca `sent`; fallo marca o mantiene `failed`.
- Valor de negocio: Permite recuperación manual confiable sin perder trazabilidad de fallos.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_successful_for_failed_notification`
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_successful_for_pending_notification`
  - `tests/modules/notifications/test_notification_service.py::TestRetrySend::test_retry_smtp_failure_keeps_notification_failed`

### CU-U-NO-10: Contrato ORM mantiene columnas y enums críticos

- Tipo de prueba: Unitaria
- Dominio: Notifications
- Contexto: Los modelos ORM deben mantenerse alineados manualmente con el esquema SQL inicial definido en `init.sql`.
- Objetivo: Detectar cambios accidentales en tabla, columnas o enums usados por el módulo.
- Escenario: Se inspeccionan el modelo `Notification` y enums de evento y estado.
- Variantes cubiertas:
  - Tabla y columnas críticas existen.
  - Enum de eventos conserva valores funcionales.
  - Enum de estados conserva valores de entrega.
- Resultado esperado: El contrato ORM mantiene nombres y valores críticos.
- Valor de negocio: Reduce riesgo de desalineación entre modelos, SQL y consumidores del módulo.
- Pruebas automatizadas:
  - `tests/modules/notifications/test_notification_model.py::test_notification_model_matches_database_contract`
  - `tests/modules/notifications/test_notification_model.py::test_notification_event_type_enum_matches_business_contract`
  - `tests/modules/notifications/test_notification_model.py::test_notification_status_enum_matches_delivery_contract`
