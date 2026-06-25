# Casos de Prueba - Auth

## Alcance

Estos casos documentan las pruebas de valor del módulo `auth`. El foco está en autenticación local, emisión y rotación de tokens, revocación de refresh tokens, resolución del usuario autenticado, autorización por roles, normalización de datos de identidad, Google OAuth y administración básica de roles.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos de seguridad funcional o contratos transversales del backend.

## Integración

### CU-I-AU-01: `get_current_user` rechaza refresh token como Bearer

- **Tipo de prueba:** Integración
- **Dominio:** Auth
- **Contexto:** Solo access tokens deben autorizar endpoints protegidos. Un refresh token no debe funcionar como Bearer token.
- **Objetivo:** Validar que la dependencia base rechaza payload con `type=refresh`.
- **Escenario:** Se invoca `get_current_user` con token cuyo payload no es de tipo `access`.
- **Variantes cubiertas:**
  - Token tipo `refresh` usado como Bearer.
- **Resultado esperado:** La dependencia responde `401 Unauthorized` con `WWW-Authenticate: Bearer`.
- **Valor de negocio:** Evita usar refresh tokens como credenciales de acceso directo.
- **Pruebas automatizadas:**
  - `tests/modules/auth/test_auth_dependency.py::test_get_current_user_rejects_refresh_token`

### CU-I-AU-02: Controller de login expone contrato de sesión

- **Tipo de prueba:** Integración
- **Dominio:** Auth
- **Contexto:** El endpoint `/auth/login` autentica credenciales JSON y debe emitir una sesión usable por navegador sin exponer detalles ante errores.
- **Objetivo:** Validar emisión de tokens, cookie HTTP-only de refresh y traducción de errores relevantes.
- **Escenario:** Se llama al controller de login con credenciales válidas, credenciales inválidas y usuario con password temporal pendiente.
- **Variantes cubiertas:**
  - Login válido retorna access token, refresh token y setea cookie HTTP-only.
  - Credenciales inválidas devuelven `401` con mensaje genérico.
  - Password temporal pendiente devuelve `403` con contrato específico para cambio obligatorio.
- **Resultado esperado:** El login válido emite sesión y los errores se traducen sin filtrar detalles sensibles.
- **Valor de negocio:** Protege el punto de entrada principal y mantiene contrato compatible con clientes web.
- **Pruebas automatizadas:**
  - `tests/modules/auth/test_auth_router.py::test_login_returns_tokens_and_sets_refresh_cookie`
  - `tests/modules/auth/test_auth_router.py::test_login_returns_401_for_invalid_credentials`
  - `tests/modules/auth/test_auth_router.py::test_login_returns_403_for_temporary_password`

### CU-I-AU-03: Controller de refresh renueva sesión desde body o cookie

- **Tipo de prueba:** Integración
- **Dominio:** Auth
- **Contexto:** El endpoint `/auth/refresh` debe aceptar refresh token desde body o cookie HTTP-only y renovar la cookie cuando emite sesión nueva.
- **Objetivo:** Validar fuentes admitidas de refresh token, cookie de salida y rechazo de token ausente o inválido.
- **Escenario:** Se llama al controller con refresh token en body, desde cookie o sin token válido.
- **Variantes cubiertas:**
  - Refresh desde body emite nuevos tokens y setea cookie actualizada.
  - Refresh desde cookie se usa cuando el body no contiene token.
  - Token ausente o inválido devuelve `401`.
- **Resultado esperado:** Solo un refresh token disponible y válido renueva la sesión; casos inválidos no emiten tokens nuevos.
- **Valor de negocio:** Soporta clientes web seguros y evita renovaciones sin credencial válida.
- **Pruebas automatizadas:**
  - `tests/modules/auth/test_auth_router.py::test_refresh_uses_body_token_and_sets_new_cookie`
  - `tests/modules/auth/test_auth_router.py::test_refresh_uses_cookie_when_body_token_is_missing`
  - `tests/modules/auth/test_auth_router.py::test_refresh_returns_401_for_missing_or_invalid_token`

### CU-I-AU-04: Controller de logout revoca sesión y limpia cookie

- **Tipo de prueba:** Integración
- **Dominio:** Auth
- **Contexto:** Logout debe revocar el refresh token enviado y limpiar la cookie del navegador.
- **Objetivo:** Validar cierre de sesión correcto y traducción de refresh inválido a error de contrato.
- **Escenario:** Usuario autenticado ejecuta logout con refresh válido o inválido.
- **Variantes cubiertas:**
  - Logout válido revoca refresh token y limpia cookie.
  - Refresh inválido en logout se traduce a `400`.
- **Resultado esperado:** Logout exitoso devuelve `204`; refresh inválido no revoca tokens ajenos.
- **Valor de negocio:** Permite cierre de sesión seguro y consistente entre backend y navegador.
- **Pruebas automatizadas:**
  - `tests/modules/auth/test_auth_router.py::test_logout_revokes_refresh_token_and_clears_cookie`
  - `tests/modules/auth/test_auth_router.py::test_logout_returns_400_for_invalid_refresh_token`

### CU-I-AU-05: `/auth/me` no expone credenciales ni campos sensibles

- **Tipo de prueba:** Integración
- **Dominio:** Auth
- **Contexto:** El endpoint de usuario actual retorna información de sesión al frontend.
- **Objetivo:** Validar que la respuesta contiene identidad y roles, pero no `password_hash` ni refresh tokens.
- **Escenario:** Usuario autenticado consulta `/auth/me`.
- **Variantes cubiertas:**
  - Respuesta incluye identidad y roles.
  - Respuesta no expone `password_hash` ni `refresh_tokens`.
- **Resultado esperado:** La respuesta incluye id, email, nombre, apellido y roles; no incluye secretos.
- **Valor de negocio:** Evita exposición accidental de credenciales.
- **Pruebas automatizadas:**
  - `tests/modules/auth/test_auth_router.py::test_get_me_returns_current_user_without_sensitive_fields`

### CU-I-AU-06: Activación de cuenta expone contrato de estado inicial

- **Tipo de prueba:** Integración
- **Dominio:** Auth
- **Contexto:** Usuarios creados administrativamente pueden activar su cuenta y consultar datos mínimos de activación antes de definir contraseña.
- **Objetivo:** Validar que el controller delega activación, retorna información segura y traduce errores de token.
- **Escenario:** Se activa una cuenta o se consulta información de activación con token válido e inválido.
- **Variantes cubiertas:**
  - Activación válida devuelve `204` y pasa datos requeridos al servicio.
  - Error de activación se traduce a `400`.
  - Consulta de información retorna email, roles y datos iniciales.
  - Error de consulta se traduce a `400`.
- **Resultado esperado:** El flujo expone solo datos necesarios y mantiene errores controlados para tokens inválidos o expirados.
- **Valor de negocio:** Soporta onboarding seguro sin passwords temporales reutilizables.
- **Pruebas automatizadas:**
  - `tests/modules/auth/test_auth_router.py::test_activate_account_delegates_and_maps_activation_errors`
  - `tests/modules/auth/test_auth_router.py::test_activation_info_returns_data_and_maps_activation_errors`

### CU-I-AU-07: Google controller maneja cookie, state y redirect

- **Tipo de prueba:** Integración
- **Dominio:** Auth
- **Contexto:** El controller de Google OAuth agrega protección CSRF mediante cookie `state`, redirige al frontend y entrega refresh token como cookie HTTP-only.
- **Objetivo:** Validar que login setea cookie HTTP-only, callback inválido limpia cookie, callback exitoso redirige correctamente y persiste refresh en cookie.
- **Escenario:** Se ejecuta flujo controller de Google login/callback.
- **Variantes cubiertas:**
  - Login redirige a Google y setea cookie `state` HTTP-only.
  - Callback con state inconsistente redirige a error y limpia cookie.
  - Callback exitoso redirige con access token, limpia cookie de state y setea refresh cookie.
  - Error controlado del service redirige a error y limpia cookie.
- **Resultado esperado:** La cookie de state se gestiona correctamente, el refresh token queda en cookie HTTP-only y los redirects contienen éxito o error según corresponda.
- **Valor de negocio:** Protege el flujo OAuth navegador-backend-frontend.
- **Pruebas automatizadas:**
  - `tests/modules/auth/test_auth_router.py::test_google_login_redirects_and_sets_http_only_state_cookie`
  - `tests/modules/auth/test_auth_router.py::test_google_callback_state_mismatch_redirects_error_and_clears_cookie`
  - `tests/modules/auth/test_auth_router.py::test_google_callback_success_redirects_token_and_sets_refresh_cookie`
  - `tests/modules/auth/test_auth_router.py::test_google_callback_service_error_redirects_error_and_clears_cookie`
