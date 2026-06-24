<h1 align="center"><em>Auth</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo `auth`
> desde una perspectiva funcional e interna. Su objetivo es explicar cómo está
> implementado y qué debe saber alguien antes de modificarlo. El contrato HTTP
> formal queda en OpenAPI y el detalle interactivo queda en Swagger.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Ámbito y responsabilidades](#ámbito-y-responsabilidades)
- [Estructura interna](#estructura-interna)
- [Funcionalidades](#funcionalidades)
- [Endpoints disponibles](#endpoints-disponibles)
- [Contratos principales](#contratos-principales)
- [Reglas de negocio](#reglas-de-negocio)
- [Configuración por entorno](#configuración-por-entorno)
- [Consideraciones operativas](#consideraciones-operativas)

## Resumen operativo

El módulo **`auth`** gestiona autenticación, sesiones JWT, refresh tokens
persistidos, autorización por roles y administración básica de usuarios y roles.

**Permite:**

- Iniciar sesión con email y password.
- Renovar sesiones mediante refresh tokens rotativos.
- Revocar refresh tokens durante logout.
- Completar cambio de contraseña temporal.
- Activar cuentas nuevas mediante enlace de un solo uso.
- Resolver el usuario autenticado desde un Bearer token.
- Restringir endpoints mediante roles.
- Iniciar sesión con Google OAuth.
- Administrar usuarios y asignaciones de roles.

## Ámbito y responsabilidades

El módulo **`auth`** centraliza identidad, sesión y autorización transversal para
el backend. También expone endpoints administrativos para usuarios y roles.

#### Responsabilidades principales

- Validación de credenciales locales.
- Emisión de `access_token` y `refresh_token`.
- Persistencia, rotación y revocación de refresh tokens.
- Activación de cuentas nuevas mediante token de un solo uso.
- Reemplazo de credenciales temporales.
- Validación de Bearer tokens para endpoints protegidos.
- Autorización basada en roles mediante `require_roles`.
- Inicio de sesión con Google OAuth y aprovisionamiento básico de estudiantes.
- Creación, consulta y actualización administrativa de usuarios.
- Consulta, actualización y asignación administrativa de roles.

#### Fuera de alcance

- Recuperación o cambio de contraseña por endpoint HTTP.
- Permisos granulares distintos a roles.
- Reglas de negocio propias de prácticas, documentos o dashboard administrativo.
- Gestión documental o notificaciones funcionales.

> [!IMPORTANT]
> Aunque `auth` administra usuarios y roles, los permisos concretos de cada flujo
> de negocio se definen en los módulos consumidores mediante `require_roles`.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/auth/controllers/auth_controller.py` | Define endpoints de sesión, usuario actual y Google OAuth. |
| Controller | `app/modules/auth/controllers/user_controller.py` | Define endpoints administrativos de usuarios y asignaciones de roles. |
| Controller | `app/modules/auth/controllers/role_controller.py` | Define endpoints administrativos de roles. |
| Dependencies | `app/modules/auth/dependencies/auth_dependency.py` | Resuelve el usuario autenticado desde Bearer token. |
| Dependencies | `app/modules/auth/dependencies/role_dependency.py` | Valida que el usuario tenga al menos un rol permitido. |
| Service | `app/modules/auth/services/auth_service.py` | Orquesta login, refresh, creación de sesión y logout. |
| Service | `app/modules/auth/services/token_service.py` | Crea, decodifica y valida tokens JWT. |
| Service | `app/modules/auth/services/google_oauth_service.py` | Orquesta el flujo OAuth con Google. |
| Service | `app/modules/auth/services/user_service.py` | Crea y actualiza usuarios con normalización y password hash. |
| Service | `app/modules/auth/services/role_service.py` | Lista, actualiza, asigna y remueve roles. |
| Repository | `app/modules/auth/repositories/...` | Encapsula persistencia de usuarios, roles, refresh tokens y tokens de activación. |
| Models | `app/modules/auth/models/...` | Define entidades ORM de usuarios, roles, asignaciones, refresh tokens y activación de cuenta. |
| Schemas | `app/modules/auth/schemas/...` | Define contratos de entrada y salida del módulo. |
| Utils | `app/modules/auth/utils/normalization.py` | Normaliza RUT y teléfonos. |

El módulo reutiliza configuración desde `app/core/config.py` y sesiones de base de
datos mediante `app/core/database/database.py`.

## Funcionalidades

#### Login con email y password

1. El cliente llama a `POST /auth/login` con un cuerpo JSON que incluye `email` y
   `password`.
2. El controller construye `AuthService` con repositorios y servicios internos.
3. `AuthService` busca el usuario por email.
4. `PasswordService` valida la contraseña contra `password_hash`.
5. Si las credenciales son válidas, se emiten `access_token` y `refresh_token`.
6. El refresh token se persiste como hash en `refresh_tokens`.
7. El refresh token también se configura en cookie HTTP-only.
8. Se retorna `TokenResponse`.

Si el usuario debe reemplazar una contraseña temporal, el backend responde `403`
con detalle `TEMPORARY_PASSWORD_CHANGE_REQUIRED`.

#### Contraseña temporal y activación de cuenta

1. Un usuario con credencial temporal llama a
   `POST /auth/complete-temporary-password`.
2. Envía email, contraseña temporal y nueva contraseña.
3. Si la credencial temporal es válida, se reemplaza por la contraseña definitiva.
4. Una cuenta nueva puede consultar datos mínimos con `GET /auth/activation-info`.
5. La activación final se completa con `POST /auth/activate-account` usando token
   de enlace, nueva contraseña y año de ingreso opcional.

#### Renovación de sesión

1. El cliente llama a `POST /auth/refresh`.
2. El refresh token puede venir en el body JSON o en la cookie HTTP-only
   configurada por login/OAuth.
3. `TokenService` decodifica el JWT y exige `type=refresh`.
4. `AuthService` obtiene el token persistido por `jti`.
5. Se valida que no esté revocado, no esté expirado y coincida con su hash.
6. Se verifica que el usuario exista y esté activo.
7. El refresh token anterior se revoca.
8. Se crea una nueva sesión con nuevos tokens y se actualiza la cookie.

#### Logout y revocación

1. El cliente llama a `POST /auth/logout` con Bearer token válido.
2. El refresh token puede venir en el body JSON o en la cookie HTTP-only.
3. Si hay refresh token, se valida que pertenezca al usuario actual.
4. El refresh token persistido se marca con `revoked_at`.
5. La cookie de refresh se limpia.
6. El endpoint responde `204 No Content`.

#### Usuario autenticado

1. El cliente llama a `GET /auth/me` con `Authorization: Bearer <token>`.
2. `get_current_user` decodifica el access token.
3. Se exige `type=access` y un `sub` convertible a entero.
4. Se consulta el usuario en base de datos.
5. Se rechaza si el usuario no existe o está inactivo.
6. Se retorna `CurrentUserResponse` con sus roles.

#### Autorización por roles

1. El módulo consumidor declara `Depends(require_roles([...]))`.
2. La dependencia obtiene el usuario actual mediante `get_current_user`.
3. Se comparan los roles del usuario con los roles permitidos.
4. Si existe al menos una coincidencia, se permite continuar.
5. Si no hay coincidencias, se retorna `403`.

#### Login con Google OAuth

1. El navegador abre `GET /auth/google/login`.
2. El backend genera un `state` firmado y lo guarda en cookie HTTP-only.
3. El usuario es redirigido a Google.
4. Google vuelve a `GET /auth/google/callback` con `code` y `state`.
5. El backend valida que el `state` coincida con la cookie.
6. Se intercambia el `code` por tokens de Google y se valida el `id_token`.
7. Se exige email verificado y dominio permitido.
8. Si el usuario no existe, se crea como `Estudiante`.
9. Se emiten tokens internos, se guarda el refresh token en cookie `HttpOnly` y
   se redirige al callback del frontend con el access token.

#### Administración de usuarios

1. Un usuario con rol administrativo llama a endpoints `/users`.
2. El controller valida autenticación y roles administrativos.
3. La creación verifica unicidad de email y RUT.
4. Si se envía contraseña, `UserService` la hashea.
5. Si no se envía contraseña, se prepara activación por enlace.
6. `UserService` normaliza RUT y teléfonos.
7. La actualización solo modifica campos enviados.

#### Administración de roles

1. Un usuario con rol administrativo llama a endpoints `/roles` o `/users/{user_id}/roles`.
2. El controller valida autenticación y roles administrativos.
3. Se pueden listar roles, actualizar descripciones, asignar roles y remover asignaciones.
4. La asignación exige que usuario y rol existan.
5. No se permite duplicar una asignación usuario-rol.

## Endpoints disponibles

| Método | Ruta | Propósito | Acceso |
| --- | --- | --- | --- |
| POST | `/auth/login` | Inicia sesión con email y password. | Público |
| POST | `/auth/complete-temporary-password` | Reemplaza una credencial temporal por contraseña definitiva. | Público con credencial temporal válida |
| GET | `/auth/activation-info` | Consulta datos mínimos de una cuenta pendiente de activación. | Público con token de activación |
| POST | `/auth/activate-account` | Activa una cuenta mediante enlace de un solo uso. | Público con token de activación |
| POST | `/auth/refresh` | Rota un refresh token y emite nuevos tokens. | Refresh token |
| GET | `/auth/google/login` | Redirige al flujo OAuth de Google. | Público |
| GET | `/auth/google/callback` | Procesa callback OAuth y redirige al frontend. | Público con `state` válido |
| GET | `/auth/me` | Retorna el usuario autenticado. | Bearer token |
| POST | `/auth/logout` | Cierra sesión y revoca refresh token si fue enviado. | Bearer token |
| POST | `/users` | Crea un usuario. | Superadmin |
| GET | `/users` | Lista usuarios con filtros opcionales. | Superadmin |
| GET | `/users/{user_id}` | Obtiene un usuario por ID. | Superadmin |
| PATCH | `/users/{user_id}` | Actualiza parcialmente un usuario. | Superadmin |
| GET | `/users/{user_id}/roles` | Lista roles de un usuario. | Superadmin |
| POST | `/users/{user_id}/roles` | Asigna un rol a un usuario. | Superadmin |
| DELETE | `/users/{user_id}/roles/{role_id}` | Remueve una asignación usuario-rol. | Superadmin |
| GET | `/roles` | Lista roles existentes. | Superadmin |
| GET | `/roles/{role_id}` | Obtiene un rol por ID. | Superadmin |
| PATCH | `/roles/{role_id}` | Actualiza descripción de un rol. | Superadmin |

## Contratos principales

<details>
<summary><strong>LoginRequest</strong></summary>

Payload JSON usado por `POST /auth/login`.

```json
{
  "email": "student@correo.cl",
  "password": "my_secure_password"
}
```

</details>

<details>
<summary><strong>CompleteTemporaryPasswordRequest</strong></summary>

Payload usado cuando un usuario debe reemplazar una credencial temporal por una
contraseña definitiva.

```json
{
  "email": "student@correo.cl",
  "temporary_password": "temporary_password",
  "new_password": "my_secure_password"
}
```

</details>

<details>
<summary><strong>ActivationAccountInfoResponse</strong></summary>

Respuesta de `GET /auth/activation-info` para mostrar datos mínimos antes de
activar una cuenta.

```json
{
  "email": "student@correo.cl",
  "first_name": "Juan",
  "last_name": "Pérez",
  "roles": ["Estudiante"],
  "admission_year": 2024
}
```

</details>

<details>
<summary><strong>ActivateAccountRequest</strong></summary>

Payload usado para activar una cuenta nueva mediante enlace de un solo uso.

```json
{
  "token": "activation-token",
  "new_password": "my_secure_password",
  "admission_year": 2024
}
```

</details>

<details>
<summary><strong>TokenResponse</strong></summary>

Respuesta estándar para login, refresh y Google OAuth interno.

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "token_type": "bearer"
}
```

</details>

<details>
<summary><strong>RefreshTokenRequest</strong></summary>

Payload usado por `POST /auth/refresh`.

```json
{
  "refresh_token": "..."
}
```

El body es opcional si el refresh token viene en la cookie HTTP-only configurada
por el backend.

</details>

<details>
<summary><strong>LogoutRequest</strong></summary>

Payload opcional usado por `POST /auth/logout` para revocar la sesión actual.

```json
{
  "refresh_token": "..."
}
```

El body es opcional si el refresh token viene en la cookie HTTP-only configurada
por el backend.

</details>

<details>
<summary><strong>CurrentUserResponse</strong></summary>

Respuesta de `GET /auth/me`.

```json
{
  "id": 1,
  "email": "student@correo.cl",
  "first_name": "Juan",
  "last_name": "Pérez",
  "roles": ["Estudiante"]
}
```

</details>

<details>
<summary><strong>UserCreateRequest</strong></summary>

Payload administrativo para crear usuarios.

```json
{
  "email": "user@correo.cl",
  "password": null,
  "first_name": "Juan",
  "last_name": "Pérez",
  "rut": "12.345.678-9",
  "degree": "Ingeniería Civil Informática",
  "cod_degree": "ICI",
  "admission_year": 2024,
  "phone": "+56912345678",
  "role_ids": [1]
}
```

Si no se envía `password`, el backend genera una credencial interna y el usuario
define su contraseña mediante enlace de activación.

</details>

<details>
<summary><strong>UserUpdateRequest</strong></summary>

Payload administrativo para actualización parcial de usuarios.

```json
{
  "first_name": "Juan Carlos",
  "phone": "+56987654321",
  "is_active": true
}
```

</details>

<details>
<summary><strong>UserResponse</strong></summary>

Respuesta administrativa con datos del usuario.

```json
{
  "id": 1,
  "email": "student@correo.cl",
  "first_name": "Juan",
  "last_name": "Pérez",
  "rut": "12.345.678-9",
  "degree": "Ingeniería Civil Informática",
  "cod_degree": null,
  "sexo": null,
  "phone": "+56912345678",
  "profession": null,
  "position": null,
  "departament": null,
  "sup_phone": null,
  "is_active": true,
  "is_verified": false,
  "created_at": "2026-06-12T18:30:00Z"
}
```

</details>

<details>
<summary><strong>RoleResponse</strong></summary>

Representa un rol del sistema.

```json
{
  "id": 1,
  "name": "Estudiante",
  "description": "Rol correspondiente a estudiantes en practicas",
  "created_at": "2026-06-12T18:30:00Z"
}
```

</details>

<details>
<summary><strong>AssignRoleRequest</strong></summary>

Payload para asignar un rol a un usuario.

```json
{
  "role_id": 2
}
```

</details>

## Reglas de negocio

#### Tokens de acceso

El access token es el token que autoriza endpoints protegidos.

**Claims principales:**

| Claim | Uso |
| --- | --- |
| `sub` | ID del usuario como string. |
| `email` | Correo del usuario autenticado. |
| `roles` | Lista de roles asociados. |
| `type` | Debe ser `access`. |
| `exp` | Expiración del token. |

> [!WARNING]
> `get_current_user` rechaza tokens que no tengan `type=access`.

#### Refresh tokens

El refresh token permite renovar sesión sin reutilizar credenciales.

**Reglas actuales:**

- Incluye `sub`, `jti`, `type=refresh` y `exp`.
- Se almacena en base de datos como hash, no como texto plano.
- Debe existir en `refresh_tokens` para ser aceptado.
- Debe coincidir con el hash persistido.
- Debe estar vigente y sin `revoked_at`.
- Se revoca durante cada renovación exitosa.
- La renovación emite un nuevo access token y un nuevo refresh token.
- Puede recibirse desde body JSON o desde cookie HTTP-only.
- En una renovación exitosa se actualiza la cookie de refresh.

#### Logout y revocación

`POST /auth/logout` siempre exige Bearer token válido.

**Reglas actuales:**

- Si no se envía `refresh_token` ni existe cookie, el endpoint responde `204` y
  limpia la cookie si estaba presente en el cliente.
- Si se envía `refresh_token`, debe pertenecer al usuario autenticado.
- Si el refresh token es inválido o pertenece a otro usuario, se retorna `400`.
- Si es válido, se marca `revoked_at` en base de datos.
- Al finalizar, la cookie de refresh se elimina de la respuesta.

#### Usuario autenticado

`get_current_user` es la dependencia base para endpoints protegidos.

**Rechaza la solicitud si:**

- El JWT expiró o es inválido.
- El token no es de tipo `access`.
- El payload no incluye `sub` válido.
- El usuario no existe.
- El usuario está inactivo.

#### Autorización por roles

`require_roles` permite acceso si el usuario tiene al menos uno de los roles
permitidos por el endpoint.

**Rol administrativo usado por `/users` y `/roles`:**

- `Superadmin`

> [!IMPORTANT]
> Otros roles del sistema pueden tener permisos de negocio en sus propios módulos,
> pero la administración de usuarios y roles queda reservada a `Superadmin`.

#### Google OAuth

El flujo OAuth permite autenticar cuentas Google y emitir tokens internos del
sistema.

**Reglas actuales:**

- Requiere `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` y URLs OAuth configuradas.
- Usa un `state` firmado y guardado en cookie HTTP-only.
- Valida firma, audiencia, expiración e issuer del `id_token` de Google.
- Exige email verificado.
- Restringe dominios según `GOOGLE_ALLOWED_DOMAINS`.
- Por defecto permite `ufromail.cl` y `ufrontera.cl`.
- Si el usuario no existe, lo crea con rol `Estudiante`.
- El RUT de usuarios creados por Google usa formato sintético `google:<hash>`.
- Si el login es exitoso, el callback redirige a
  `GOOGLE_FRONTEND_SUCCESS_URL?token=<access_token>` y configura la cookie
  `REFRESH_TOKEN_COOKIE_NAME` como `HttpOnly`.
- Si el login falla, redirige a `GOOGLE_FRONTEND_ERROR_URL?error=<codigo>`.

**Códigos de error esperados por frontend:**

- `unauthorized_domain`
- `invalid_callback`
- `missing_token`
- `server_unavailable`
- `user_not_found`

#### Administración de usuarios

**Reglas actuales:**

- Crear usuario exige email único.
- Crear usuario exige RUT único.
- `password` es opcional al crear usuarios administrativos.
- Si se entrega contraseña, se almacena hasheada.
- Si no se entrega contraseña, el usuario debe activar su cuenta mediante enlace.
- `rut`, `phone` y `sup_phone` se normalizan antes de persistir.
- Los usuarios creados por endpoint administrativo quedan `is_active=True`.
- Los usuarios creados por endpoint administrativo quedan `is_verified=False`.
- La creación puede recibir `role_ids` para asignar roles iniciales.
- La actualización solo modifica campos enviados.

#### Activación de cuenta y contraseña temporal

**Reglas actuales:**

- Los tokens de activación son de un solo uso.
- `GET /auth/activation-info` expone solo datos mínimos para renderizar la vista
  de activación.
- `POST /auth/activate-account` define contraseña definitiva y puede registrar
  `admission_year`.
- `POST /auth/complete-temporary-password` reemplaza una credencial temporal por
  una contraseña definitiva.
- Si un usuario intenta login con una contraseña temporal pendiente de cambio, el
  backend responde `TEMPORARY_PASSWORD_CHANGE_REQUIRED`.

#### Administración de roles

**Reglas actuales:**

- Los roles base se cargan desde `app/core/database/init.sql`.
- La asignación de rol exige que usuario y rol existan.
- No se permite asignar dos veces el mismo rol al mismo usuario.
- Remover un rol exige que la asignación exista.
- El endpoint de roles solo actualiza la descripción del rol.

## Configuración por entorno

| Variable | Uso |
| --- | --- |
| `JWT_SECRET_KEY` | Secreto usado para firmar y validar tokens JWT. |
| `JWT_ALGORITHM` | Algoritmo JWT usado por `TokenService`. Por defecto `HS256`. |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | Duración de access tokens. |
| `REFRESH_TOKEN_EXPIRE_DAYS` | Duración de refresh tokens persistidos. |
| `REFRESH_TOKEN_COOKIE_NAME` | Nombre de la cookie HTTP-only usada para refresh/logout. |
| `REFRESH_TOKEN_COOKIE_SECURE` | Controla si la cookie de refresh exige HTTPS. |
| `GOOGLE_CLIENT_ID` | Client ID OAuth usado para construir la autorización y validar audiencia. |
| `GOOGLE_CLIENT_SECRET` | Secreto OAuth usado al intercambiar el código de autorización. |
| `GOOGLE_REDIRECT_URI` | Callback backend registrado ante Google. |
| `GOOGLE_ALLOWED_DOMAINS` | Dominios de correo permitidos para login Google. |
| `GOOGLE_FRONTEND_SUCCESS_URL` | URL frontend a la que se redirige un login Google exitoso. |
| `GOOGLE_FRONTEND_ERROR_URL` | URL frontend a la que se redirigen errores de login Google. |
| `GOOGLE_STATE_EXPIRE_MINUTES` | Vigencia del token `state` usado en OAuth. |
| `GOOGLE_STATE_COOKIE_NAME` | Nombre de la cookie HTTP-only usada para validar `state`. |
| `GOOGLE_COOKIE_SECURE` | Controla si la cookie OAuth exige HTTPS. |

## Consideraciones operativas

- `/auth/login` usa JSON con `email` y `password`.
- `/auth/login` y el callback de Google configuran una cookie HTTP-only con el
  refresh token.
- `/auth/refresh` y `/auth/logout` aceptan refresh token por JSON body o por la
  cookie HTTP-only configurada por el backend.
- `JWT_SECRET_KEY` debe configurarse en entornos reales.
- `REFRESH_TOKEN_COOKIE_SECURE=True` debe usarse cuando la API opere sobre HTTPS.
- `GOOGLE_COOKIE_SECURE=True` debe usarse cuando el callback OAuth opere sobre HTTPS.
- `init.sql` siembra roles base y asigna usuario 1 como `Estudiante` y usuario 2 como `Director de carrera`.
- Los endpoints `/users` y `/roles` requieren que exista al menos un usuario con
  rol `Superadmin` para operar.
