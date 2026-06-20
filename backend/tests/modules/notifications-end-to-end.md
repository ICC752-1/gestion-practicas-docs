# Casos de Prueba - Notifications

## Alcance

Estos casos documentan las pruebas de valor del módulo `notifications`. El foco está en persistencia de notificaciones, modos de despacho `simulated` y `real`, reintentos operativos, permisos de consulta, endpoint legacy de envío SMTP directo, contratos de eventos y seguridad del contenido HTML generado.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos verificables que protegen reglas operativas o contratos relevantes del sistema.

## End-to-End

### CU-E2E-NO-01: Usuario consulta notificaciones generadas por una práctica

- Tipo de prueba: End-to-end
- Dominio: Notifications / Internships
- Contexto: Un usuario debería poder ver desde la API las notificaciones generadas por acciones reales sobre su práctica.
- Objetivo: Validar autenticación, acción de práctica, persistencia de notificación y consulta de bandeja en conjunto.
- Escenario: Estudiante crea una práctica; un rol administrativo aprueba o rechaza; el estudiante consulta `/notifications` y el detalle correspondiente.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: La notificación aparece en la bandeja del destinatario y solo ese usuario puede consultar su detalle.
- Valor de negocio: Da confianza sobre el flujo visible de comunicación al estudiante.
- Pruebas automatizadas:
  - Pendiente de implementación.

### CU-E2E-NO-02: Flujo documental genera notificaciones visibles para participantes

- Tipo de prueba: End-to-end
- Dominio: Notifications / Documents
- Contexto: La revisión documental es una interacción crítica entre estudiante y roles administrativos.
- Objetivo: Validar que carga, observación y aprobación documental producen notificaciones consultables por los usuarios correspondientes.
- Escenario: Estudiante sube un documento; rol documental lo observa; estudiante consulta notificación; luego el documento se aprueba y se genera nueva notificación.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: Cada evento documental queda disponible en la bandeja correcta y con payload coherente.
- Valor de negocio: Protege comunicación documental y reduce incertidumbre durante el trámite.
- Pruebas automatizadas:
  - Pendiente de implementación.

### CU-E2E-NO-03: Reintento operativo de notificación fallida

- Tipo de prueba: End-to-end
- Dominio: Notifications
- Contexto: En operación real, una notificación puede fallar por SMTP y requerir reintento manual por un rol autorizado.
- Objetivo: Validar el flujo completo de recuperación de una notificación fallida en un entorno SMTP controlado.
- Escenario: Se genera una notificación `failed`; un rol autorizado ejecuta `/notifications/{id}/retry`; el sistema reenvía y actualiza estado.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: El reintento actualiza la notificación a `sent` si SMTP responde correctamente o conserva `failed` si vuelve a fallar.
- Valor de negocio: Da confianza sobre una operación administrativa de recuperación ante fallas de correo.
- Pruebas automatizadas:
  - Pendiente de implementación.
