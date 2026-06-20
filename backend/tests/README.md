# Estrategia de Pruebas Backend

## Objetivo

Este directorio documenta los casos de prueba de backend que aportan valor real al sistema. El foco no es aumentar la cantidad de pruebas, sino proteger reglas de negocio, permisos, estados, trazabilidad, integridad documental y flujos críticos del proceso de prácticas.

Una prueba se considera valiosa cuando ayuda a detectar una regresión que afectaría al estudiante, a los roles administrativos, a la consistencia del trámite o a la seguridad funcional del sistema.

## Tipos de prueba

| Tipo | Uso en este backend |
| --- | --- |
| Unitaria | Valida una regla, servicio, schema o decisión de negocio en aislamiento, usando dobles de prueba cuando corresponde. |
| Integración | Valida que varias partes internas colaboren correctamente, por ejemplo endpoint + dependencias + servicio, o servicio + repositorio. |
| End-to-end | Valida un flujo completo de negocio atravesando las capas principales del backend. Deben ser pocas y centradas en flujos críticos. |

## Criterios de valor

Una prueba debe existir si cumple al menos uno de estos criterios:

- Protege una regla de negocio crítica.
- Evita aprobar, rechazar, derivar o exportar datos en un estado incorrecto.
- Verifica permisos relevantes por rol.
- Protege trazabilidad administrativa.
- Evita inconsistencias en requisitos académicos o institucionales.
- Valida un contrato documental o de sesión importante.
- Cubre una regresión probable o costosa de detectar manualmente.

Una prueba debería evitarse, consolidarse o reemplazarse si solo verifica detalles internos sin comportamiento observable, duplica otra prueba equivalente o existe únicamente para aumentar cobertura numérica.

## Glosario

| Término | Significado |
| --- | --- |
| Práctica estival | Práctica realizada en periodo `Verano` o `Invierno`, fuera del periodo académico regular. Para aprobación final requiere seguro escolar o excepción administrativa. |
| Seguro escolar | Requisito institucional que indica que el estudiante tiene cobertura registrada para la práctica. |
| Excepción administrativa | Autorización trazable que permite avanzar una práctica aunque falte una regla exceptuable. No modifica el requisito original. |
| Estado terminal | Estado final donde una práctica no debería modificarse por acciones normales. Actualmente: `Aprobada`, `Rechazada` y `Reprobada`. |
| Paquete documental DIRAE | Resumen de documentos requeridos y aprobados para evaluar si una práctica puede exportarse o tramitarse hacia DIRAE. |
| Exportable | Condición que indica que la práctica está aprobada y tiene todos los documentos requeridos aprobados para ser incluida en la exportación DIRAE. |
| Inducción | Requisito obligatorio para tramitar la aprobación de `Práctica de Estudio I`. |
| Secuencialidad | Regla académica que exige aprobar una práctica previa antes de aprobar una posterior, por ejemplo Práctica I antes de Práctica II. |

## Resumen de cobertura documentada

| Módulo | Casos unitarios | Tests unitarios | Casos integración | Tests integración | Casos E2E | Tests E2E | E2E pendientes |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Admin | 13 | 20 | 5 | 6 | 3 | 0 | 3 |
| Auth | 18 | 58 | 7 | 15 | 3 | 0 | 3 |
| Documents | 13 | 47 | 4 | 12 | 3 | 0 | 3 |
| Internships | 24 | 96 | 3 | 6 | 2 | 0 | 2 |
| Notifications | 10 | 22 | 6 | 12 | 3 | 0 | 3 |
| **Total** | **78** | **243** | **25** | **51** | **14** | **0** | **14** |

En total hay 117 casos documentados y 294 referencias a tests automatizados. Los 14 casos end-to-end están documentados como pendientes de implementación.

## Índice de casos de prueba

Este índice resume cada caso con su nombre corto, tipo, módulo, cantidad de tests asociados y archivo detallado.

| ID | Tipo | Módulo | Caso | Tests | Archivo |
| --- | --- | --- | --- | ---: | --- |
| CU-U-AD-01 | Unitaria | Admin | Resumen administrativo conserva totales y estados | 2 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-02 | Unitaria | Admin | Listado administrativo de estudiantes conserva datos relevantes | 1 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-03 | Unitaria | Admin | Listado administrativo de prácticas mapea estudiante y estado | 2 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-04 | Unitaria | Admin | Filtros normalizados del dashboard agrupan estados funcionales | 1 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-05 | Unitaria | Admin | Detalle administrativo de práctica existente o inexistente | 2 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-06 | Unitaria | Admin | Listado de requisitos académicos del estudiante | 1 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-07 | Unitaria | Admin | Transiciones válidas de requisitos académicos | 1 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-08 | Unitaria | Admin | Transiciones inválidas de requisitos académicos | 1 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-09 | Unitaria | Admin | Actualización de requisito académico registra trazabilidad | 3 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-10 | Unitaria | Admin | Actualización de requisito académico emite notificación sin bloquear flujo | 2 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-11 | Unitaria | Admin | Listado de requisitos institucionales del estudiante | 1 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-12 | Unitaria | Admin | Seguro escolar se crea o actualiza correctamente | 2 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-U-AD-13 | Unitaria | Admin | Seguro escolar rechaza usuarios que no son estudiantes | 1 | [`modules/admin-unitarias.md`](modules/admin-unitarias.md) |
| CU-I-AD-01 | Integración | Admin | Roles autorizados para seguro escolar | 2 | [`modules/admin-integracion.md`](modules/admin-integracion.md) |
| CU-I-AD-02 | Integración | Admin | Detalle de práctica inexistente se traduce a 404 | 1 | [`modules/admin-integracion.md`](modules/admin-integracion.md) |
| CU-I-AD-03 | Integración | Admin | Transición inválida se traduce a 400 | 1 | [`modules/admin-integracion.md`](modules/admin-integracion.md) |
| CU-I-AD-04 | Integración | Admin | Requisito académico inexistente se traduce a 404 | 1 | [`modules/admin-integracion.md`](modules/admin-integracion.md) |
| CU-I-AD-05 | Integración | Admin | Seguro escolar de usuario no estudiante se traduce a 404 | 1 | [`modules/admin-integracion.md`](modules/admin-integracion.md) |
| CU-E2E-AD-01 | End-to-end | Admin | Coordinador consulta dashboard y detalle de práctica | Pendiente | [`modules/admin-end-to-end.md`](modules/admin-end-to-end.md) |
| CU-E2E-AD-02 | End-to-end | Admin | Coordinador actualiza requisito académico y estudiante recibe notificación | Pendiente | [`modules/admin-end-to-end.md`](modules/admin-end-to-end.md) |
| CU-E2E-AD-03 | End-to-end | Admin | Director registra seguro escolar y práctica estival puede aprobarse | Pendiente | [`modules/admin-end-to-end.md`](modules/admin-end-to-end.md) |
| CU-U-AU-01 | Unitaria | Auth | Credenciales locales válidas o incorrectas | 4 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-02 | Unitaria | Auth | Login emite tokens y persiste refresh token como hash | 1 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-03 | Unitaria | Auth | Creación de sesión para usuario emite claims y refresh persistido | 1 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-04 | Unitaria | Auth | Refresh token válido rota sesión y revoca token anterior | 1 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-05 | Unitaria | Auth | Refresh rechaza tokens inválidos o no vigentes | 3 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-06 | Unitaria | Auth | Refresh rechaza usuario inexistente o inactivo | 2 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-07 | Unitaria | Auth | Logout revoca refresh token válido y tolera ausencia de token | 2 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-08 | Unitaria | Auth | Logout rechaza refresh inválido o de otro usuario | 3 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-09 | Unitaria | Auth | TokenService emite y valida claims críticos | 11 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-10 | Unitaria | Auth | RefreshTokenRepository persiste, revoca y valida vigencia | 6 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-11 | Unitaria | Auth | PasswordService hashea y valida contraseñas | 3 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-12 | Unitaria | Auth | RUT y teléfonos se normalizan y validan | 9 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-13 | Unitaria | Auth | Dependencia de roles permite o rechaza según autorización | 2 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-14 | Unitaria | Auth | Google OAuth construye URL y valida callback | 3 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-15 | Unitaria | Auth | Google OAuth emite sesión para usuario existente | 1 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-16 | Unitaria | Auth | Google OAuth crea estudiante para dominio permitido | 1 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-17 | Unitaria | Auth | Google OAuth rechaza dominio o código inválido | 2 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-U-AU-18 | Unitaria | Auth | RoleService actualiza, asigna y remueve roles | 3 | [`modules/auth-unitarias.md`](modules/auth-unitarias.md) |
| CU-I-AU-01 | Integración | Auth | `get_current_user` rechaza refresh token como Bearer | 1 | [`modules/auth-integracion.md`](modules/auth-integracion.md) |
| CU-I-AU-02 | Integración | Auth | `get_current_user` valida access token y usuario vigente | 5 | [`modules/auth-integracion.md`](modules/auth-integracion.md) |
| CU-I-AU-03 | Integración | Auth | Controller de login traduce credenciales inválidas | 2 | [`modules/auth-integracion.md`](modules/auth-integracion.md) |
| CU-I-AU-04 | Integración | Auth | Controller de refresh traduce refresh inválido | 1 | [`modules/auth-integracion.md`](modules/auth-integracion.md) |
| CU-I-AU-05 | Integración | Auth | Controller de logout traduce refresh inválido | 1 | [`modules/auth-integracion.md`](modules/auth-integracion.md) |
| CU-I-AU-06 | Integración | Auth | `/auth/me` no expone credenciales ni campos sensibles | 1 | [`modules/auth-integracion.md`](modules/auth-integracion.md) |
| CU-I-AU-07 | Integración | Auth | Google controller maneja cookie, state y redirect | 4 | [`modules/auth-integracion.md`](modules/auth-integracion.md) |
| CU-E2E-AU-01 | End-to-end | Auth | Login local, consulta de usuario actual y logout | Pendiente | [`modules/auth-end-to-end.md`](modules/auth-end-to-end.md) |
| CU-E2E-AU-02 | End-to-end | Auth | Login y rotación de refresh token | Pendiente | [`modules/auth-end-to-end.md`](modules/auth-end-to-end.md) |
| CU-E2E-AU-03 | End-to-end | Auth | Google OAuth completo en entorno controlado | Pendiente | [`modules/auth-end-to-end.md`](modules/auth-end-to-end.md) |
| CU-U-DO-01 | Unitaria | Documents | Carga documental válida persiste metadata y archivo privado | 2 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-02 | Unitaria | Documents | Carga documental rechaza archivo inválido | 3 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-03 | Unitaria | Documents | Carga documental valida práctica, tipo documental y propietario | 4 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-04 | Unitaria | Documents | Carga documental limpia archivo si falla persistencia | 1 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-05 | Unitaria | Documents | Listado y descarga respetan permisos documentales | 3 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-06 | Unitaria | Documents | Descarga rechaza archivos inexistentes, eliminados o paths inseguros | 3 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-07 | Unitaria | Documents | Revisión documental respeta roles, estados y comentario obligatorio | 5 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-08 | Unitaria | Documents | Eliminación lógica respeta propietario, roles y estado aprobado | 4 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-09 | Unitaria | Documents | Paquete DIRAE exportable requiere práctica aprobada y documentos requeridos aprobados | 4 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-10 | Unitaria | Documents | Paquete DIRAE selecciona documentos vigentes y últimos aprobados | 5 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-11 | Unitaria | Documents | Paquete DIRAE construye datos del estudiante y controla acceso | 4 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-12 | Unitaria | Documents | Exportación DIRAE genera CSV y auditoría estructurada | 6 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-U-DO-13 | Unitaria | Documents | Contrato ORM documental mantiene columnas y enums críticos | 3 | [`modules/documents-unitarias.md`](modules/documents-unitarias.md) |
| CU-I-DO-01 | Integración | Documents | Descarga HTTP exige autenticación y respeta propietario | 3 | [`modules/documents-integracion.md`](modules/documents-integracion.md) |
| CU-I-DO-02 | Integración | Documents | Controller delega operaciones documentales al service y preserva contrato | 6 | [`modules/documents-integracion.md`](modules/documents-integracion.md) |
| CU-I-DO-03 | Integración | Documents | Roles documentales restringen revisión y exportación | 2 | [`modules/documents-integracion.md`](modules/documents-integracion.md) |
| CU-I-DO-04 | Integración | Documents | Controller propaga errores de servicio | 1 | [`modules/documents-integracion.md`](modules/documents-integracion.md) |
| CU-E2E-DO-01 | End-to-end | Documents | Estudiante carga documento y rol documental lo aprueba | Pendiente | [`modules/documents-end-to-end.md`](modules/documents-end-to-end.md) |
| CU-E2E-DO-02 | End-to-end | Documents | Documento observado se corrige con nueva versión aprobada | Pendiente | [`modules/documents-end-to-end.md`](modules/documents-end-to-end.md) |
| CU-E2E-DO-03 | End-to-end | Documents | Exportación DIRAE de práctica aprobada con documentos completos | Pendiente | [`modules/documents-end-to-end.md`](modules/documents-end-to-end.md) |
| CU-U-IN-01 | Unitaria | Internships | Bloquear aprobación final de práctica estival sin seguro ni excepción | 2 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-02 | Unitaria | Internships | Permitir aprobación estival si el seguro fue regularizado antes de aprobar | 1 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-03 | Unitaria | Internships | Permitir avance a revisión de práctica estival sin seguro | 1 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-04 | Unitaria | Internships | Bloquear Práctica I si el estudiante no aprobó la inducción | 2 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-05 | Unitaria | Internships | Permitir Práctica I con inducción aprobada | 2 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-06 | Unitaria | Internships | Bloquear Práctica II sin Práctica I aprobada | 3 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-07 | Unitaria | Internships | Permitir Práctica II con Práctica I aprobada o excepción | 5 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-08 | Unitaria | Internships | Validar secuencialidad de Tesis | 3 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-09 | Unitaria | Internships | Validar regla de Práctica Controlada y ramo paralelo | 2 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-10 | Unitaria | Internships | Validar aprobación normal versus aprobación directa | 5 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-11 | Unitaria | Internships | Impedir acciones sobre estados terminales | 6 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-12 | Unitaria | Internships | Rechazo y derivación deben exigir comentario | 4 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-13 | Unitaria | Internships | Validar excepciones administrativas | 5 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-14 | Unitaria | Internships | Elegibilidad de registro informa bloqueos sin impedir creación | 8 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-15 | Unitaria | Internships | Crear Práctica II no exige Práctica I aprobada | 2 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-16 | Unitaria | Internships | Aprobación sincroniza requisito académico | 2 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-17 | Unitaria | Internships | Creación calcula seguro escolar desde requisito institucional | 4 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-18 | Unitaria | Internships | Dashboard normaliza estados y calcula estadísticas | 3 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-19 | Unitaria | Internships | Edición administrativa exige motivo, rol y campos válidos | 6 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-20 | Unitaria | Internships | Anulación lógica conserva trazabilidad | 3 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-21 | Unitaria | Internships | Contrato de creación de práctica valida payload | 11 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-22 | Unitaria | Internships | Contrato de excepción valida regla y motivo | 7 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-23 | Unitaria | Internships | Permisos de lectura permiten propietario o rol privilegiado | 5 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-U-IN-24 | Unitaria | Internships | Contrato ORM mantiene columnas y enums críticos | 4 | [`modules/internships-unitarias.md`](modules/internships-unitarias.md) |
| CU-I-IN-01 | Integración | Internships | Secretaría puede derivar, pero no aprobar ni rechazar | 1 | [`modules/internships-integracion.md`](modules/internships-integracion.md) |
| CU-I-IN-02 | Integración | Internships | Tracking permite propietario y roles privilegiados | 4 | [`modules/internships-integracion.md`](modules/internships-integracion.md) |
| CU-I-IN-03 | Integración | Internships | Dashboard rechaza rol estudiante | 1 | [`modules/internships-integracion.md`](modules/internships-integracion.md) |
| CU-E2E-IN-01 | End-to-end | Internships | Flujo completo de Práctica I aprobada | Pendiente | [`modules/internships-end-to-end.md`](modules/internships-end-to-end.md) |
| CU-E2E-IN-02 | End-to-end | Internships | Flujo completo de práctica estival bloqueada y luego aprobada | Pendiente | [`modules/internships-end-to-end.md`](modules/internships-end-to-end.md) |
| CU-U-NO-01 | Unitaria | Notifications | Persistir notificaciones simuladas sin invocar SMTP | 2 | [`modules/notifications-unitarias.md`](modules/notifications-unitarias.md) |
| CU-U-NO-02 | Unitaria | Notifications | Enviar notificaciones reales y registrar resultado de entrega | 2 | [`modules/notifications-unitarias.md`](modules/notifications-unitarias.md) |
| CU-U-NO-03 | Unitaria | Notifications | Evitar envío real cuando la configuración SMTP no está lista | 2 | [`modules/notifications-unitarias.md`](modules/notifications-unitarias.md) |
| CU-U-NO-04 | Unitaria | Notifications | Rechazar envío SMTP persistente sin destinatario de correo | 1 | [`modules/notifications-unitarias.md`](modules/notifications-unitarias.md) |
| CU-U-NO-05 | Unitaria | Notifications | Helpers de eventos conservan contrato de ruteo y payload | 2 | [`modules/notifications-unitarias.md`](modules/notifications-unitarias.md) |
| CU-U-NO-06 | Unitaria | Notifications | Contenido HTML omite datos vacíos y escapa valores dinámicos | 2 | [`modules/notifications-unitarias.md`](modules/notifications-unitarias.md) |
| CU-U-NO-07 | Unitaria | Notifications | Productores toleran notificaciones como efecto secundario opcional | 2 | [`modules/notifications-unitarias.md`](modules/notifications-unitarias.md) |
| CU-U-NO-08 | Unitaria | Notifications | Reintento respeta configuración y estados elegibles | 3 | [`modules/notifications-unitarias.md`](modules/notifications-unitarias.md) |
| CU-U-NO-09 | Unitaria | Notifications | Reintento actualiza estado según resultado SMTP | 3 | [`modules/notifications-unitarias.md`](modules/notifications-unitarias.md) |
| CU-U-NO-10 | Unitaria | Notifications | Contrato ORM mantiene columnas y enums críticos | 3 | [`modules/notifications-unitarias.md`](modules/notifications-unitarias.md) |
| CU-I-NO-01 | Integración | Notifications | Usuario autenticado lista sus notificaciones | 1 | [`modules/notifications-integracion.md`](modules/notifications-integracion.md) |
| CU-I-NO-02 | Integración | Notifications | Usuario solo consulta detalle de notificaciones propias | 3 | [`modules/notifications-integracion.md`](modules/notifications-integracion.md) |
| CU-I-NO-03 | Integración | Notifications | Controller de reintento traduce resultado operacional | 3 | [`modules/notifications-integracion.md`](modules/notifications-integracion.md) |
| CU-I-NO-04 | Integración | Notifications | Endpoint legacy de envío directo traduce errores SMTP | 3 | [`modules/notifications-integracion.md`](modules/notifications-integracion.md) |
| CU-I-NO-05 | Integración | Notifications | Eventos de ciclo de práctica se persisten en modo simulado | 1 | [`modules/notifications-integracion.md`](modules/notifications-integracion.md) |
| CU-I-NO-06 | Integración | Notifications | Eventos documentales se persisten en modo simulado | 1 | [`modules/notifications-integracion.md`](modules/notifications-integracion.md) |
| CU-E2E-NO-01 | End-to-end | Notifications | Usuario consulta notificaciones generadas por una práctica | Pendiente | [`modules/notifications-end-to-end.md`](modules/notifications-end-to-end.md) |
| CU-E2E-NO-02 | End-to-end | Notifications | Flujo documental genera notificaciones visibles para participantes | Pendiente | [`modules/notifications-end-to-end.md`](modules/notifications-end-to-end.md) |
| CU-E2E-NO-03 | End-to-end | Notifications | Reintento operativo de notificación fallida | Pendiente | [`modules/notifications-end-to-end.md`](modules/notifications-end-to-end.md) |

## Archivos

| Archivo | Propósito |
| --- | --- |
| `modules/admin-unitarias.md` | Casos unitarios del módulo `admin`. |
| `modules/admin-integracion.md` | Casos de integración del módulo `admin`. |
| `modules/admin-end-to-end.md` | Flujos end-to-end documentados del módulo `admin`. |
| `modules/auth-unitarias.md` | Casos unitarios del módulo `auth`. |
| `modules/auth-integracion.md` | Casos de integración del módulo `auth`. |
| `modules/auth-end-to-end.md` | Flujos end-to-end documentados del módulo `auth`. |
| `modules/documents-unitarias.md` | Casos unitarios del módulo `documents`. |
| `modules/documents-integracion.md` | Casos de integración del módulo `documents`. |
| `modules/documents-end-to-end.md` | Flujos end-to-end documentados del módulo `documents`. |
| `modules/internships-unitarias.md` | Casos unitarios del módulo `internships`. |
| `modules/internships-integracion.md` | Casos de integración del módulo `internships`. |
| `modules/internships-end-to-end.md` | Flujos end-to-end documentados del módulo `internships`. |
| `modules/notifications-unitarias.md` | Casos unitarios del módulo `notifications`. |
| `modules/notifications-integracion.md` | Casos de integración del módulo `notifications`. |
| `modules/notifications-end-to-end.md` | Flujos end-to-end documentados del módulo `notifications`. |
