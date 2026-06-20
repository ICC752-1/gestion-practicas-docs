# Casos de Prueba - Auth

## Alcance

Estos casos documentan las pruebas de valor del módulo `auth`. El foco está en autenticación local, emisión y rotación de tokens, revocación de refresh tokens, resolución del usuario autenticado, autorización por roles, normalización de datos de identidad, Google OAuth y administración básica de roles.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos de seguridad funcional o contratos transversales del backend.

## End-to-End

### CU-E2E-AU-01: Login local, consulta de usuario actual y logout

- Tipo de prueba: End-to-end
- Dominio: Auth
- Contexto: Este es el flujo básico de sesión para cualquier usuario del sistema.
- Objetivo: Validar credenciales, emisión de tokens, acceso a endpoint protegido y cierre de sesión.
- Escenario: Usuario inicia sesión con email/password, consulta `/auth/me` y ejecuta logout con refresh token.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: Login entrega tokens, `/auth/me` responde usuario correcto y logout revoca sesión.
- Valor de negocio: Da confianza sobre el flujo de autenticación principal.
- Pruebas automatizadas:
  - Pendiente de implementación.

### CU-E2E-AU-02: Login y rotación de refresh token

- Tipo de prueba: End-to-end
- Dominio: Auth
- Contexto: La renovación de sesión debe rotar refresh tokens y evitar reutilización del anterior.
- Objetivo: Validar el flujo completo de refresh rotation atravesando endpoint y persistencia.
- Escenario: Usuario inicia sesión, usa refresh token para renovar, intenta reutilizar el refresh anterior.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: El primer refresh emite nueva sesión; el refresh anterior queda revocado y no puede reutilizarse.
- Valor de negocio: Protege sesiones contra replay de refresh tokens.
- Pruebas automatizadas:
  - Pendiente de implementación.

### CU-E2E-AU-03: Google OAuth completo en entorno controlado

- Tipo de prueba: End-to-end
- Dominio: Auth
- Contexto: El login federado atraviesa navegador, Google, backend y frontend.
- Objetivo: Validar el flujo completo de Google OAuth con proveedor o doble controlado.
- Escenario: Usuario institucional inicia OAuth, se valida state, se procesa callback y se emiten tokens internos.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: Usuario autorizado recibe sesión interna; cuentas no autorizadas son rechazadas.
- Valor de negocio: Da confianza sobre el acceso federado institucional.
- Pruebas automatizadas:
  - Pendiente de implementación.
