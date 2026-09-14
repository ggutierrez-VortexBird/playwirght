# Arquitectura

## Tabla de contenidos

- [Vista general](#vista-general)
- ["vorTest" vs "Acta"](#vortest-vs-acta)
- [Los tres procesos](#los-tres-procesos)
- [Flujo: ejecución de un caso de prueba](#flujo-ejecución-de-un-caso-de-prueba)
- [Flujo: modo grabador](#flujo-modo-grabador)
- [Modelo de datos](#modelo-de-datos)
- [Decisiones de diseño](#decisiones-de-diseño)

## Vista general

vorTest es un monolito Next.js (App Router) con dos procesos Node adicionales de larga duración que comparten la misma base de código y base de datos PostgreSQL vía Prisma:

- **web** — la app Next.js (páginas + `app/api/*` route handlers + Server Actions en `lib/*/actions.ts`).
- **worker** (`scripts/worker.ts`) — motor de ejecución: hace polling de `Ejecucion` en estado `pendiente` y corre el script de Playwright asociado.
- **recorder** (`scripts/recorder-worker.ts`) — servidor HTTP + WebSocket standalone para el modo grabador.

No hay cola de mensajes (Redis, etc.): la coordinación entre `web` y `worker` es vía polling sobre la tabla `Ejecucion`.

## "vorTest" vs "Acta"

Son dos cosas distintas y el nombre puede confundir:

- **vorTest** es el nombre del producto/plataforma.
- **Acta** es un concepto de dominio dentro del producto: el modelo Prisma `Acta` representa el PDF de evidencia que se genera al terminar una ejecución (`lib/acta/render-pdf.ts`, `lib/acta/template.ts`). Cada `Ejecucion` tiene como máximo un `Acta` asociado (relación 1-a-1, campo `Ejecucion.acta`).

## Los tres procesos

```mermaid
flowchart LR
    subgraph Dev["npm run dev (concurrently)"]
        Web[web: next dev]
        Worker[worker: scripts/worker.ts]
        Recorder[recorder: scripts/recorder-worker.ts]
    end
    Web -->|Prisma| DB[(PostgreSQL)]
    Worker -->|Prisma, polling 5s| DB
    Recorder -->|Prisma| DB
    Web -. Server Action .-> Recorder
    Browser[Navegador del usuario] -->|HTTP| Web
    Browser -->|WebSocket RECORDER_PUBLIC_URL| Recorder
```

## Flujo: ejecución de un caso de prueba

```mermaid
sequenceDiagram
    participant U as Usuario
    participant W as Next.js (web)
    participant DB as PostgreSQL
    participant WK as worker (scripts/worker.ts)
    participant PW as Playwright (headless)

    U->>W: POST /api/casos/{id}/ejecutar
    W->>DB: Ejecucion.create(estado="pendiente")
    W-->>U: { ejecucionId, redirectTo }
    loop cada 5s
        WK->>DB: buscar Ejecucion en "pendiente"
    end
    WK->>DB: marcar "corriendo"
    WK->>PW: npx playwright test (script del CasoPrueba, reporter custom)
    PW-->>WK: eventos JSON por stdout (env, step, substep, assertion, end)
    WK->>DB: crear PasoEjecucion / PasoSubaccion / Artefacto por evento
    WK->>DB: marcar Ejecucion como "paso" / "fallo" / "reparado" / "errorMotor"
    U->>W: POST /api/ejecuciones/{id}/acta
    W->>DB: generar Acta (PDF) con consecutivo anual
```

Detalle del motor (`lib/worker/*`, `scripts/worker.ts`, `scripts/my-reporter.js`):
- El script de `CasoPrueba.script` se escribe a un archivo temporal y se corre con `npx playwright test`, usando la config de `playwright.config.ts` (`testDir: './runtime/ejecuciones'`).
- El reporter custom (`scripts/my-reporter.js`) emite eventos JSON por stdout que el worker parsea línea a línea para ir creando `PasoEjecucion`/`PasoSubaccion` en tiempo real (streaming de progreso).
- Los artefactos (`video`, `captura`, `trace`) se guardan como filas `Artefacto` con `sha256` y tamaño en bytes.
- Si un paso falla y se recupera con un selector de respaldo, el motor lo marca como auto-reparado (`PasoEjecucion.selfHealed`, estado `reparado`).

## Flujo: modo grabador

```mermaid
sequenceDiagram
    participant U as Usuario
    participant W as Next.js (web)
    participant R as recorder-worker (HTTP+WS)
    participant CG as codegen-runner.ts
    participant Browser as Navegador headed (Playwright)

    U->>W: iniciar grabación (Server Action iniciarSesionGrabacion)
    W->>R: POST /internal/start (RECORDER_INTERNAL_SECRET)
    R->>CG: lanza proceso hijo (Playwright codegen)
    CG->>Browser: launch headed
    U->>Browser: interactúa (clicks, formularios)
    Browser-->>CG: eventos de codegen
    CG-->>R: actualiza .spec.ts generado
    U->>W: WebSocket (RECORDER_PUBLIC_URL) — progreso en vivo
    U->>W: guardar sesión
    W->>R: cerrar sesión de grabación
    R->>W: SesionGrabacion.specCode (texto del .spec.ts)
```

El script generado (`SesionGrabacion.specCode`) es la fuente de verdad; los `PasoGrabado` (una fila por paso detectado) se derivan del parseo del spec de forma best-effort y no se usan para regenerar el script (comentario en `prisma/schema.prisma`, modelo `SesionGrabacion`).

## Modelo de datos

16 modelos en `prisma/schema.prisma` (PostgreSQL, UUID v4 como PK, `camelCase`):

```mermaid
erDiagram
    Usuario ||--o{ UsuarioEspacio : pertenece
    Espacio ||--o{ UsuarioEspacio : tiene
    Espacio ||--o{ Proyecto : contiene
    Proyecto ||--o{ CasoPrueba : contiene
    Proyecto ||--o{ Credencial : tiene
    Proyecto ||--o{ SesionGrabacion : tiene
    Usuario ||--o{ CasoPrueba : responsable_de
    CasoPrueba ||--o{ Ejecucion : genera
    CasoPrueba ||--o{ SesionGrabacion : origina
    CasoPrueba ||--o{ PasoGrabado : tiene
    CasoPrueba ||--o{ ParametroGrabacion : tiene
    CasoPrueba ||--o{ JuegoDeDatos : tiene
    Ejecucion ||--o{ PasoEjecucion : tiene
    Ejecucion ||--o{ PasoSubaccion : tiene
    Ejecucion ||--o{ Artefacto : produce
    Ejecucion ||--o| Acta : genera
    PasoEjecucion ||--o{ PasoSubaccion : tiene
    PasoEjecucion ||--o{ Artefacto : tiene
    SesionGrabacion ||--o{ PasoGrabado : registra
    SesionGrabacion ||--o{ ParametroGrabacion : registra
    SesionGrabacion }o--o| Credencial : usa
```

Enums: `EjecucionEstado` (`pendiente, corriendo, paso, fallo, reparado, errorMotor, cancelado`), `PasoEjecucionEstado` (`paso, fallo, reparado`), `ArtefactoTipo` (`video, captura, trace`), `CasoOrigen` (`subirScript, grabador, mixto`).

Reglas de integridad relevantes (comentario del schema): `Restrict` en relaciones hacia filas de evidencia (ej. no se puede borrar un `Proyecto` con `CasoPrueba` asociados), `Cascade` en filas hijas dependientes (ej. borrar una `Ejecucion` borra sus `PasoEjecucion`).

## Decisiones de diseño

- **Sin cola de mensajes**: la comunicación `web` → `worker` es indirecta, vía estado en base de datos (`Ejecucion.estado = "pendiente"`) y polling cada 5 segundos, no un broker dedicado.
- **Script como texto en BD, no archivo en disco**: `CasoPrueba.script` es una columna `TEXT`. El worker lo materializa a un archivo temporal solo al momento de ejecutar.
- **Modo grabador basado en `playwright codegen`, no en canvas + CDP screencast**: el enfoque vigente lanza un navegador real (headed) y usa el motor de grabación nativo de Playwright (`scripts/codegen-runner.ts`) en vez de renderizar un canvas remoto vía Chrome DevTools Protocol. El `.spec.ts` que emite `codegen` es la fuente de verdad (`SesionGrabacion.specCode`).
- **Reporter custom sobre stdout**: en vez de leer archivos de resultado después de terminar, el worker parsea eventos JSON en tiempo real desde el reporter custom de Playwright, lo que permite mostrar el progreso paso a paso mientras la ejecución corre.
