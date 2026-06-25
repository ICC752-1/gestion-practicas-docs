<h1 align="center"><em>Casos de Prueba Frontend</em></h1>

## Objetivo

Este documento define un conjunto minimo de pruebas frontend para proteger el
comportamiento transversal mas importante sin convertir la suite en pruebas
fragiles de estilos o detalles visuales.

El foco inicial es cubrir navegacion por rol, proteccion de rutas, manejo de
sesion y el flujo principal de inscripcion de practica.

## Criterios de valor

Una prueba frontend debe existir si protege al menos uno de estos puntos:

- Redireccion correcta segun rol autenticado.
- Bloqueo visual de rutas privadas para usuarios sin sesion o sin rol permitido.
- Manejo consistente de tokens y sesiones invalidas.
- Restauracion de sesion despues de recargar la aplicacion.
- Flujo critico del estudiante para crear una solicitud de practica.

Se deben evitar snapshots, pruebas de clases CSS, textos decorativos o asserts
que dependan de layout salvo que afecten directamente el flujo funcional.

## Resumen de cobertura propuesta

| Area | Tipo | Casos | Archivo objetivo |
| --- | --- | ---: | --- |
| Redireccion por rol | Unitaria | 4 | `src/services/roleRouting.test.js` |
| Rutas protegidas | Integracion ligera | 3 | `src/components/PrivateRoute.test.jsx` |
| Cliente API | Unitaria | 2 | `src/services/api.test.js` |
| Autenticacion | Integracion ligera | 2 | `src/context/AuthContext.test.jsx` |
| Inscripcion de practica | Integracion ligera | 3 | `src/pages/Registration/RegistrationPage.test.jsx` |
| **Total** |  | **14** |  |

## Casos documentados

| ID | Tipo | Area | Caso | Resultado esperado |
| --- | --- | --- | --- | --- |
| FE-U-RR-01 | Unitaria | Redireccion por rol | Estudiante accede a su panel | `getRedirectPathForRoles(["Estudiante"])` devuelve `/dashboard`. |
| FE-U-RR-02 | Unitaria | Redireccion por rol | Rol administrativo accede a su panel especifico | `Encargado de practica` devuelve `/encargado`, `Director de carrera` devuelve `/director` y `Secretaria de Carrera` devuelve `/secretaria`. |
| FE-U-RR-03 | Unitaria | Redireccion por rol | Supervisor accede a su panel | `Supervisor de practica` devuelve `/supervisor`. |
| FE-U-RR-04 | Unitaria | Redireccion por rol | Rol desconocido queda fuera de paneles privados | Un rol no reconocido o una lista vacia devuelve `/landing`. |
| FE-I-PR-01 | Integracion ligera | Rutas protegidas | Usuario no autenticado no ve contenido privado | `PrivateRoute` redirige a `/login` y no renderiza `children`. |
| FE-I-PR-02 | Integracion ligera | Rutas protegidas | Usuario con rol permitido ve contenido protegido | `PrivateRoute` renderiza `children` cuando `user.roles` contiene un rol incluido en `allowedRoles`. |
| FE-I-PR-03 | Integracion ligera | Rutas protegidas | Usuario autenticado sin rol permitido ve acceso denegado | `PrivateRoute` muestra la vista de acceso denegado y no renderiza `children`. |
| FE-U-API-01 | Unitaria | Cliente API | Peticion protegida adjunta access token | Si existe `localStorage.token`, el interceptor agrega `Authorization: Bearer <token>`. |
| FE-U-API-02 | Unitaria | Cliente API | Sesion invalida se limpia ante `401` no recuperable | Si una respuesta `401` no puede renovarse, se eliminan `token` y `refresh_token`, y se redirige a `/login`. |
| FE-I-AUTH-01 | Integracion ligera | Autenticacion | Sesion valida se restaura al cargar | Con `localStorage.token` y `/auth/me` exitoso, `AuthContext` deja el usuario autenticado. |
| FE-I-AUTH-02 | Integracion ligera | Autenticacion | Sesion invalida se descarta al cargar | Con `localStorage.token` y `/auth/me` fallando, `AuthContext` elimina tokens y deja la sesion no autenticada. |
| FE-I-REG-01 | Integracion ligera | Inscripcion de practica | Formulario inicia en el primer paso | `RegistrationPage` muestra el paso inicial de informacion personal y no muestra pantalla de exito. |
| FE-I-REG-02 | Integracion ligera | Inscripcion de practica | Envio exitoso crea solicitud | Al completar el flujo, se llama a `internshipService.createInternship` con el payload esperado y se muestra la pantalla de exito. |
| FE-I-REG-03 | Integracion ligera | Inscripcion de practica | Error de envio no se trata como exito | Si `createInternship` falla, se muestra un mensaje de error y no se renderiza `RegistrationSuccess`. |

## Alcance inicial de automatizacion

Estas pruebas deben implementarse con una configuracion minima de `Vitest`,
`React Testing Library` y `jsdom` cuando se agregue soporte de test al frontend.

La suite inicial no debe cubrir:

- Estilos, clases Tailwind o layout visual.
- Snapshots de paginas completas.
- Contratos HTTP del backend fuera de mocks controlados.
- Todos los formularios o componentes reutilizables.

## Notas de implementacion

- Mockear `useAuth` para probar `PrivateRoute` sin depender del flujo completo de login.
- Mockear `authService.getMe` para probar restauracion de sesion en `AuthContext`.
- Mockear `internshipService.createInternship` para probar `RegistrationPage` sin backend real.
- Mantener las pruebas enfocadas en comportamiento observable: render permitido,
  bloqueo, redireccion, llamada de servicio y mensaje de error.
