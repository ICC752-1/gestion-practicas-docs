# Casos de Prueba - Auth

## Alcance

Estos casos documentan las pruebas de valor del módulo `auth`. El foco está en autenticación local, emisión y rotación de tokens, revocación de refresh tokens, resolución del usuario autenticado, autorización por roles, normalización de datos de identidad, Google OAuth y administración básica de roles.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos de seguridad funcional o contratos transversales del backend.

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
