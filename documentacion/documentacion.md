# vorTest

Plataforma de automatización de pruebas con Playwright. Gestiona espacios de trabajo, proyectos, casos de prueba y sus ejecuciones, y genera un acta de evidencia en PDF por cada ejecución.

> Nombre interno del paquete: `vortest` (`playwright_vortex/package.json`). El código fuente vive en la carpeta `playwright_vortex/`.

## Tabla de contenidos

- [Qué hace hoy](#qué-hace-hoy)
- [Instalación](#instalación)
- [Variables de entorno](#variables-de-entorno)
- [Correr el proyecto localmente](#correr-el-proyecto-localmente)
- [Correr los tests](#correr-los-tests)
- [Documentos relacionados](#documentos-relacionados)

## Qué hace hoy

vorTest organiza el trabajo en una jerarquía fija (nombres de los modelos en `prisma/schema.prisma`):

```
Espacio → Proyecto → CasoPrueba → Ejecucion → Acta
```

- **Espacio**: agrupación de proyectos, con usuarios miembros (`UsuarioEspacio`).
- **Proyecto**: pertenece a un espacio; tiene ambiente, credenciales propias y casos de prueba.
- **CasoPrueba**: guarda el código del script Playwright en la columna `script` (texto, no un archivo en disco). Puede venir de subir un `.spec.ts` (`origen = subirScript`) o del modo grabador (`origen = grabador` o `mixto`).
- **Ejecucion**: una corrida de un caso. Tiene estado (`pendiente`, `corriendo`, `paso`, `fallo`, `reparado`, `errorMotor`, `cancelado`), pasos (`PasoEjecucion`), sub-acciones (`PasoSubaccion`) y artefactos (`video`, `captura`, `trace`).
- **Acta**: PDF de evidencia generado 1-a-1 por ejecución, con un consecutivo anual único.

Dos formas de crear un caso de prueba:
1. **Subir script**: se sube un archivo `.spec.ts`/`.test.ts`/`.spec.js`/`.test.js` (`POST /api/casos`, ver `app/api/casos/route.ts`).
2. **Grabador**: se graba una sesión interactuando con el navegador real (Playwright codegen headed) y el `.spec.ts` resultante queda en `SesionGrabacion.specCode` (ver `ARCHITECTURE.md`).

Las ejecuciones corren en un proceso worker separado (`scripts/worker.ts`) que hace polling de la tabla `Ejecucion` cada 5 segundos y corre el script con Playwright en modo headless.

## Instalación

Requisitos: Node.js 20+, Docker (para PostgreSQL), npm.

```bash
cd playwright_vortex

# 1. Levantar PostgreSQL
docker compose -f docker-compose.dev.yml up postgres -d

# 2. Instalar dependencias (el postinstall descarga los navegadores de Playwright)
npm install

# 3. Configurar variables de entorno
cp .env.example .env

# 4. Crear las tablas
npx prisma migrate dev

# 5. Cargar el usuario superadmin
npx prisma db seed
```

El seed crea el usuario inicial: email `admin@admin.com`, contraseña definida en `SEED_ADMIN_PASSWORD` (`.env`), rol `superadmin`.

## Variables de entorno

Definidas en `playwright_vortex/.env.example`:

| Variable | Descripción |
|---|---|
| `DATABASE_URL` | Conexión a PostgreSQL |
| `SESSION_SECRET` | Secreto de sesión (`iron-session`), mínimo 32 caracteres |
| `SEED_ADMIN_PASSWORD` | Contraseña del usuario superadmin creado por el seed |
| `NODE_ENV` | Entorno de ejecución |
| `RECORDER_WS_PORT` | Puerto HTTP + WS del `recorder-worker` (modo grabador) |
| `RECORDER_PUBLIC_URL` | URL pública del WebSocket, expuesta al navegador del frontend |
| `RECORDER_INTERNAL_URL` | URL interna que usa Next.js para llamar a `/internal/start` del recorder |
| `RECORDER_INTERNAL_SECRET` | Secreto compartido entre Next.js y el recorder-worker (`openssl rand -hex 32`) |
| `RECORDER_MAX_SESSIONS` | Máximo de sesiones de grabación concurrentes (default 3) |
| `RECORDER_HEARTBEAT_TIMEOUT_MS` | Timeout de heartbeat en ms (default 600000) |

> Nota de verificación: al momento de escribir esto, el archivo `.env` local no define `RECORDER_INTERNAL_URL` aunque sí está en `.env.example`. Si el modo grabador falla al iniciar sesión desde Next.js, revisar esa variable primero.

## Correr el proyecto localmente

```bash
npm run dev
```

Levanta 3 procesos en paralelo (vía `concurrently`, ver `package.json`):

| Proceso | Comando | Función |
|---|---|---|
| `web` | `next dev` | Next.js en `http://localhost:3000` |
| `worker` | `node --import tsx scripts/worker.ts` | Motor de ejecución de casos |
| `recorder` | `node --import tsx scripts/recorder-worker.ts` | Servidor HTTP+WS del modo grabador |

Cada proceso también se puede levantar por separado: `npm run dev:web`, `npm run dev:worker`, `npm run dev:recorder`.

Otros scripts relevantes (`package.json`):

| Script | Qué hace |
|---|---|
| `npm run db:studio` | Abre Prisma Studio |
| `npm run db:reset` | Resetea la base de datos (borra todo) |
| `npm run cleanup:sesiones` | Limpia `SesionGrabacion` viejas en estado terminal |
| `npm run lint` | `next lint` |
| `npm run typecheck` | `tsc --noEmit` |

## Correr los tests

- **Unitarios / integración**: `npm test` (Jest + Testing Library, `jest.config.mjs`). `npm run test:watch` para modo watch.
- **E2E**: `npx playwright test e2e/` (requiere la app corriendo). Especificar un archivo puntual, ej. `npx playwright test e2e/casos.spec.ts`, también funciona.

> Nota de verificación: `playwright.config.ts` define `testDir: './runtime/ejecuciones'` (la carpeta que usa el motor de ejecución en producción, no la carpeta `e2e/`). Correr `npx playwright test` sin argumentos no recoge los specs de `e2e/`; hay que apuntar explícitamente a `e2e/`.

## Documentos relacionados

- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — arquitectura, modelo de datos, diagrama de flujo.
- [`API.md`](./API.md) — rutas HTTP expuestas en `app/api/`.
- [`SETUP.md`](./SETUP.md) — entorno de desarrollo y troubleshooting.
- [`DEPLOYMENT.md`](./DEPLOYMENT.md) — Docker Compose de desarrollo.
- [`CONTRIBUTING.md`](./CONTRIBUTING.md) — convenciones y flujo de trabajo.
- [`CHANGELOG.md`](./CHANGELOG.md) — historial de versiones.
- [`plan_ejecucion_hus.md`](./plan_ejecucion_hus.md) — proceso interno de ejecución de historias de usuario.
- [`acta-mockups.html`](./acta-mockups.html) — mockups navegables de la UI.
