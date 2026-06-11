# Modelo de despliegue

## Alcance

Este documento resume el modelo de despliegue vigente para el Sistema de Gestión de Prácticas en la VPS asignada al equipo. El objetivo es describir la arquitectura operacional, los repositorios involucrados y el flujo CI/CD a un nivel alto, sin incluir valores secretos ni credenciales.

El despliegue opera sobre PDS-2 y cubre publicación, TLS, DNS público y ejecución contenedorizada para los servicios activos del sistema.

## Resumen de arquitectura

El sistema se publica mediante un único hostname DuckDNS:

```text
https://gestion-practicas-team-b.duckdns.org
```

El tráfico público entra por Nginx del sistema operativo en la VPS. Nginx termina TLS con certificados de Let's Encrypt y redirige las peticiones hacia el contenedor frontend expuesto solo en loopback. Desde ahí, el frontend sirve la aplicación React y actúa como proxy interno para las rutas `/api`.

```text
Internet
  |
  | HTTPS 443
  v
DuckDNS + Nginx del host
  |
  | http://127.0.0.1:3000
  v
frontend: Nginx estático + proxy /api
  |
  | http://backend:8000
  v
backend: FastAPI
  |
  | postgres://db:5432
  v
PostgreSQL
```

## Componentes principales

| Componente | Responsabilidad |
| --- | --- |
| DuckDNS | Provee DNS público para la VPS sin depender de un subdominio institucional. |
| Nginx del host | Recibe tráfico público, fuerza HTTPS y reenvía al frontend local. |
| Let's Encrypt | Emite y renueva el certificado TLS del hostname público. |
| Docker Compose | Orquesta frontend, backend y base de datos en una red Docker común. |
| Frontend | Sirve la aplicación React compilada y reenvía `/api` hacia el backend. |
| Backend | Expone la API FastAPI y se conecta a PostgreSQL. |
| PostgreSQL | Persiste los datos del sistema en un volumen Docker nombrado. |

## Repositorios involucrados

| Repositorio | Rol en el despliegue |
| --- | --- |
| `gestion-practicas-backend` | Define CI y CD para construir la imagen del backend, ejecutar lint/tests y desplegar el servicio backend cuando el workflow esté disponible en `main`. |
| `gestion-practicas-frontend` | Define CI y CD para construir la imagen del frontend, ejecutar lint/build y desplegar el servicio frontend cuando el workflow esté disponible en `main`. |
| `gestion-practicas-deployment` | Define `compose.prod.yml`, ejemplos de entorno, scripts de despliegue y sincronización de archivos base hacia la VPS. |
| `gestion-practicas-docs` | Mantiene documentación técnica y operacional del proyecto. |
| `gestion-practicas-vps-management` | Opera el estado del host: usuarios, llaves, Nginx, DuckDNS, certificados y runbooks. |

## Flujo de tráfico

1. El usuario ingresa a `https://gestion-practicas-team-b.duckdns.org`.
2. DuckDNS resuelve el hostname hacia la IP pública de PDS-2.
3. Nginx del host recibe la conexión HTTPS y usa el certificado de Let's Encrypt.
4. Nginx reenvía el tráfico a `127.0.0.1:3000`, puerto donde escucha el contenedor frontend.
5. El frontend sirve archivos estáticos para la SPA.
6. Las llamadas del navegador a `/api` llegan al Nginx del contenedor frontend.
7. El Nginx del contenedor frontend reenvía esas llamadas a `http://backend:8000`.
8. El backend procesa la petición y consulta PostgreSQL por la red interna de Docker cuando corresponde.

Este diseño evita publicar directamente los puertos del backend y la base de datos. Solo el proxy del host escucha en Internet.

## Modelo CI/CD

El despliegue evita usar GHCR u otro registro de contenedores. En su lugar, los runners construyen imágenes Docker y las transfieren como archivos comprimidos hacia la VPS.

Los workflows de CD de backend y frontend están configurados para ejecutarse desde `main`. En la verificación de Sprint 10.22, esos workflows existen en ramas de desarrollo, pero aún no están presentes en `main`; por lo tanto, el despliegue productivo queda condicionado a aceptar o mergear los PR correspondientes hacia `main`.

Flujo general:

1. GitHub Actions ejecuta verificaciones del repositorio afectado.
2. El runner construye la imagen Docker del servicio.
3. La imagen se exporta con `docker save` y se comprime.
4. El archivo se copia a `/srv/team-b/releases` en la VPS mediante SSH/SCP.
5. El script remoto carga la imagen con `docker load`.
6. La imagen se reetiqueta con una etiqueta estable local `:deploy`.
7. Docker Compose levanta o actualiza solo el servicio correspondiente.
8. El script valida salud básica del servicio y ejecuta limpieza acotada de imágenes antiguas.

## Modelo de imágenes

Las imágenes se construyen con etiquetas inmutables por commit y luego se reetiquetan en la VPS con etiquetas estables usadas por Compose:

```text
gestion-practicas-backend:<commit_sha>
gestion-practicas-backend:deploy
gestion-practicas-frontend:<commit_sha>
gestion-practicas-frontend:deploy
```

Compose consume los tags `:deploy`. Los archivos comprimidos bajo `/srv/team-b/releases` permiten conservar material de reversión durante una ventana corta.

## Seguridad operacional

El despliegue usa el usuario Unix `ci` en la VPS. Este usuario no recibe acceso amplio a Docker por grupo; opera mediante `sudo` con permisos acotados a los comandos necesarios para cargar imágenes, reetiquetarlas, ejecutar Compose y limpiar imágenes antiguas.

Los secretos se administran fuera del repositorio:

- Credenciales SSH en GitHub Actions Secrets.
- Claves de host SSH fijas en `VPS_KNOWN_HOSTS` para evitar depender de `ssh-keyscan` en cada ejecución.
- Variables sensibles de aplicación en `/srv/team-b/app/.env`.
- Token DuckDNS en archivo del host legible solo por root.

Ninguno de esos valores debe documentarse ni versionarse.

## Red y persistencia

La red de aplicación es una red bridge de Docker administrada por Compose. El backend y PostgreSQL solo son accesibles desde esa red interna.

La persistencia principal está en volúmenes Docker nombrados:

- `pg_data`: datos de PostgreSQL.
- `backend_logs`: logs persistidos por el backend.

No se debe ejecutar `docker compose down -v` en producción salvo que se quiera destruir explícitamente la base de datos.

## DNS y TLS

DuckDNS mantiene el hostname público apuntando a PDS-2 mediante un timer de systemd. Let's Encrypt emite el certificado TLS y su renovación queda gestionada por Certbot.

Este punto es relevante para integraciones como Google OAuth, porque los callbacks externos requieren un origen HTTPS con DNS público.

## Limpieza y control de almacenamiento

La limpieza de Docker se mantiene acotada al proyecto. Los scripts de despliegue eliminan imágenes no utilizadas con etiquetas del sistema y eliminan archivos antiguos en `/srv/team-b/releases`.

No se eliminan volúmenes de datos como parte del despliegue normal.

## Estado actual verificado

El despliegue vigente quedó verificado con:

- Frontend público respondiendo HTTP `200`.
- Documentación OpenAPI en `/api/docs` respondiendo HTTP `200`.
- Esquema OpenAPI en `/api/openapi.json` respondiendo HTTP `200`.
- Backend y PostgreSQL en estado saludable dentro de Docker Compose.
- Workflows CI/CD definidos para backend y frontend en ramas de desarrollo, con CD configurado para activarse desde `main`.
- Pendiente: aceptar o mergear los PR hacia `main` para que los workflows de CD backend/frontend queden activos en la rama productiva.
- El repositorio `gestion-practicas-deployment` mantiene `compose.prod.yml` y scripts necesarios para ejecutar el despliegue desde los workflows de aplicación.

## Limitaciones y próximos pasos

- El flujo Google OAuth aún debe cerrarse funcionalmente a nivel de aplicación; la infraestructura ya provee el origen HTTPS público requerido.
- Conviene migrar las GitHub Actions a versiones compatibles con Node.js 24 antes de que GitHub retire Node.js 20 de los runners.
- Las alertas Dependabot y advertencias lint existentes deben tratarse como deuda de seguimiento, no como parte de la operación base del CI/CD.
