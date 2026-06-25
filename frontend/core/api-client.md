<h1 align="center"><em>Cliente API frontend</em></h1>

> [!NOTE]
> Esta documentacion tecnica describe el cliente HTTP actual del frontend. Su
> objetivo es explicar como se configuran las llamadas al backend, como se
> adjuntan tokens, como se renueva la sesion y que patron siguen los servicios.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Ambito y responsabilidades](#ambito-y-responsabilidades)
- [Estructura interna](#estructura-interna)
- [Cliente Axios](#cliente-axios)
- [Interceptores](#interceptores)
- [Servicios consumidores](#servicios-consumidores)
- [Integracion con backend](#integracion-con-backend)
- [Limites actuales](#limites-actuales)

## Resumen operativo

El frontend centraliza la comunicacion HTTP con el backend en
`src/services/api.js`. Este archivo crea una instancia Axios compartida,
configura la URL base desde `VITE_API_URL`, adjunta el access token a las
peticiones y gestiona renovacion automatica de sesion.

**Permite:**

- Reutilizar una unica instancia Axios para llamadas al backend.
- Configurar la URL base de API mediante variable de entorno Vite.
- Adjuntar `Authorization: Bearer <token>` cuando existe sesion local.
- Renovar el access token antes de su expiracion.
- Reintentar una peticion protegida despues de un `401` si el refresh funciona.
- Limpiar sesion local y redirigir a login cuando no se puede renovar.
- Encapsular endpoints en servicios de dominio bajo `src/services`.

## Ambito y responsabilidades

El cliente API es la capa tecnica de transporte HTTP del frontend. No contiene
reglas funcionales de negocio; solo prepara peticiones, gestiona sesion a nivel
HTTP y delega los contratos concretos a servicios especializados.

#### Responsabilidades principales

- Crear la instancia Axios compartida.
- Leer `VITE_API_URL` como `baseURL`.
- Leer tokens desde `localStorage`.
- Adjuntar el access token a peticiones no excluidas.
- Decodificar localmente `exp` del JWT para anticipar refresh.
- Ejecutar `POST /auth/refresh` cuando corresponde.
- Evitar multiples refresh concurrentes mediante `refreshRequest`.
- Reintentar una peticion original despues de renovar token.
- Redirigir a `/login` cuando la sesion queda invalida.

#### Fuera de alcance

- Definir contratos formales de endpoints.
- Validar reglas de negocio.
- Interpretar todos los errores funcionales del backend.
- Decidir permisos o roles.
- Persistir datos de dominio.
- Validar criptograficamente JWT.

> [!IMPORTANT]
> `api.js` puede leer el payload del JWT para conocer su expiracion, pero no
> valida firma ni permisos. La validacion real pertenece al backend.

## Estructura interna

| Archivo | Responsabilidad |
| --- | --- |
| `src/services/api.js` | Instancia Axios, interceptores, refresh y limpieza de sesion. |
| `src/services/authService.js` | Login, logout, usuario actual y URL OAuth. |
| `src/services/internshipService.js` | Operaciones frontend sobre practicas. |
| `src/services/coordinatorService.js` | Operaciones administrativas. |
| `src/services/documentService.js` | Operaciones documentales. |
| `src/services/notificationService.js` | Consulta de notificaciones. |
| `src/services/roleRouting.js` | Redireccion local segun roles. |
| `src/services/oauthErrors.js` | Traduccion de errores OAuth para interfaz. |

## Cliente Axios

La instancia compartida se crea en `src/services/api.js`:

```js
const api = axios.create({
    baseURL: import.meta.env.VITE_API_URL,
});
```

`VITE_API_URL` define la URL base usada por todos los servicios que importan
`api`.

Ejemplos esperados:

```env
VITE_API_URL=http://localhost:8000
```

```env
VITE_API_URL=/api
```

El segundo caso se usa cuando el frontend se sirve bajo el mismo origen y Nginx
redirige `/api` hacia el backend.

## Interceptores

#### Interceptor de request

El interceptor de request esta definido en `src/services/api.js` mediante
`api.interceptors.request.use(...)`.

Antes de enviar una peticion, este interceptor:

1. Lee `token` desde `localStorage`.
2. Detecta si la peticion es `/auth/login` o `/auth/refresh`.
3. Si no es una peticion de autenticacion y el token esta cerca de expirar,
   intenta renovarlo.
4. Si la renovacion falla, limpia sesion local y redirige a `/login`.
5. Si hay token disponible, agrega `Authorization: Bearer <token>`.
6. Devuelve la configuracion final de Axios.

Las peticiones a `/auth/login` y `/auth/refresh` se excluyen del refresh
preventivo para evitar ciclos de renovacion.

#### Margen de renovacion

El cliente usa un margen de 30 segundos:

```js
const TOKEN_REFRESH_MARGIN_SECONDS = 30;
```

Si el access token expira en 30 segundos o menos, el cliente intenta renovarlo
antes de enviar la peticion original.

#### Lectura de expiracion JWT

`getTokenExpiration` separa el JWT, decodifica el payload con `window.atob` y lee
el campo `exp`.

Si el token no puede decodificarse o no contiene expiracion valida, el cliente no
intenta refresh preventivo por expiracion.

#### Refresh compartido

`refreshAccessToken` usa una variable de modulo llamada `refreshRequest`.

Mientras existe una renovacion en curso, las siguientes peticiones reutilizan la
misma promesa en vez de disparar varios `POST /auth/refresh` simultaneos.

El flujo es:

1. Lee `refresh_token` desde `localStorage`.
2. Si no existe, lanza `missing_refresh_token`.
3. Si no hay refresh activo, ejecuta `POST /auth/refresh`.
4. Guarda los nuevos `access_token` y `refresh_token`.
5. Devuelve el nuevo access token.
6. Al finalizar, libera `refreshRequest`.

#### Interceptor de response

El interceptor de response esta definido en `src/services/api.js` mediante
`api.interceptors.response.use(...)`.

Cuando una respuesta falla, este interceptor:

1. Obtiene la peticion original desde `error.config`.
2. Verifica si el status es `401`.
3. Evita reintentar si la peticion ya tiene `_retry`.
4. Evita reintentar `/auth/login` y `/auth/refresh`.
5. Marca `_retry = true`.
6. Intenta renovar token.
7. Si obtiene nuevo token, actualiza `Authorization` y reejecuta la peticion.
8. Si falla, limpia sesion local y redirige a `/login`.
9. Rechaza el error si no puede resolverlo.

## Servicios consumidores

Los servicios de dominio importan `api` y encapsulan endpoints concretos.

El patron actual es:

```js
async operationName(params) {
  const response = await api.get('/resource', { params });
  return response.data;
}
```

Los servicios retornan `response.data` y dejan que paginas o hooks manejen estado
de carga, errores visibles y decisiones de interfaz.

#### Servicios actuales

| Servicio | Endpoints principales |
| --- | --- |
| `authService.js` | `/auth/login`, `/auth/me`, `/auth/logout`, `/auth/google/login` |
| `internshipService.js` | `/internships`, `/internships/me`, `/internships/{id}`, acciones de practica |
| `coordinatorService.js` | `/admin/summary`, `/admin/internships`, acciones administrativas |
| `documentService.js` | `/documents/types`, documentos por practica, descarga, revision y eliminacion |
| `notificationService.js` | `/notifications`, `/notifications/{id}` |

#### Manejo de errores

El cliente Axios no normaliza todos los errores funcionales. Los errores se
propagan a los consumidores y normalmente se interpretan en paginas o hooks.

Ejemplo actual: `usePractice` distingue `403`, `404` y errores genericos para
mostrar mensajes especificos en la vista de detalle administrativo.

## Integracion con backend

El cliente espera que el backend acepte Bearer token en endpoints protegidos:

```http
Authorization: Bearer <access_token>
```

Para renovacion, el cliente envia:

```json
{
  "refresh_token": "jwt"
}
```

a:

```http
POST /auth/refresh
```

y espera recibir:

```json
{
  "access_token": "jwt",
  "refresh_token": "jwt"
}
```

El detalle formal de contratos HTTP pertenece al backend.

## Limites actuales

- El cliente depende de `localStorage` para leer access token y refresh token.
- El refresh preventivo depende de que el access token sea un JWT decodificable
  con campo `exp`.
- No existe normalizacion global de errores de negocio.
- No hay cancelacion centralizada de requests al desmontar componentes.
- No hay cache HTTP ni deduplicacion general de consultas fuera del refresh.
- La URL base queda resuelta por Vite al construir la aplicacion.
