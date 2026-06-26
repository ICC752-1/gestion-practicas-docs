# Casos de Prueba - Notifications

## Alcance

Estos casos documentan las pruebas end-to-end de mayor valor del módulo `notifications`. El foco está en comprobar que una acción real de negocio genera una notificación persistida y visible para su destinatario.

## End-to-End

### CU-E2E-NO-01: Acción real genera notificación visible para el estudiante

- **Tipo de prueba:** End-to-end
- **Dominio:** Notifications / Internships
- **Contexto:** Un usuario debería poder ver desde la API las notificaciones generadas por acciones reales sobre su práctica.
- **Objetivo:** Validar autenticación, acción de práctica, persistencia de notificación, consulta de bandeja, detalle y aislamiento por destinatario.
- **Escenario:** Estudiante crea una práctica; Dirección la aprueba; el estudiante consulta `/notifications` y el detalle correspondiente; otro estudiante intenta consultar ese detalle y recibe rechazo.
- **Resultado esperado:** La notificación aparece en la bandeja del destinatario, contiene payload de la práctica y solo ese usuario puede consultar su detalle.
- **Valor de negocio:** Da confianza sobre el flujo visible de comunicación al estudiante.
- **Pruebas automatizadas:**
  - `tests/e2e/test_notifications_e2e.py::test_real_internship_action_generates_notification_visible_to_student`
