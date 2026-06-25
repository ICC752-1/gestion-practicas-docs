<h1 align="center"><em>Autenticacion frontend</em></h1>

> [!NOTE]
> Esta documentacion tecnica describe el comportamiento actual de autenticacion
> en el frontend. Su objetivo es explicar como se inicia, restaura, renueva y
> cierra una sesion desde la interfaz, y que puntos deben conocerse antes de
> modificar este flujo.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Ambito y responsabilidades](#ambito-y-responsabilidades)
- [Estructura interna](#estructura-interna)
- [Flujos principales](#flujos-principales)
- [Tokens y sesion local](#tokens-y-sesion-local)
- [Roles y usuario autenticado](#roles-y-usuario-autenticado)
- [Integracion con backend](#integracion-con-backend)
- [Consideraciones de seguridad](#consideraciones-de-seguridad)

## Resumen operativo

El frontend gestiona la sesion de usuario mediante `AuthContext`, `authService`,
`PrivateRoute` y el cliente Axios centralizado en `src/services/api.js`.

**Permite:**

- Iniciar sesion con email y password.
- Redirigir al flujo OAuth de Google expuesto por el backend.
- Procesar el callback OAuth del backend.
- Restaurar sesion al recargar la aplicacion.
- Obtener el usuario autenticado desde `/auth/me`.
- Mantener access token y refresh token en `localStorage`.
- Renovar tokens automaticamente desde interceptores Axios.
- Cerrar sesion local y remotamente.
- Restringir vistas segun roles del usuario autenticado.

## Ambito y responsabilidades

El frontend administra el estado de sesion necesario para renderizar vistas,
llamar endpoints protegidos y aplicar restricciones visuales por rol.

#### Responsabilidades principales

- Guardar tokens retornados por el backend.
- Exponer estado de autenticacion mediante `AuthContext`.
- Resolver el usuario actual mediante `/auth/me`.
- Limpiar sesion local cuando el token deja de ser valido.
- Anadir `Authorization: Bearer <token>` a peticiones protegidas.
- Renovar tokens antes de su expiracion o tras respuestas `401`.
- Redirigir usuarios no autenticados a `/login`.
- Mostrar denegacion visual cuando el usuario no tiene un rol permitido.

#### Fuera de alcance

- Validar credenciales directamente.
- Emitir, firmar o verificar criptograficamente tokens.
- Decidir permisos reales de negocio.
- Persistir refresh tokens de forma segura fuera del navegador.
- Sustituir la autorizacion del backend.
- Definir el contrato formal de autenticacion.

> [!IMPORTANT]
> El frontend usa roles y sesion para controlar navegacion e interfaz, pero la
> seguridad efectiva depende de la validacion del backend en cada endpoint
> protegido.

## Estructura interna

| Archivo | Responsabilidad |
| --- | --- |
| `src/context/AuthContext.jsx` | Mantiene usuario, token, estado de carga, errores y operaciones de sesion. |
| `src/context/auth-context.js` | Define el contexto React de autenticacion. |
| `src/context/useAuth.js` | Expone el hook consumidor de autenticacion. |
| `src/services/authService.js` | Encapsula login, logout, `/auth/me` y URL OAuth. |
| `src/services/api.js` | Anade token, renueva sesion y maneja `401` desde Axios. |
| `src/components/PrivateRoute.jsx` | Protege rutas segun sesion y roles permitidos. |
| `src/routes/AppRoutes.jsx` | Declara rutas publicas, protegidas y roles permitidos. |
| `src/pages/Auth/AuthCallbackPage.jsx` | Procesa el retorno del flujo OAuth. |
| `src/services/roleRouting.js` | Resuelve el destino local recomendado segun roles. |
| `src/services/oauthErrors.js` | Traduce codigos de error OAuth a mensajes de interfaz. |

## Flujos principales

#### Login con email y password

1. El usuario envia email y password desde la vista de login.
2. `AuthContext.login` llama a `authService.login`.
3. `authService.login` ejecuta `POST /auth/login`.
4. Si el backend responde correctamente, se guardan `access_token` y
   `refresh_token` en `localStorage`.
5. `AuthContext` actualiza el token local.
6. El frontend llama a `authService.getMe`.
7. `GET /auth/me` retorna el usuario autenticado y sus roles.
8. `AuthContext` guarda el usuario y finaliza el estado de carga.
9. La navegacion posterior depende de los roles del usuario.

Si el backend responde `401`, el contexto define el error visible como
`Credenciales invalidas`. Si no hay respuesta HTTP, se muestra `Servidor no
disponible`.

#### Restauracion de sesion

1. `AuthContext` inicializa `token` leyendo `localStorage.getItem("token")`.
2. Si no hay token, finaliza la carga sin usuario autenticado.
3. Si existe token, llama a `authService.getMe`.
4. Si `/auth/me` responde correctamente, guarda el usuario autenticado.
5. Si la consulta falla, elimina `token` y `refresh_token` de `localStorage`.
6. La sesion queda como no autenticada.

Este flujo permite mantener sesion despues de recargar la pagina, siempre que el
access token siga siendo valido o pueda renovarse desde el cliente Axios.

#### Login con Google OAuth

1. La interfaz obtiene la URL de login con `authService.getGoogleLoginUrl`.
2. El navegador abre el endpoint `/auth/google/login` del backend.
3. El backend completa el flujo con Google.
4. El backend redirige al frontend en `/auth/callback`.
5. `AuthCallbackPage` delega el procesamiento en
   `AuthContext.handleOAuthCallback`.
6. `handleOAuthCallback` revisa si la URL trae `error`.
7. Si hay error, lanza un error OAuth local.
8. Si no hay error, exige el parametro `token`.
9. Guarda el access token en `localStorage`.
10. Llama a `/auth/me` para resolver usuario y roles.
11. Si `/auth/me` falla, limpia sesion local y reporta callback invalido.
12. Si el usuario se resuelve correctamente, guarda el usuario autenticado.

> [!WARNING]
> En el flujo OAuth actual el callback frontend guarda el access token recibido
> en query string. El refresh token no se guarda explicitamente desde este flujo
> en `AuthContext`; la continuidad de sesion depende del comportamiento del
> backend y del cliente HTTP.

#### Renovacion automatica de tokens

La renovacion automatica esta centralizada en `src/services/api.js`.

1. Antes de cada request, el interceptor lee `token` desde `localStorage`.
2. Si la peticion no es `/auth/login` ni `/auth/refresh`, evalua la expiracion
   del JWT.
3. Si el access token expira en 30 segundos o menos, llama a
   `refreshAccessToken`.
4. `refreshAccessToken` lee `refresh_token` desde `localStorage`.
5. Ejecuta `POST /auth/refresh` con `{ "refresh_token": "<token>" }`.
6. Si la respuesta es valida, reemplaza `token` y `refresh_token` en
   `localStorage`.
7. La peticion original continua con el nuevo `Authorization`.

Para evitar renovaciones concurrentes, `api.js` reutiliza una promesa global
`refreshRequest` mientras existe una renovacion en curso.

#### Reintento tras respuesta 401

1. Si una respuesta retorna `401`, el interceptor revisa si la peticion puede
   reintentarse.
2. No reintenta `/auth/login` ni `/auth/refresh`.
3. Marca la peticion original con `_retry`.
4. Intenta renovar tokens con `refreshAccessToken`.
5. Si la renovacion funciona, reemplaza `Authorization` y reejecuta la peticion.
6. Si la renovacion falla, limpia la sesion y redirige a `/login`.

#### Logout

1. La interfaz llama a `AuthContext.logout`.
2. `AuthContext.logout` delega en `authService.logout`.
3. `authService.logout` lee `refresh_token` desde `localStorage`.
4. Si existe refresh token, intenta llamar a `POST /auth/logout`.
5. Si el backend falla, el cierre local continua igualmente.
6. El servicio elimina `token` y `refresh_token`.
7. `AuthContext` limpia `user` y `token`.
8. El navegador redirige a `/landing`.

## Tokens y sesion local

El frontend usa las siguientes claves de `localStorage`:

| Clave | Uso |
| --- | --- |
| `token` | Access token JWT usado en `Authorization`. |
| `refresh_token` | Refresh token usado para renovar sesion. |

`src/services/api.js` decodifica localmente el payload del JWT solo para leer
`exp` y decidir si conviene renovar el access token. Esta lectura no valida la
firma del token y no debe usarse como mecanismo de seguridad.

La sesion se limpia cuando:

- Falla la restauracion inicial con `/auth/me`.
- No existe refresh token durante una renovacion necesaria.
- `/auth/refresh` falla.
- Una respuesta `401` no puede resolverse con refresh.
- El usuario ejecuta logout.

## Roles y usuario autenticado

El usuario autenticado se obtiene desde `/auth/me` y se guarda en `AuthContext`.

`PrivateRoute` usa `user.roles` para comparar contra `allowedRoles` definidos en
`AppRoutes.jsx`.

Los grupos actuales son:

| Grupo frontend | Roles |
| --- | --- |
| Estudiantes | `Estudiante` |
| Administracion | `Encargado de practica`, `Director de carrera`, `Secretaria de Carrera` |
| Supervisores | `Supervisor de practica` |

Cuando el usuario esta autenticado pero no tiene un rol permitido, `PrivateRoute`
muestra una pantalla de acceso denegado y ofrece navegar al panel resuelto por
`getRedirectPathForRoles`.

## Integracion con backend

| Operacion | Endpoint |
| --- | --- |
| Login local | `POST /auth/login` |
| Usuario actual | `GET /auth/me` |
| Refresh token | `POST /auth/refresh` |
| Logout | `POST /auth/logout` |
| Inicio OAuth Google | `GET /auth/google/login` |
| Callback OAuth backend | Redireccion hacia `/auth/callback` en frontend |

El frontend espera que `POST /auth/login` y `POST /auth/refresh` retornen al
menos:

```json
{
  "access_token": "jwt",
  "refresh_token": "jwt"
}
```

El frontend espera que `/auth/me` retorne un usuario con arreglo de roles:

```json
{
  "id": 1,
  "email": "user@example.com",
  "roles": ["Estudiante"]
}
```

El detalle formal de estos contratos pertenece al backend.

## Consideraciones de seguridad

- Guardar tokens en `localStorage` expone la sesion a riesgos si ocurre XSS.
- La validacion local de expiracion JWT no valida firma ni permisos.
- Las restricciones de `PrivateRoute` solo controlan navegacion y renderizado.
- Los roles del frontend deben mantenerse alineados con los roles reales del
  backend.
- Los endpoints protegidos deben seguir validando Bearer token y roles en
  backend.
- El logout local debe completarse aunque el backend no pueda revocar el refresh
  token.
