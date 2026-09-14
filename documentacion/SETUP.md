# Setup de entorno de desarrollo

## Tabla de contenidos

- [Prerequisitos](#prerequisitos)
- [Pasos](#pasos)
- [Verificar que todo funciona](#verificar-que-todo-funciona)
- [Troubleshooting](#troubleshooting)

## Prerequisitos

- [Node.js](https://nodejs.org/) 20+
- [Docker](https://www.docker.com/) (para PostgreSQL)
- npm (viene con Node)

## Pasos

Todos los comandos se corren desde `playwright_vortex/`.

1. **PostgreSQL**
   ```bash
   docker compose -f docker-compose.dev.yml up postgres -d
   ```
   Crea el contenedor `acta-postgres` en el puerto `5432`, con usuario/contraseña/DB `acta`/`acta`/`acta` (definidos en `docker-compose.dev.yml`, servicio `postgres`).

2. **Dependencias**
   ```bash
   npm install
   ```
   El script `postinstall` (`scripts/install-playwright-browsers.js`) descarga los navegadores de Playwright automáticamente.

3. **Variables de entorno**
   ```bash
   cp .env.example .env
   ```
   Ver el detalle de cada variable en [`documentacion.md`](./documentacion.md#variables-de-entorno).

4. **Migraciones**
   ```bash
   npx prisma migrate dev
   ```
   Pide un nombre de migración (ej. `init`). Alternativa sin generar archivos de migración: `npx prisma db push`.

5. **Seed**
   ```bash
   npx prisma db seed
   ```
   Crea el usuario `admin@admin.com` / contraseña de `SEED_ADMIN_PASSWORD` / rol `superadmin`.

6. **Levantar la app**
   ```bash
   npm run dev
   ```

## Verificar que todo funciona

- Abrir `http://localhost:3000` e iniciar sesión con las credenciales del seed.
- Inspeccionar la base de datos: `npx prisma studio`.
- Confirmar que los 3 procesos (`web`, `worker`, `recorder`) arrancaron sin error en la salida de `npm run dev` (cada uno tiene su propio prefijo de color/nombre gracias a `concurrently`).

## Troubleshooting

Notas de verificación detectadas al comparar `.env.example` contra el `.env` real y la configuración de Playwright — no son bugs confirmados, son puntos a revisar si algo falla:

- **El modo grabador no conecta / falla al iniciar sesión desde Next.js**: revisar que `RECORDER_INTERNAL_URL` esté definida en `.env`. En el `.env` local existente al momento de escribir esto, esa variable no estaba presente aunque sí figura en `.env.example` (apunta a `http://localhost:3100` en dev, o al nombre del servicio Docker `http://recorder:3100` en `docker-compose.dev.yml`).
- **`npx playwright test` no encuentra los specs de `e2e/`**: `playwright.config.ts` define `testDir: './runtime/ejecuciones'`, que es la carpeta que usa el motor de ejecución en tiempo de ejecución — no `e2e/`. Hay que invocar los tests E2E apuntando explícitamente a la carpeta: `npx playwright test e2e/`.
- **El worker falla al lanzar Playwright fuera de Docker**: el `Dockerfile` usa la imagen `mcr.microsoft.com/playwright:v1.62.1-noble`, que trae Chromium/Firefox/WebKit y sus dependencias de sistema preinstaladas. Corriendo el worker directamente en el host (sin Docker), esas dependencias deben estar instaladas manualmente o vía el `postinstall` de Playwright.
- **`RECORDER_INTERNAL_SECRET` vacío**: `.env.example` trae ese valor como cadena vacía (`""`). Debe generarse (`openssl rand -hex 32`) y coincidir exactamente entre el proceso `web` y el proceso `recorder` — si se genera solo en uno, las llamadas a `/internal/start` fallan.
