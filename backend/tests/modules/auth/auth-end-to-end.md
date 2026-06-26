# Casos de Prueba - Auth

## Alcance

Estos casos documentan las pruebas end-to-end de mayor valor del módulo `auth`. El foco está en el flujo principal de sesión local y en la rotación de refresh tokens, sin automatizar OAuth externo real por su costo operacional.

## End-to-End

### CU-E2E-AU-01: Login, usuario actual, refresh y logout

- **Tipo de prueba:** End-to-end
- **Dominio:** Auth
- **Contexto:** Este es el flujo base de sesión para cualquier usuario del sistema.
- **Objetivo:** Validar credenciales, emisión de tokens, endpoint protegido, rotación de refresh token y cierre de sesión.
- **Escenario:** Usuario inicia sesión, consulta `/auth/me`, renueva sesión, intenta reutilizar el refresh anterior, ejecuta logout y verifica que el refresh vigente queda revocado.
- **Resultado esperado:** Login entrega tokens, `/auth/me` responde el usuario correcto, refresh rota la sesión, el refresh anterior no puede reutilizarse y logout revoca la sesión vigente.
- **Valor de negocio:** Protege el contrato de autenticación principal y el control contra replay de refresh tokens.
- **Pruebas automatizadas:**
  - `tests/e2e/test_auth_e2e.py::test_login_me_refresh_and_logout_flow`
