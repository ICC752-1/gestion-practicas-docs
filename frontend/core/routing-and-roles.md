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
- Exponer vistas publicas de autenticacion, informacion y evaluacion publica del supervisor.
- Mantener rutas legacy que hoy redirigen a paneles o vistas nuevas.
- Proteger vistas funcionales mediante sesion activa.
- Restringir vistas protegidas segun roles del usuario autenticado.
- Mostrar una pantalla de acceso denegado cuando el rol no coincide.
- Sugerir un panel de destino segun los roles del usuario.
- Mantener la tabla de rutas en un punto central del frontend.

## Ambito y responsabilidades

La capa de rutas define que vista debe renderizarse para cada URL y aplica
restricciones visuales basadas en autenticacion y roles.

#### Responsabilidades principales

- Declarar rutas publicas, protegidas y redirects de compatibilidad.
- Asociar cada ruta con su pagina React correspondiente.
- Definir grupos de roles usados por `PrivateRoute`.
- Redirigir rutas historicas hacia flujos vigentes.
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
| `src/routes/AppRoutes.jsx` | Declara rutas, redirecciones, paneles por rol y compatibilidad legacy. |
| `src/components/PrivateRoute.jsx` | Evalua sesion, carga y roles antes de renderizar una vista protegida. |
| `src/services/roleRouting.js` | Resuelve el panel local recomendado segun los roles del usuario. |
| `src/context/AuthContext.jsx` | Provee `isAuthenticated`, `loading` y `user.roles`. |
| `src/context/useAuth.js` | Expone el estado de autenticacion a `PrivateRoute`. |

## Rutas publicas

Las rutas publicas actuales no usan `PrivateRoute`.

| Ruta | Vista | Proposito |
| --- | --- | --- |
| `/` | `Navigate` hacia `/landing` | Redireccion inicial. |
| `/landing` | `LandingPage` | Pagina publica de entrada. |
| `/login` | `Login` | Inicio de sesion local u OAuth. |
| `/activar-cuenta` | `ActivateAccountPage` | Primer acceso y definicion de password. |
| `/auth/callback` | `AuthCallbackPage` | Procesamiento del retorno OAuth. |
| `/faq` | `FAQPage` | Preguntas frecuentes. |
| `/requisitos` | `RequirementsPage` | Informacion academica previa a la solicitud. |
| `/supervisor/evaluacion/:token` | `SupervisorEvaluationPage` | Formulario publico de evaluacion por invitacion. |

## Rutas protegidas

Las rutas protegidas se envuelven con `PrivateRoute` o redirigen a vistas vigentes.

| Ruta | Vista | Roles permitidos |
| --- | --- | --- |
| `/dashboard/*` | `StudentDashboardPage` | `Estudiante` |
| `/encargado` | `CoordinatorDashboardPage` | `Encargado de practica` |
| `/encargado/agenda` | `CoordinatorDashboardPage` | `Encargado de practica` |
| `/encargado/cartas-presentacion` | `CoordinatorDashboardPage` | `Encargado de practica` |
| `/encargado/induccion` | `CoordinatorDashboardPage` | `Encargado de practica` |
| `/encargado/estudiantes` | `CoordinatorDashboardPage` | `Encargado de practica` |
| `/encargado/practica/:id` | `PracticeDetailPage` | `Encargado de practica`, `Director de carrera` |
| `/director` | `CoordinatorDashboardPage` | `Director de carrera` |
| `/director/agenda` | `CoordinatorDashboardPage` | `Director de carrera` |
| `/director/cartas-presentacion` | `CoordinatorDashboardPage` | `Director de carrera` |
| `/director/induccion` | `CoordinatorDashboardPage` | `Director de carrera` |
| `/director/estudiantes` | `CoordinatorDashboardPage` | `Director de carrera` |
| `/director/practica/:id` | `PracticeDetailPage` | `Encargado de practica`, `Director de carrera` |
| `/induccion/admin` | `InductionAdminPage` | `Encargado de practica`, `Director de carrera` |
| `/reportes/admin` | `FicaDashboardPage` | `Encargado de practica`, `Director de carrera`, `FICA` |
| `/estudiantes/admin` | `StudentAccountsPage` | `Encargado de practica`, `Director de carrera` |
| `/secretaria` | `SecretaryDashboardPage` | `Secretaria de Carrera` |
| `/seguimiento/:internshipId` | Redirect hacia `/dashboard/seguimiento/:internshipId` | `Estudiante` |
| `/supervisor` | `SupervisorPage` | `Supervisor de practica` |
| `/autoevaluacion/:internshipId` | `SelfEvaluationPage` | `Estudiante` |
| `/entrevistas` | `InterviewSchedulingPage` | `Estudiante`, `Encargado de practica`, `Director de carrera` |
| `/cartas-presentacion` | `PresentationLettersPage` | `Estudiante`, `Encargado de practica`, `Director de carrera` |
| `/fica/*` | `FicaDashboardPage` | `FICA` |
| `/superadmin/usuarios` | `SuperadminDashboardPage` | `Superadmin` |
| `/superadmin/auditoria` | `SuperadminDashboardPage` | `Superadmin` |

> [!WARNING]
> Varias rutas historicas siguen existiendo solo como compatibilidad y ya no son
> la vista canonica del flujo. Ejemplos: `/inscripcion`, `/coordinador`,
> `/coordinador/practica/:id`, `/practicas/nueva/preinscripcion`,
> `/practicas/nueva/formulario`, `/seguimiento`, `/autoevaluacion`, `/fica` y
> `/superadmin`.

En particular, `/fica` ya no es la vista final: hoy redirige a
`/fica/resumen`, y el panel FICA organiza su navegacion interna en rutas como
`/fica/resumen`, `/fica/cumplimiento`, `/fica/distribuciones`,
`/fica/indicadores` y `/fica/organizaciones`.

## Roles frontend

`AppRoutes.jsx` define grupos locales de roles para proteger rutas.

```js
const STUDENT_ROLES = ['Estudiante']
const DECISION_ADMIN_ROLES = ['Encargado de practica', 'Director de carrera']
const REPORT_ROLES = ['Encargado de practica', 'Director de carrera', 'FICA']
const PRACTICE_MANAGER_ROLES = ['Encargado de practica']
const CAREER_DIRECTOR_ROLES = ['Director de carrera']
const SECRETARY_ROLES = ['Secretaria de Carrera']
const SUPERVISOR_ROLES = ['Supervisor de practica']
const FICA_ROLES = ['FICA']
const SUPERADMIN_ROLES = ['Superadmin']
```

Estos grupos se usan en `allowedRoles` dentro de `PrivateRoute`.

Su responsabilidad es responder esta pregunta:

**"Este usuario puede entrar a esta ruta concreta?"**

Por ejemplo, para `/encargado/practica/:id`:

```jsx
<PrivateRoute allowedRoles={DECISION_ADMIN_ROLES}>
    <PracticeDetailPage />
</PrivateRoute>
```

Si el usuario no tiene alguno de los roles permitidos, la ruta no se renderiza y
se muestra la pantalla de acceso denegado.

`roleRouting.js`, en cambio, no protege rutas. Su responsabilidad es resolver a
que panel deberia enviarse un usuario segun sus roles.

Su pregunta es:

**"Si necesito mandar a este usuario a su panel principal, cual ruta conviene?"**

Actualmente contempla estos destinos principales:

```js
Superadmin -> /superadmin/usuarios
FICA -> /fica
Estudiante -> /dashboard
Encargado de practica -> /encargado
Director de carrera -> /director
Secretaria de Carrera -> /secretaria
Supervisor de practica -> /supervisor
```

Para FICA, ese destino base pasa luego por el redirect de rutas y termina en la
vista canónica `/fica/resumen`.

`getRedirectPathForRoles` devuelve:

| Rol detectado | Destino |
| --- | --- |
| `Superadmin` | `/superadmin/usuarios` |
| `FICA` | `/fica` |
| `Estudiante` | `/dashboard` |
| `Encargado de practica` | `/encargado` |
| `Director de carrera` | `/director` |
| `Secretaria de Carrera` | `/secretaria` |
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
| Incluye `Superadmin` | `/superadmin/usuarios` |
| Incluye `FICA` | `/fica` |
| Incluye `Estudiante` | `/dashboard` |
| Incluye `Encargado de practica` | `/encargado` |
| Incluye `Director de carrera` | `/director` |
| Incluye `Secretaria de Carrera` | `/secretaria` |
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
- Existen redirects legacy que deben mantenerse alineados con las rutas canonicas.
- La denegacion visual no impide llamadas manuales a endpoints protegidos.
