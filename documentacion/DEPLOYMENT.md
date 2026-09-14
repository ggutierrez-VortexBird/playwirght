# Deployment

## Tabla de contenidos

- [Estado actual](#estado-actual)
- [Docker Compose de desarrollo](#docker-compose-de-desarrollo)
- [Imagen base](#imagen-base)
- [Limitación conocida del servicio recorder](#limitación-conocida-del-servicio-recorder)

## Estado actual

El repositorio solo incluye un `docker-compose.dev.yml` orientado a desarrollo local. No hay Dockerfile ni pipeline de CI/CD de producción en `playwright_vortex/` al momento de escribir esto — no se documenta un despliegue de producción porque no existe en el código.

## Docker Compose de desarrollo

```bash
docker compose -f docker-compose.dev.yml up
```

Servicios definidos (`docker-compose.dev.yml`):

| Servicio | Contenedor | Comando | Notas |
|---|---|---|---|
| `postgres` | `acta-postgres` | imagen `postgres:16-alpine` | Puerto `5432`, healthcheck `pg_isready` |
| `app` | `acta-dev` | `npx prisma migrate deploy && node --import tsx prisma/seed.ts && npm run dev:web` | Puerto `3000`. Usa `migrate deploy` (no `migrate dev`) porque en contenedor no hay a quién preguntarle nada interactivamente |
| `worker` | `acta-worker` | `node --import tsx scripts/worker.ts` | Motor de ejecución; sin este servicio ningún caso se ejecuta dentro de Docker aunque `app` y `postgres` estén sanos |
| `recorder` | `acta-recorder` | `node --import tsx scripts/recorder-worker.ts` | Puerto `3100` (HTTP interno + WS) |

Variables de entorno requeridas por el compose (sin default, fallan si no están definidas): `SESSION_SECRET`, `SEED_ADMIN_PASSWORD` (en `app` y `worker`); `SESSION_SECRET` también en `recorder`.

Variables con default si no se definen: `RECORDER_INTERNAL_SECRET` (`dev-recorder-secret-change-me-in-prod-32chars`), `RECORDER_MAX_SESSIONS` (`3`), `RECORDER_HEARTBEAT_TIMEOUT_MS` (`600000`).

Volúmenes nombrados: `playwright-scripts` (compartido por `app` y `worker`), `artefactos` (evidencias de ejecución, compartido por `app` y `worker`), `postgres-data`.

## Imagen base

`Dockerfile` usa `mcr.microsoft.com/playwright:v1.62.1-noble` como base — trae Node, Chromium/Firefox/WebKit y sus dependencias de sistema preinstaladas, necesarias para que el worker y el recorder puedan lanzar navegadores. El tag de la imagen debe coincidir con la versión de `@playwright/test` en `package.json` (`^1.62.1` al momento de escribir esto); si se actualiza esa dependencia, hay que actualizar también el tag del Dockerfile (advertencia dejada como comentario en el propio archivo).

Los tres servicios de Node (`app`, `worker`, `recorder`) comparten el mismo `Dockerfile` y solo cambian el `command:` en `docker-compose.dev.yml`.

## Limitación conocida del servicio recorder

Documentada en el propio `README.md` de `playwright_vortex/`: el servicio `recorder` lanza un navegador headed (`headless: false`, en `scripts/codegen-runner.ts`) para el modo grabador. Eso requiere una pantalla — típicamente Xvfb dentro de un contenedor Linux — y el compose actual no la configura ni fue probado con una grabación real de punta a punta en Docker. Los servicios `app` y `worker`, que corren siempre headless, no tienen esta limitación.
