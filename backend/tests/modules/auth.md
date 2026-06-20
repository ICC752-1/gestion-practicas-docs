# Casos de Prueba - Auth

## Alcance

Estos casos documentan las pruebas de valor del módulo `auth`. El foco está en autenticación local, emisión y rotación de tokens, revocación de refresh tokens, resolución del usuario autenticado, autorización por roles, normalización de datos de identidad, Google OAuth y administración básica de roles.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos de seguridad funcional o contratos transversales del backend.

## Unitarias

### CU-U-AU-01: Credenciales locales válidas o incorrectas

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: El login local valida email y password antes de emitir una sesión.
- Objetivo: Confirmar que solo credenciales válidas autentican al usuario.
- Escenario: Se autentica un usuario con credenciales válidas, usuario inexistente y password incorrecta.
- Variantes cubiertas:
  - Usuario existente con password válido.
  - Usuario inexistente.
  - Password inválido.
  - Login de alto nivel lanza error ante credenciales inválidas.
- Resultado esperado: Solo el caso válido retorna usuario; los casos inválidos no emiten sesión.
- Valor de negocio: Protege el punto de entrada principal al sistema.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_service.py::test_authenticate_user_returns_user_with_valid_credentials`
  - `tests/modules/auth/test_auth_service.py::test_authenticate_user_returns_none_when_user_missing`
  - `tests/modules/auth/test_auth_service.py::test_authenticate_user_returns_none_for_invalid_password`
  - `tests/modules/auth/test_auth_service.py::test_login_raises_for_invalid_credentials`

### CU-U-AU-02: Login emite tokens y persiste refresh token como hash

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Una sesión válida debe entregar access token, refresh token y persistir solo el hash del refresh token.
- Objetivo: Validar emisión de tokens y almacenamiento seguro del refresh token.
- Escenario: Un usuario inicia sesión con credenciales válidas.
- Variantes cubiertas:
  - Access token incluye subject, email y roles.
  - Refresh token usa JTI generado.
  - Refresh token se persiste con hash, no en texto plano.
  - El token tiene expiración persistida.
- Resultado esperado: Se retorna `TokenResponse` y se crea un refresh token revocable persistido como hash.
- Valor de negocio: Permite sesiones renovables sin almacenar secretos en texto claro.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_service.py::test_login_returns_tokens_for_valid_credentials`

### CU-U-AU-03: Creación de sesión para usuario emite claims y refresh persistido

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Google OAuth y otros flujos internos reutilizan la creación de sesión sin pasar por password local.
- Objetivo: Confirmar que `create_session_for_user` emite tokens internos equivalentes al login local.
- Escenario: Se crea una sesión para un usuario ya autenticado por otro mecanismo.
- Variantes cubiertas:
  - Access token conserva subject, email y roles.
  - Refresh token conserva subject y JTI.
  - Refresh token se persiste con hash.
- Resultado esperado: El usuario recibe tokens internos y queda un refresh token revocable en persistencia.
- Valor de negocio: Mantiene un contrato único de sesión para login local y OAuth.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_service.py::test_create_session_for_user_returns_tokens_and_persists_refresh_token`

### CU-U-AU-04: Refresh token válido rota sesión y revoca token anterior

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: El backend usa refresh tokens rotativos para reducir riesgo por reutilización de tokens antiguos.
- Objetivo: Validar que un refresh válido genera nueva sesión y revoca el refresh anterior.
- Escenario: Se renueva sesión con un refresh token válido, vigente y persistido.
- Variantes cubiertas:
  - Se consulta refresh persistido por `jti`.
  - Se valida usuario asociado.
  - Se revoca el refresh anterior.
  - Se crea un refresh token nuevo con nuevo `jti`.
- Resultado esperado: Se emiten nuevos tokens y el refresh anterior queda revocado.
- Valor de negocio: Protege sesiones contra replay de refresh tokens reutilizados.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_service.py::test_refresh_session_rotates_valid_refresh_token`

### CU-U-AU-05: Refresh rechaza tokens inválidos o no vigentes

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: El refresh token debe cumplir contrato JWT y coincidir con el registro persistido.
- Objetivo: Validar que refresh tokens malformados, no persistidos, revocados, expirados o alterados son rechazados.
- Escenario: Se intenta renovar sesión con payload inválido, decode error o token persistido inválido.
- Variantes cubiertas:
  - Payload con `type` incorrecto.
  - `sub` no convertible a entero.
  - `jti` ausente o vacío.
  - Error al decodificar JWT.
  - Refresh inexistente.
  - Refresh de otro usuario.
  - Refresh revocado o expirado.
  - Hash persistido no coincide con token recibido.
- Resultado esperado: La renovación falla con `ValueError` y no emite nueva sesión.
- Valor de negocio: Evita renovación de sesiones con tokens manipulados o no vigentes.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_service.py::test_refresh_session_rejects_invalid_payload`
  - `tests/modules/auth/test_auth_service.py::test_refresh_session_rejects_decode_errors`
  - `tests/modules/auth/test_auth_service.py::test_refresh_session_rejects_invalid_persisted_token`

### CU-U-AU-06: Refresh rechaza usuario inexistente o inactivo

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Aunque el refresh token sea válido, la sesión no debe renovarse si el usuario ya no existe o está inactivo.
- Objetivo: Confirmar que el estado actual del usuario se valida durante renovación.
- Escenario: Se intenta refrescar una sesión asociada a usuario inexistente o inactivo.
- Variantes cubiertas:
  - Usuario no encontrado.
  - Usuario inactivo.
- Resultado esperado: La renovación falla y no se emiten tokens nuevos.
- Valor de negocio: Permite cortar acceso mediante desactivación de cuenta.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_service.py::test_refresh_session_rejects_missing_user`
  - `tests/modules/auth/test_auth_service.py::test_refresh_session_rejects_inactive_user`

### CU-U-AU-07: Logout revoca refresh token válido y tolera ausencia de token

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: El logout puede revocar el refresh token del dispositivo actual si el frontend lo envía.
- Objetivo: Validar revocación explícita y comportamiento idempotente cuando no se envía refresh token.
- Escenario: Se ejecuta logout con refresh token válido y luego sin refresh token.
- Variantes cubiertas:
  - Refresh token válido se revoca.
  - Ausencia de refresh token no intenta revocación.
- Resultado esperado: Logout con token revoca; logout sin token termina sin cambios persistidos.
- Valor de negocio: Permite cerrar sesión de forma segura sin romper clientes que no envían refresh token.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_service.py::test_logout_session_revokes_refresh_token`
  - `tests/modules/auth/test_auth_service.py::test_logout_session_without_refresh_token_does_not_revoke`

### CU-U-AU-08: Logout rechaza refresh inválido o de otro usuario

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Un usuario no debe poder revocar ni usar un refresh token que no corresponde a su sesión.
- Objetivo: Validar que logout con refresh token inválido se rechaza.
- Escenario: Se ejecuta logout con decode error, payload inválido, token de otro usuario o token persistido inexistente.
- Variantes cubiertas:
  - Decode error.
  - `type` distinto de `refresh`.
  - `sub` no coincide con usuario autenticado.
  - `jti` ausente o vacío.
  - Refresh persistido inexistente.
  - Refresh persistido pertenece a otro usuario.
- Resultado esperado: Logout falla con `ValueError` y no revoca tokens ajenos.
- Valor de negocio: Evita operaciones cruzadas entre sesiones o usuarios.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_service.py::test_logout_session_rejects_decode_errors`
  - `tests/modules/auth/test_auth_service.py::test_logout_session_rejects_invalid_payload`
  - `tests/modules/auth/test_auth_service.py::test_logout_session_rejects_invalid_persisted_token`

### CU-U-AU-09: TokenService emite y valida claims críticos

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Los JWT transportan identidad, roles, tipo de token, expiración y JTI.
- Objetivo: Proteger el contrato de access tokens, refresh tokens, OAuth state tokens y hashing de refresh tokens.
- Escenario: Se crean y decodifican tokens de acceso, refresh y estado OAuth.
- Variantes cubiertas:
  - Access token incluye `sub`, `email`, `roles`, `type=access` y `exp`.
  - Refresh token incluye `sub`, `jti`, `type=refresh` y `exp`.
  - Refresh token puede usar JTI provisto.
  - JTI generados son únicos.
  - Hash de token es estable y no igual al token original.
  - Verificación de hash acepta coincidencia y rechaza token distinto.
  - Decode de token válido retorna payload.
  - OAuth state token incluye claims esperados.
  - Token inválido lanza error.
- Resultado esperado: Los tokens mantienen claims críticos y verificaciones esperadas.
- Valor de negocio: Protege el contrato central usado por autenticación y autorización en todo el backend.
- Pruebas automatizadas:
  - `tests/modules/auth/test_token_service.py::test_create_access_token_contains_expected_claims`
  - `tests/modules/auth/test_token_service.py::test_create_refresh_token_contains_expected_claims`
  - `tests/modules/auth/test_token_service.py::test_create_refresh_token_uses_provided_jti`
  - `tests/modules/auth/test_token_service.py::test_generate_token_jti_returns_unique_values`
  - `tests/modules/auth/test_token_service.py::test_hash_token_returns_stable_hash`
  - `tests/modules/auth/test_token_service.py::test_verify_token_hash_returns_true_for_matching_token`
  - `tests/modules/auth/test_token_service.py::test_verify_token_hash_returns_false_for_different_token`
  - `tests/modules/auth/test_token_service.py::test_decode_token_returns_payload`
  - `tests/modules/auth/test_token_service.py::test_create_oauth_state_token_contains_expected_claims`
  - `tests/modules/auth/test_token_service.py::test_decode_oauth_state_token_returns_payload`
  - `tests/modules/auth/test_token_service.py::test_decode_token_raises_for_invalid_token`

### CU-U-AU-10: RefreshTokenRepository persiste, revoca y valida vigencia

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Los refresh tokens persistidos permiten revocación, rotación y verificación de vigencia.
- Objetivo: Validar operaciones básicas del repositorio y reglas de validez temporal.
- Escenario: Se crea, consulta, revoca y evalúa vigencia de refresh tokens.
- Variantes cubiertas:
  - Creación persiste y refresca entidad.
  - Consulta por `jti` retorna entidad.
  - Revocación marca `revoked_at`.
  - Token activo y no expirado es válido.
  - Token revocado no es válido.
  - Token expirado no es válido.
- Resultado esperado: El repositorio mantiene estado de revocación y expiración correctamente.
- Valor de negocio: Soporta cierre de sesión y refresh token rotation con persistencia revocable.
- Pruebas automatizadas:
  - `tests/modules/auth/test_refresh_token_repository.py::test_create_refresh_token_persists_and_refreshes_entity`
  - `tests/modules/auth/test_refresh_token_repository.py::test_get_refresh_token_by_jti_returns_matching_entity`
  - `tests/modules/auth/test_refresh_token_repository.py::test_revoke_refresh_token_sets_revoked_at`
  - `tests/modules/auth/test_refresh_token_repository.py::test_is_refresh_token_valid_returns_true_for_active_unexpired_token`
  - `tests/modules/auth/test_refresh_token_repository.py::test_is_refresh_token_valid_returns_false_for_revoked_token`
  - `tests/modules/auth/test_refresh_token_repository.py::test_is_refresh_token_valid_returns_false_for_expired_token`

### CU-U-AU-11: PasswordService hashea y valida contraseñas

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Las contraseñas no deben almacenarse ni compararse como texto plano.
- Objetivo: Validar hashing y verificación de contraseñas.
- Escenario: Se hashea una contraseña y se verifica con password correcto e incorrecto.
- Variantes cubiertas:
  - Hash generado no es igual al password original.
  - Password válido verifica correctamente.
  - Password inválido se rechaza.
- Resultado esperado: El servicio permite validar passwords sin exponer el secreto original.
- Valor de negocio: Protege credenciales de usuarios.
- Pruebas automatizadas:
  - `tests/modules/auth/test_password_service.py::test_hash_password_returns_hash`
  - `tests/modules/auth/test_password_service.py::test_verify_password_accepts_valid_password`
  - `tests/modules/auth/test_password_service.py::test_verify_password_rejects_invalid_password`

### CU-U-AU-12: RUT y teléfonos se normalizan y validan

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Los datos de identidad y contacto se usan transversalmente en usuarios, prácticas y notificaciones.
- Objetivo: Validar normalización y rechazo de formatos inválidos en schemas y service.
- Escenario: Se crean o actualizan usuarios con RUT y teléfonos en distintos formatos.
- Variantes cubiertas:
  - RUT válido se normaliza.
  - RUT inválido se rechaza.
  - Teléfono válido se normaliza a formato esperado.
  - Teléfono inválido se rechaza.
  - `UserCreateRequest` y `UserUpdateRequest` normalizan campos.
  - `UserService` persiste campos normalizados.
- Resultado esperado: Los datos quedan en formato consistente o se rechazan si son inválidos.
- Valor de negocio: Evita duplicados, inconsistencias y errores de contacto.
- Pruebas automatizadas:
  - `tests/modules/auth/test_normalization.py::test_normalize_rut_accepts_valid_values`
  - `tests/modules/auth/test_normalization.py::test_normalize_rut_rejects_invalid_values`
  - `tests/modules/auth/test_normalization.py::test_normalize_phone_accepts_valid_values`
  - `tests/modules/auth/test_normalization.py::test_normalize_phone_rejects_invalid_values`
  - `tests/modules/auth/test_user_schema.py::test_user_create_request_normalizes_rut_and_phone`
  - `tests/modules/auth/test_user_schema.py::test_user_update_request_normalizes_rut_and_phone`
  - `tests/modules/auth/test_user_schema.py::test_user_create_request_rejects_invalid_fields`
  - `tests/modules/auth/test_user_service.py::test_create_user_normalizes_rut_and_phones`
  - `tests/modules/auth/test_user_service.py::test_update_user_normalizes_fields`

### CU-U-AU-13: Dependencia de roles permite o rechaza según autorización

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Los módulos consumidores dependen de `require_roles` para proteger endpoints administrativos.
- Objetivo: Confirmar que un usuario con rol permitido accede y uno sin rol permitido recibe `403`.
- Escenario: Se evalúa la dependencia con roles permitidos y no permitidos.
- Variantes cubiertas:
  - Usuario con rol requerido.
  - Usuario sin rol requerido.
- Resultado esperado: La dependencia permite acceso si hay coincidencia de rol y rechaza si no la hay.
- Valor de negocio: Protege endpoints administrativos y de negocio por rol.
- Pruebas automatizadas:
  - `tests/modules/auth/test_role_dependency.py::test_require_roles_allows_when_role_present`
  - `tests/modules/auth/test_role_dependency.py::test_require_roles_rejects_when_role_missing`

### CU-U-AU-14: Google OAuth construye URL y valida callback

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: El flujo OAuth inicia redirigiendo a Google y luego procesa un callback con `code` y `state`.
- Objetivo: Validar parámetros críticos de autorización y errores controlados de callback.
- Escenario: Se construye URL OAuth y se procesan callbacks incompletos o con state inválido.
- Variantes cubiertas:
  - URL contiene `client_id`, `redirect_uri`, `response_type`, `scope` y `state`.
  - Callback sin parámetros requeridos se rechaza.
  - Callback con state inválido se rechaza.
- Resultado esperado: El flujo OAuth inicia con parámetros correctos y rechaza callbacks no confiables.
- Valor de negocio: Protege el flujo federado contra callbacks inválidos o alterados.
- Pruebas automatizadas:
  - `tests/modules/auth/test_google_oauth_service.py::test_build_authorization_url_contains_google_parameters`
  - `tests/modules/auth/test_google_oauth_service.py::test_authenticate_callback_rejects_missing_callback_params`
  - `tests/modules/auth/test_google_oauth_service.py::test_authenticate_callback_rejects_invalid_state`

### CU-U-AU-15: Google OAuth emite sesión para usuario existente

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Un usuario institucional existente puede autenticarse vía Google y recibir tokens internos del sistema.
- Objetivo: Validar que OAuth reutiliza el usuario local y crea sesión interna con roles existentes.
- Escenario: Google retorna identidad verificada de un usuario ya registrado.
- Variantes cubiertas:
  - Email verificado y dominio permitido.
  - Usuario existente encontrado por email.
  - Access token interno conserva roles del usuario.
  - Refresh token interno se persiste como hash.
- Resultado esperado: Se retorna `TokenResponse` interno para el usuario local existente.
- Valor de negocio: Permite login federado sin duplicar usuarios.
- Pruebas automatizadas:
  - `tests/modules/auth/test_google_oauth_service.py::test_authenticate_callback_issues_app_tokens_for_existing_user`

### CU-U-AU-16: Google OAuth crea estudiante para dominio permitido

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: Estudiantes institucionales pueden aprovisionarse automáticamente desde Google si pertenecen a un dominio permitido.
- Objetivo: Validar creación de usuario estudiante, hash de password sintético, RUT sintético y asignación de rol.
- Escenario: Google retorna identidad verificada de un email permitido que no existe localmente.
- Variantes cubiertas:
  - Se crea usuario activo y verificado.
  - Se genera password hash no conocido por el usuario.
  - Se genera RUT sintético `google:*`.
  - Se asigna rol `Estudiante`.
  - Se emiten tokens internos.
- Resultado esperado: El usuario nuevo queda creado como estudiante y recibe sesión interna.
- Valor de negocio: Reduce fricción de ingreso para cuentas institucionales permitidas.
- Pruebas automatizadas:
  - `tests/modules/auth/test_google_oauth_service.py::test_authenticate_callback_creates_student_for_allowed_domain`

### CU-U-AU-17: Google OAuth rechaza dominio o código inválido

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: El login federado solo debe aceptar cuentas institucionales permitidas y códigos válidos emitidos por Google.
- Objetivo: Validar rechazo controlado de identidades no autorizadas o intercambio de código fallido.
- Escenario: Se procesa identidad con dominio no autorizado y token endpoint rechaza el código.
- Variantes cubiertas:
  - Dominio no autorizado.
  - Código de autorización inválido.
- Resultado esperado: El service lanza `GoogleOAuthError` con código controlado.
- Valor de negocio: Evita acceso federado desde cuentas externas no permitidas.
- Pruebas automatizadas:
  - `tests/modules/auth/test_google_oauth_service.py::test_authenticate_callback_rejects_unauthorized_domain`
  - `tests/modules/auth/test_google_oauth_service.py::test_exchange_authorization_code_rejects_invalid_code`

### CU-U-AU-18: RoleService actualiza, asigna y remueve roles

- Tipo de prueba: Unitaria
- Dominio: Auth
- Contexto: La administración de roles permite ajustar permisos de usuarios y descripciones de roles.
- Objetivo: Validar operaciones básicas del service de roles.
- Escenario: Se actualiza descripción de rol, se asigna un rol a usuario y se remueve una asignación.
- Variantes cubiertas:
  - Actualización de descripción.
  - Creación de asignación usuario-rol.
  - Remoción de asignación existente.
- Resultado esperado: El service delega persistencia y construye asignaciones con IDs correctos.
- Valor de negocio: Soporta gestión administrativa de permisos.
- Pruebas automatizadas:
  - `tests/modules/auth/test_role_service.py::test_update_role_updates_description`
  - `tests/modules/auth/test_role_service.py::test_assign_role_creates_assignment`
  - `tests/modules/auth/test_role_service.py::test_remove_role_delegates_to_repository`

## Integración

### CU-I-AU-01: `get_current_user` rechaza refresh token como Bearer

- Tipo de prueba: Integración
- Dominio: Auth
- Contexto: Solo access tokens deben autorizar endpoints protegidos. Un refresh token no debe funcionar como Bearer token.
- Objetivo: Validar que la dependencia base rechaza payload con `type=refresh`.
- Escenario: Se invoca `get_current_user` con token cuyo payload no es de tipo `access`.
- Variantes cubiertas:
  - Token tipo `refresh` usado como Bearer.
- Resultado esperado: La dependencia responde `401 Unauthorized` con `WWW-Authenticate: Bearer`.
- Valor de negocio: Evita usar refresh tokens como credenciales de acceso directo.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_dependency.py::test_get_current_user_rejects_refresh_token`

### CU-I-AU-02: `get_current_user` valida access token y usuario vigente

- Tipo de prueba: Integración
- Dominio: Auth
- Contexto: La dependencia `get_current_user` es la base de autorización de todos los endpoints protegidos.
- Objetivo: Validar acceso exitoso con access token y rechazo de payload inválido, usuario inexistente o usuario inactivo.
- Escenario: Se resuelve el usuario autenticado desde un Bearer token.
- Variantes cubiertas:
  - Access token válido retorna usuario activo.
  - Error de decode devuelve `401`.
  - Payload sin `sub` o con `sub` inválido devuelve `401`.
  - Usuario inexistente devuelve `401`.
  - Usuario inactivo devuelve `401`.
- Resultado esperado: Access token válido retorna usuario activo; casos inválidos devuelven `401`.
- Valor de negocio: Protege todos los módulos que dependen de autenticación.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_dependency.py::test_get_current_user_returns_active_user_from_access_token`
  - `tests/modules/auth/test_auth_dependency.py::test_get_current_user_rejects_decode_errors`
  - `tests/modules/auth/test_auth_dependency.py::test_get_current_user_rejects_invalid_subject`
  - `tests/modules/auth/test_auth_dependency.py::test_get_current_user_rejects_missing_user`
  - `tests/modules/auth/test_auth_dependency.py::test_get_current_user_rejects_inactive_user`

### CU-I-AU-03: Controller de login traduce credenciales inválidas

- Tipo de prueba: Integración
- Dominio: Auth
- Contexto: El endpoint `/auth/login` expone errores de credenciales al frontend.
- Objetivo: Validar que credenciales inválidas se traducen a `401 Unauthorized` sin filtrar detalles.
- Escenario: Se llama al controller de login con email o password inválidos.
- Variantes cubiertas:
  - Login con JSON `email` y `password` válido retorna tokens.
  - Credenciales inválidas devuelven `401` con mensaje genérico.
- Resultado esperado: El endpoint devuelve `401` con mensaje genérico.
- Valor de negocio: Evita enumeración de usuarios y mantiene contrato HTTP claro.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_router.py::test_login_returns_tokens_for_valid_json_credentials`
  - `tests/modules/auth/test_auth_router.py::test_login_returns_401_for_invalid_credentials`

### CU-I-AU-04: Controller de refresh traduce refresh inválido

- Tipo de prueba: Integración
- Dominio: Auth
- Contexto: El endpoint `/auth/refresh` debe comunicar refresh inválido o expirado como falta de autorización.
- Objetivo: Validar la traducción `ValueError` del service a `401 Unauthorized`.
- Escenario: Se llama al controller con refresh token inválido.
- Variantes cubiertas:
  - Refresh inválido en service se traduce a `401`.
- Resultado esperado: El endpoint devuelve `401` y no emite tokens nuevos.
- Valor de negocio: Protege renovación de sesión frente a tokens inválidos.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_router.py::test_refresh_returns_401_for_invalid_refresh_token`

### CU-I-AU-05: Controller de logout traduce refresh inválido

- Tipo de prueba: Integración
- Dominio: Auth
- Contexto: Logout requiere Bearer token válido y puede recibir refresh token opcional para revocación.
- Objetivo: Validar que refresh token inválido en logout se traduce a `400 Bad Request`.
- Escenario: Usuario autenticado intenta logout con refresh token inválido o ajeno.
- Variantes cubiertas:
  - Refresh inválido en logout se traduce a `400`.
- Resultado esperado: El endpoint devuelve `400` y no revoca tokens ajenos.
- Valor de negocio: Evita operaciones inconsistentes de cierre de sesión.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_router.py::test_logout_returns_400_for_invalid_refresh_token`

### CU-I-AU-06: `/auth/me` no expone credenciales ni campos sensibles

- Tipo de prueba: Integración
- Dominio: Auth
- Contexto: El endpoint de usuario actual retorna información de sesión al frontend.
- Objetivo: Validar que la respuesta contiene identidad y roles, pero no `password_hash` ni refresh tokens.
- Escenario: Usuario autenticado consulta `/auth/me`.
- Variantes cubiertas:
  - Respuesta incluye identidad y roles.
  - Respuesta no expone `password_hash` ni `refresh_tokens`.
- Resultado esperado: La respuesta incluye id, email, nombre, apellido y roles; no incluye secretos.
- Valor de negocio: Evita exposición accidental de credenciales.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_router.py::test_get_me_returns_current_user_without_sensitive_fields`

### CU-I-AU-07: Google controller maneja cookie, state y redirect

- Tipo de prueba: Integración
- Dominio: Auth
- Contexto: El controller de Google OAuth agrega protección CSRF mediante cookie `state` y redirige al frontend.
- Objetivo: Validar que login setea cookie HTTP-only, callback inválido limpia cookie y callback exitoso redirige correctamente.
- Escenario: Se ejecuta flujo controller de Google login/callback.
- Variantes cubiertas:
  - Login redirige a Google y setea cookie `state` HTTP-only.
  - Callback con state inconsistente redirige a error y limpia cookie.
  - Callback exitoso redirige con token y limpia cookie.
  - Error controlado del service redirige a error y limpia cookie.
- Resultado esperado: La cookie de state se gestiona correctamente y los redirects contienen éxito o error según corresponda.
- Valor de negocio: Protege el flujo OAuth navegador-backend-frontend.
- Pruebas automatizadas:
  - `tests/modules/auth/test_auth_router.py::test_google_login_redirects_and_sets_http_only_state_cookie`
  - `tests/modules/auth/test_auth_router.py::test_google_callback_state_mismatch_redirects_error_and_clears_cookie`
  - `tests/modules/auth/test_auth_router.py::test_google_callback_success_redirects_token_and_clears_cookie`
  - `tests/modules/auth/test_auth_router.py::test_google_callback_service_error_redirects_error_and_clears_cookie`

## End-to-End

### CU-E2E-AU-01: Login local, consulta de usuario actual y logout

- Tipo de prueba: End-to-end
- Dominio: Auth
- Contexto: Este es el flujo básico de sesión para cualquier usuario del sistema.
- Objetivo: Validar credenciales, emisión de tokens, acceso a endpoint protegido y cierre de sesión.
- Escenario: Usuario inicia sesión con email/password, consulta `/auth/me` y ejecuta logout con refresh token.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: Login entrega tokens, `/auth/me` responde usuario correcto y logout revoca sesión.
- Valor de negocio: Da confianza sobre el flujo de autenticación principal.
- Pruebas automatizadas:
  - Pendiente de implementación.

### CU-E2E-AU-02: Login y rotación de refresh token

- Tipo de prueba: End-to-end
- Dominio: Auth
- Contexto: La renovación de sesión debe rotar refresh tokens y evitar reutilización del anterior.
- Objetivo: Validar el flujo completo de refresh rotation atravesando endpoint y persistencia.
- Escenario: Usuario inicia sesión, usa refresh token para renovar, intenta reutilizar el refresh anterior.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: El primer refresh emite nueva sesión; el refresh anterior queda revocado y no puede reutilizarse.
- Valor de negocio: Protege sesiones contra replay de refresh tokens.
- Pruebas automatizadas:
  - Pendiente de implementación.

### CU-E2E-AU-03: Google OAuth completo en entorno controlado

- Tipo de prueba: End-to-end
- Dominio: Auth
- Contexto: El login federado atraviesa navegador, Google, backend y frontend.
- Objetivo: Validar el flujo completo de Google OAuth con proveedor o doble controlado.
- Escenario: Usuario institucional inicia OAuth, se valida state, se procesa callback y se emiten tokens internos.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: Usuario autorizado recibe sesión interna; cuentas no autorizadas son rechazadas.
- Valor de negocio: Da confianza sobre el acceso federado institucional.
- Pruebas automatizadas:
  - Pendiente de implementación.
