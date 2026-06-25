<h1 align="center"><em>Rutas y roles frontend</em></h1>

> [!NOTE]
> Esta documentacion tecnica describe el sistema actual de rutas, proteccion de
> vistas y redireccion por roles en el frontend. Su objetivo es explicar como se
> organiza la navegacion y que limites tiene la autorizacion aplicada en la
> interfaz.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Ambito y responsabilidades](#ambito-y-responsabilidades)
- [Estructura interna](#estructura-interna)
- [Rutas publicas](#rutas-publicas)
- [Rutas protegidas](#rutas-protegidas)
- [Roles frontend](#roles-frontend)
- [Flujo de proteccion de rutas](#flujo-de-proteccion-de-rutas)
- [Redireccion por rol](#redireccion-por-rol)
- [Limites actuales](#limites-actuales)

## Resumen operativo

El frontend centraliza la definicion de rutas en `src/routes/AppRoutes.jsx`.
Las rutas publicas renderizan vistas accesibles sin sesion, mientras que las
rutas privadas se envuelven con `PrivateRoute` y declaran roles permitidos
mediante `allowedRoles`.

**Permite:**

- Redirigir `/` hacia `/landing`.
- Exponer vistas publicas de landing, login, callback OAuth y FAQ.
- Proteger vistas funcionales mediante sesion activa.
- Restringir vistas protegidas segun roles del usuario autenticado.
- Mostrar una pantalla de acceso denegado cuando el rol no coincide.
- Sugerir un panel de destino segun los roles del usuario.
- Mantener la tabla de rutas en un punto central del frontend.

## Ambito y responsabilidades

La capa de rutas define que vista debe renderizarse para cada URL y aplica
restricciones visuales basadas en autenticacion y roles.

#### Responsabilidades principales

- Declarar rutas publicas y protegidas.
- Asociar cada ruta con su pagina React correspondiente.
- Definir grupos de roles usados por `PrivateRoute`.
- Redirigir rutas obsoletas o de compatibilidad interna.
- Evitar que usuarios no autenticados entren a vistas privadas.
- Evitar que usuarios autenticados accedan a vistas fuera de su rol.
- Resolver un destino local recomendado segun roles.

#### Fuera de alcance

- Autorizar operaciones reales de negocio.
- Validar permisos en backend.
- Decidir transiciones de estado de practicas.
- Garantizar que un usuario no pueda llamar endpoints manualmente.
- Definir la fuente oficial de roles del sistema.

> [!IMPORTANT]
> La proteccion de rutas en frontend solo controla navegacion y renderizado. Los
> endpoints protegidos deben validar autenticacion y roles en backend.

## Estructura interna

| Archivo | Responsabilidad |
| --- | --- |
| `src/routes/AppRoutes.jsx` | Declara rutas, redirecciones y roles permitidos por vista. |
| `src/components/PrivateRoute.jsx` | Evalua sesion, carga y roles antes de renderizar una vista protegida. |
| `src/services/roleRouting.js` | Resuelve el panel local recomendado segun roles del usuario. |
| `src/context/AuthContext.jsx` | Provee `isAuthenticated`, `loading` y `user.roles`. |
| `src/context/useAuth.js` | Expone el estado de autenticacion a `PrivateRoute`. |

## Rutas publicas

Las rutas publicas actuales no usan `PrivateRoute`.

| Ruta | Vista | Proposito |
| --- | --- | --- |
| `/` | `Navigate` hacia `/landing` | Redireccion inicial. |
| `/landing` | `LandingPage` | Pagina publica de entrada. |
| `/login` | `Login` | Inicio de sesion local u OAuth. |
| `/auth/callback` | `AuthCallbackPage` | Procesamiento del retorno OAuth. |
| `/faq` | `FAQPage` | Preguntas frecuentes. |

## Rutas protegidas

Las rutas protegidas se envuelven con `PrivateRoute` y reciben `allowedRoles`.

| Ruta | Vista | Roles permitidos |
| --- | --- | --- |
| `/inscripcion` | `Navigate` hacia `/practicas/nueva/preinscripcion` | Redireccion interna. |
| `/practicas/nueva/preinscripcion` | `PreRegistrationPage` | `Estudiante` |
| `/practicas/nueva/formulario` | `RegistrationPage` | `Estudiante` |
| `/dashboard` | `StudentDashboardPage` | `Estudiante` |
| `/coordinador` | `CoordinatorDashboardPage` | `Encargado de practica`, `Director de carrera`, `Secretaria de Carrera` |
| `/coordinador/practica/:id` | `PracticeDetailPage` | `Encargado de practica`, `Director de carrera`, `Secretaria de Carrera` |
| `/seguimiento` | `SeguimientoListPage` | `Estudiante` |
| `/seguimiento/:internshipId` | `SeguimientoPage` | `Estudiante` |
| `/supervisor` | `SupervisorPage` | `Supervisor de practica` |
| `/autoevaluacion` | `SelfEvaluationPage` | `Estudiante` |
| `/entrevistas` | `InterviewSchedulingPage` | `Estudiante` |

> [!WARNING]
> `/inscripcion` no renderiza una pagina propia. Solo redirige a
> `/practicas/nueva/preinscripcion`.

## Roles frontend

`AppRoutes.jsx` define grupos locales de roles para proteger rutas.

```js
const STUDENT_ROLES = ['Estudiante']

const ADMIN_ROLES = [
    'Encargado de practica',
    'Director de carrera',
    'Secretaria de Carrera',
]

const SUPERVISOR_ROLES = ['Supervisor de practica']
```

Estos grupos se usan en `allowedRoles` dentro de `PrivateRoute`.

Su responsabilidad es responder esta pregunta:

**"Este usuario puede entrar a esta ruta concreta?"**

Por ejemplo, para `/coordinador`:

```jsx
<PrivateRoute allowedRoles={ADMIN_ROLES}>
    <CoordinatorDashboardPage />
</PrivateRoute>
```

Si el usuario no tiene alguno de los roles de `ADMIN_ROLES`, la ruta no se
renderiza y se muestra la pantalla de acceso denegado.

`roleRouting.js`, en cambio, no protege rutas. Su responsabilidad es resolver a
que panel deberia enviarse un usuario segun sus roles.

Su pregunta es:

**"Si necesito mandar a este usuario a su panel principal, cual ruta conviene?"**

Actualmente usa esta lista:

```js
const coordinatorRoles = [
    "Encargado de practica",
    "Director de carrera",
    "Coordinador",
    "Coordinador FICA",
    "Secretaria de Carrera",
];
```

`getRedirectPathForRoles` devuelve:

| Rol detectado | Destino |
| --- | --- |
| `Estudiante` | `/dashboard` |
| Rol administrativo reconocido | `/coordinador` |
| `Supervisor de practica` | `/supervisor` |
| Sin coincidencia | `/landing` |

Por eso ambos archivos pueden parecer similares, pero se usan en momentos
distintos:

| Archivo | Pregunta que responde | Momento de uso |
| --- | --- | --- |
| `AppRoutes.jsx` | Puede entrar a esta ruta? | Antes de renderizar una ruta protegida. |
| `roleRouting.js` | A que panel deberia ir? | Al redirigir despues de login o desde acceso denegado. |

> [!IMPORTANT]
> `AppRoutes.jsx` y `roleRouting.js` no cumplen la misma funcion. `AppRoutes.jsx`
> es la lista de permisos por ruta; `roleRouting.js` es una tabla de destinos por
> rol.
>
> Aun asi, deben mantenerse alineados. Si `roleRouting.js` reconoce un rol
> administrativo que `AppRoutes.jsx` no permite en `/coordinador`, el usuario
> podria ser enviado a `/coordinador` y luego recibir acceso denegado. Si
> `AppRoutes.jsx` permite un rol que `roleRouting.js` no reconoce, ese usuario
> podria acceder manualmente a su ruta, pero las redirecciones automaticas no lo
> mandarian a su panel correcto.

## Flujo de proteccion de rutas

`PrivateRoute` aplica el siguiente flujo:

1. Obtiene `isAuthenticated`, `loading` y `user` desde `useAuth`.
2. Si `loading` es verdadero, renderiza un estado `Cargando...`.
3. Si el usuario no esta autenticado, redirige a `/login`.
4. Lee roles desde `user?.roles || []`.
5. Si `allowedRoles` esta vacio, permite renderizar la vista.
6. Si hay roles permitidos, verifica que al menos uno exista en `user.roles`.
7. Si no hay coincidencia, muestra `AccessDenied`.
8. Si hay coincidencia, renderiza `children`.

La comparacion de roles se realiza por coincidencia exacta de string.

## Redireccion por rol

`roleRouting.js` no autoriza vistas. Solo calcula una ruta destino para
navegacion automatica.

Se usa cuando la interfaz necesita enviar al usuario a su panel esperado, por
ejemplo despues de resolver su sesion o cuando `AccessDenied` muestra el boton
"Ir a mi panel".

La resolucion actual es:

| Condicion | Destino |
| --- | --- |
| Incluye `Estudiante` | `/dashboard` |
| Incluye algun rol administrativo reconocido por `roleRouting.js` | `/coordinador` |
| Incluye `Supervisor de practica` | `/supervisor` |
| Ninguna coincidencia | `/landing` |

Si un usuario tiene multiples roles, se aplica el primer caso que coincida en el
orden anterior.

> [!WARNING]
> Que `roleRouting.js` devuelva una ruta no significa que el usuario tenga permiso
> para abrirla. La ruta devuelta igualmente pasa por `PrivateRoute` si es una
> ruta protegida.

## Limites actuales

- La autorizacion frontend depende de strings de rol retornados por `/auth/me`.
- La comparacion de roles distingue mayusculas, minusculas, tildes y espacios.
- Los grupos de roles estan definidos localmente y deben mantenerse alineados con
  backend.
- `AppRoutes.jsx` y `roleRouting.js` no comparten una fuente unica de roles.
- No existe ruta comodin para URLs no declaradas.
- La denegacion visual no impide llamadas manuales a endpoints protegidos.
