<h1 align="center"><em>Autenticación</em></h1>

> [!NOTE]
> Esta documentación describe la implementación real del módulo de autenticación en el proyecto, incluyendo servicios, dependencias, endpoints y flujos.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Estructura del módulo](#estructura-del-módulo)
- [Endpoints disponibles](#endpoints-disponibles)
- [Definición de schemas](#definición-de-schemas)
- [Flujo principal](#flujo-principal)
- [Roles y permisos](#roles-y-permisos)
- [Consideraciones operativas](#consideraciones-operativas)


---


## Resumen operativo
El módulo de autenticación expone endpoints para:
- Iniciar sesión generando tokens JWT de tipo access (corta duración) y refresh (larga duración)
- Consultar el usuario que está actualmente autenticado
- Cerrar la sesión sin revocación real de tokens
- Permitir la gestión de usuarios y roles por usuarios administradores


> [!IMPORTANT]
>La autenticación se basa únicamente en JWT y los permisos se controlan por los roles que tenga asignado un usuario.


---


## Estructura del módulo
El módulo de autenticación se encuentra en la ruta `app/modules/auth/` donde a continuación se desglosa su estructura interna:

- `controllers/`
  - `auth_controller.py`
  - `user_controller.py`
  - `role_controller.py`
- `dependencies/`
  - `auth_dependency.py`
  - `role_dependency.py`
- `models/`
  - `role_model.py`
  - `user_model.py`
  - `user_role_model.py`
- `repositories/`
  - `role_repository.py`
  - `user_repository.py`
  - `user_role_repository.py`
- `schemas/`
  - `auth_schema.py`
  - `rol_schema.py`
  - `token_schema.py`
  - `user_schema.py`
- `services/`
  - `auth_service.py`
  - `password_service.py`
  - `role_service.py`
  - `token_service.py`
  - `user_service.py`
- `utils/normalization.py`


---


## Endpoints disponibles
| Método | Ruta | Auth | Requiere (request) | Devuelve |
|---|---|---|---|---|
| POST | `/auth/login` | Público | JSON: `email`, `password` | `TokenResponse` (200) |
| GET | `/auth/me` | Token | Header Auth | `CurrentUserResponse` (200) |
| POST | `/auth/logout` | Token | Header Auth; JSON opcional: `refresh_token` | 204 no content |
| POST | `/users` | Admin | JSON: `email`, `password`, `first_name`, `last_name`, `rut` + opcionales | `UserResponse` (201) |
| GET | `/users` | Admin | — | `list[UserResponse]` (200) |
| GET | `/users/{user_id}` | Admin | URL Path: `user_id` | `UserResponse` (200) |
| PATCH | `/users/{user_id}` | Admin | URL Path: `user_id`; JSON: `first_name`, `last_name`, `rut`, etc. | `UserResponse` (200) |
| GET | `/users/{user_id}/roles` | Admin | URL Path: `user_id` | `list[UserRoleResponse]` (200) |
| POST | `/users/{user_id}/roles` | Admin | URL Path: `user_id`; JSON: `role_id` | `UserRoleResponse` (201) |
| DELETE | `/users/{user_id}/roles/{role_id}` | Admin | URL Path: `user_id`, `role_id` | 204 no content |
| GET | `/roles` | Admin | — | `list[RoleResponse]` (200) |
| GET | `/roles/{role_id}` | Admin | URL Path: `role_id` | `RoleResponse` (200) |
| PATCH | `/roles/{role_id}` | Admin | URL Path: `role_id`; JSON: `description` | `RoleResponse` (200) |

> [!NOTE]
> Cuando el nivel de autenticación se indica como **Token**, se refiere a que sólamente se necesita el `access_token` del usuario incluido en el payload, mientras que para el nivel **Admin** ese token además debe estar asociado a un usuario perteneciente al grupo de roles administradores.

> [!WARNING]
> Antes de cualquier consulta es necesario verificar la existencia de al menos un usuario que tenga asignado algún rol administrador en la base de datos. En caso contrario el sistema por diseño no permite la creación de usuarios administradores, provocando un deadlock.
---

## Definición de schemas
| Schema | Campos requeridos | Campos opcionales |
|---|---|---|
| `LoginRequest` | `email`, `password` | — |
| `TokenResponse` | `access_token`, `refresh_token`, `token_type` | — |
| `CurrentUserResponse` | `id`, `email`, `first_name`, `last_name`, `roles` | — |
| `UserCreateRequest` | `email`, `password`, `first_name`, `last_name`, `rut` | `degree`, `cod_degree`, `sexo`, `phone`, `profession`, `position`, `departament`, `sup_phone` |
| `UserUpdateRequest` | — | `first_name`, `last_name`, `rut`, `degree`, `cod_degree`, `sexo`, `phone`, `profession`, `position`, `departament`, `sup_phone`, `is_active` |
| `UserResponse` | `id`, `email`, `first_name`, `last_name`, `rut`, `is_active`, `is_verified`, `created_at` | `degree`, `cod_degree`, `sexo`, `phone`, `profession`, `position`, `departament`, `sup_phone` |
| `RoleResponse` | `id`, `name`, `description`, `created_at` | — |
| `RoleUpdateRequest` | `description` | — |
| `UserRoleResponse` | `id`, `name` | — |
| `AssignRoleRequest` | `role_id` | — |

> [!IMPORTANT]
> Se deben respetar los tipos para cada atributo. Para mayor detalle consultar las tablas creadas en PostgreSQL


---


## Flujo principal
### 1. Solicitud del cliente

El cliente envía una petición `POST /auth/login` con el siguiente payload JSON:

```json
{
  "email": "user@example.com",
  "password": "********"
}
```


### 2. Validación de entrada

El schema `LoginRequest` valida:

- Formato correcto del `email`
- Tamaño y restricciones del `password`


### 3. Inicialización del controlador de autenticación

El archivo `auth_controller.py` recibe la solicitud e inicializa los siguientes componentes:

- `UserRepository` → acceso a base de datos
- `PasswordService` → hash y verificación de contraseñas
- `TokenService` → generación de JWT
- `AuthService` → lógica de autenticación y orquestación


### 4. Autenticación del usuario

Se ejecuta:

```python
AuthService.authenticate_user(email, password)
```

#### Flujo interno

1. Se busca el usuario por email:

```python
UserRepository.get_user_by_email(email)
```

2. Si el usuario no existe:
   - Se retorna `None`
   - Se registra el intento fallido en logs

3. Si el usuario existe:
   - Se valida la contraseña mediante:

```python
PasswordService.verify_password()
```

4. Si la contraseña es inválida:
   - Se retorna `None`
   - Se registra el intento fallido en logs


### 5. Generación de tokens

Si las credenciales son válidas:

#### Access Token

Se genera un JWT con los siguientes claims:

- `sub` → ID del usuario
- `email`
- `roles`
- `exp` → fecha de expiración

#### Refresh Token

Se genera un JWT con:

- `sub`
- `exp`


### 6. Respuesta al cliente

El endpoint retorna un `TokenResponse`:

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "token_type": "bearer"
}
```


### 7. Errores esperados

Si el usuario no existe o la contraseña es inválida:

- Se lanza una excepción HTTP `401 Unauthorized`
- Mensaje retornado:

```json
{
  "detail": "Invalid email or password"
}
```


---


## Roles y permisos
A día de hoy, el módulo de autenticación espera que existan como mínimo los siguientes roles:
- **Estudiante**
- **Supervisor de práctica**
- **Encargado de práctica**
- **Director de carrera**
- **Secretaria de carrera**

### Reglas actuales
- **Administradores**: todos excepto Estudiante.
- Todos los endpoints de `/users` y `/roles` pueden ser accedidos ùnicamente por los administradores.
- Más adelante se manejará el caso especial para el Supervisor de práctica.


---


## Consideraciones operativas
- Los tokens no se almacenan en servidor. Frontend es el único responsable de almacenarlos y enviarlos según corresponda.
- Se está considerando la inclusión de una funcionalidad para iniciar sesión con una cuenta de Google, pero no ha sido implementada de momento.
- También se está evaluando agregar una funcionalidad para restablecer contraseñas en caso de olvido por correo, pero tampoco ha sido discutido con el equipo de momento.
