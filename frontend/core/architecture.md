<h1 align="center"><em>Arquitectura frontend</em></h1>

> [!NOTE]
> Esta documentacion tecnica describe la arquitectura actual del frontend desde
> una perspectiva interna. Su objetivo es explicar como esta organizado, que
> responsabilidades tiene cada capa y que debe saber alguien antes de
> modificarlo.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Ambito y responsabilidades](#ambito-y-responsabilidades)
- [Estructura interna](#estructura-interna)
- [Flujo general de ejecucion](#flujo-general-de-ejecucion)
- [Capas principales](#capas-principales)
- [Integracion con backend](#integracion-con-backend)

## Resumen operativo

El frontend es una aplicacion **React + Vite** que implementa la interfaz web del
Sistema de Gestion de Practicas. Usa `react-router-dom` para navegacion, Axios
para comunicacion HTTP con el backend y contextos React para estado transversal
de autenticacion y notificaciones visuales.

**Permite:**

- Exponer rutas publicas para landing, login, callback OAuth y FAQ.
- Proteger vistas segun sesion activa y roles del usuario autenticado.
- Consumir endpoints del backend mediante servicios centralizados.
- Mantener sesion local con access token y refresh token.
- Renderizar flujos funcionales de inscripcion, seguimiento, documentos,
  dashboards, evaluaciones, notificaciones y entrevistas.

## Ambito y responsabilidades

El frontend centraliza la experiencia de usuario del sistema. Su responsabilidad
principal es presentar vistas, orquestar interacciones de usuario y consumir el
contrato HTTP expuesto por el backend.

#### Responsabilidades principales

- Definir rutas publicas y privadas de la aplicacion.
- Resolver autorizacion visual mediante roles recibidos desde `/auth/me`.
- Centralizar llamadas HTTP en servicios bajo `src/services`.
- Mantener el estado transversal de autenticacion en `AuthContext`.
- Reutilizar logica de consulta o estado en hooks bajo `src/hooks`.
- Separar vistas de pagina en `src/pages` y piezas reutilizables en
  `src/components`.
- Configurar la URL base del backend mediante `VITE_API_URL`.

#### Fuera de alcance

- Validar permisos reales de negocio.
- Persistir datos funcionales del sistema.
- Definir contratos HTTP formales.
- Aplicar reglas de integridad o autorizacion del backend.
- Sustituir OpenAPI, Swagger o la documentacion tecnica backend.

> [!IMPORTANT]
> Las restricciones de rutas y roles en frontend son una barrera de interfaz, no
> una garantia de seguridad. La autorizacion efectiva debe mantenerse en el
> backend.

## Estructura interna

| Area | Ruta | Responsabilidad |
| --- | --- | --- |
| Entrada React | `src/main.jsx` | Monta la aplicacion en el DOM. |
| Aplicacion raiz | `src/App.jsx` | Ensambla proveedores y rutas principales. |
| Rutas | `src/routes/AppRoutes.jsx` | Define rutas publicas, protegidas y redirecciones. |
| Paginas | `src/pages/...` | Implementan vistas completas asociadas a rutas. |
| Componentes | `src/components/...` | Agrupan UI reutilizable o especifica de un flujo. |
| Servicios API | `src/services/...` | Encapsulan llamadas HTTP al backend. |
| Cliente HTTP | `src/services/api.js` | Configura Axios, tokens, refresh e interceptores. |
| Contextos | `src/context/...` | Mantienen estado transversal de autenticacion y toasts. |
| Hooks | `src/hooks/...` | Reutilizan logica de consulta, carga y estado. |
| Constantes | `src/constants/...` | Centralizan valores compartidos por flujos concretos. |
| Estilos globales | `src/index.css`, `src/App.css` | Definen estilos base y utilidades visuales. |

## Flujo general de ejecucion

1. Vite carga `src/main.jsx`.
2. React monta la aplicacion raiz.
3. `App.jsx` envuelve la aplicacion con los proveedores necesarios.
4. `AppRoutes.jsx` define la ruta a renderizar segun la URL actual.
5. Si una ruta usa `PrivateRoute`, se consulta el estado de autenticacion desde
   `AuthContext`.
6. Las paginas consumen servicios o hooks para obtener datos del backend.
7. Los servicios usan el cliente Axios centralizado en `src/services/api.js`.
8. Los interceptores anaden `Authorization`, renuevan tokens cuando corresponde
   y redirigen a login ante sesiones invalidas.

## Capas principales

#### Rutas

Las rutas estan centralizadas en `src/routes/AppRoutes.jsx`. Este archivo define
la navegacion publica y privada de la aplicacion.

Las rutas publicas actuales son:

| Ruta | Vista |
| --- | --- |
| `/landing` | Landing publica |
| `/login` | Login |
| `/auth/callback` | Callback OAuth |
| `/faq` | Preguntas frecuentes |

Las rutas privadas se envuelven con `PrivateRoute` y declaran los roles
permitidos mediante `allowedRoles`.

#### Paginas

Las paginas viven en `src/pages`. Cada pagina representa una vista asociada a una
ruta o flujo funcional principal.

Ejemplos actuales:

- `pages/Registration`: preinscripcion e inscripcion de practicas.
- `pages/StudentDashboard`: panel del estudiante.
- `pages/CoordinatorDashboard`: dashboard y detalle administrativo.
- `pages/Seguimiento`: seguimiento de practicas.
- `pages/Supervisor`: vista del supervisor externo.
- `pages/SelfEvaluation`: autoevaluacion del estudiante.
- `pages/InterviewScheduling`: entrevistas o presentaciones.

#### Componentes

Los componentes viven en `src/components`. Algunos son transversales, como
`PrivateRoute`, headers, footer o notificaciones. Otros estan agrupados por flujo
funcional, por ejemplo `Registration`, `StudentDashboard`, `Evaluation` o
`InterviewScheduling`.

Los componentes no deberian duplicar reglas de negocio del backend. Cuando una
decision dependa de permisos, estado de practica o disponibilidad, la fuente de
verdad debe ser la respuesta del backend.

#### Servicios

Los servicios bajo `src/services` encapsulan llamadas HTTP y devuelven
`response.data` a las paginas o hooks consumidores.

Servicios actuales relevantes:

| Servicio | Responsabilidad |
| --- | --- |
| `api.js` | Cliente Axios compartido e interceptores. |
| `authService.js` | Login, logout, usuario actual y URL OAuth. |
| `internshipService.js` | Inscripcion, seguimiento y acciones sobre practicas. |
| `coordinatorService.js` | Consultas y acciones administrativas. |
| `documentService.js` | Tipos, subida, descarga, revision y eliminacion documental. |
| `notificationService.js` | Consulta de notificaciones. |
| `roleRouting.js` | Redireccion local segun roles del usuario. |
| `oauthErrors.js` | Mapeo de errores del callback OAuth. |

#### Contextos

El estado transversal se maneja con contextos React.

| Contexto | Responsabilidad |
| --- | --- |
| `AuthContext` | Usuario autenticado, token, login, logout y callback OAuth. |
| `ToastContext` | Notificaciones visuales internas de la interfaz. |

`AuthContext` restaura sesion llamando a `/auth/me` cuando existe un access token
local. Si la restauracion falla, limpia tokens locales y deja la sesion sin
usuario autenticado.

#### Hooks

Los hooks bajo `src/hooks` encapsulan logica reutilizable de consulta y estado.

Ejemplos actuales:

- `usePractice`: carga detalle administrativo de una practica.
- `useNotifications`: consulta notificaciones y mantiene estado local de ultima
  notificacion vista.
- `useCoordinatorDashboard`: obtiene datos del dashboard administrativo.

## Integracion con backend

La integracion HTTP se realiza mediante Axios desde `src/services/api.js`.

La URL base se obtiene desde:

```env
VITE_API_URL=http://localhost:8000
```

En despliegue Docker, el build puede usar:

```env
VITE_API_URL=/api
```

El frontend consume endpoints del backend, pero no define su contrato formal. Los
contratos HTTP definitivos pertenecen al backend y se consultan en OpenAPI,
Swagger o la documentacion tecnica backend correspondiente.
