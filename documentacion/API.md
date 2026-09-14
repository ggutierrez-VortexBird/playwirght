# API

Rutas HTTP expuestas por Next.js App Router en `playwright_vortex/app/api/`. Todas devuelven JSON salvo donde se indica lo contrario. La autenticación es por sesión (`iron-session`, cookie gestionada por `lib/auth.ts`); las rutas fuera de `app/api/` están protegidas por `middleware.ts`, pero cada route handler dentro de `app/api/` valida `session.userId` por su cuenta (el matcher del middleware excluye `/api`).

## Tabla de contenidos

- [Convenciones comunes](#convenciones-comunes)
- [Espacios](#espacios)
- [Proyectos](#proyectos)
- [Casos de prueba](#casos-de-prueba)
- [Ejecuciones](#ejecuciones)
- [Actas](#actas)
- [Artefactos](#artefactos)
- [Credenciales](#credenciales)
- [Usuarios y sesión](#usuarios-y-sesión)
- [Grabador](#grabador)

## Convenciones comunes

- Sin sesión activa → `401 { "error": "No autenticado" }` (o `"No autorizado"` según el handler).
- Errores de negocio que llevan `status`/`body` propios (lanzados desde `lib/*/actions.ts`) se devuelven tal cual, ej. `403 { "error": "forbidden", "message": "Se requiere rol de superadmin" }`.
- `404` para recursos no encontrados, normalmente `{ "error": "not_found" }`.

## Espacios

`app/api/espacios/route.ts`, `app/api/espacios/[id]/route.ts`

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/espacios` | Lista espacios (`listEspacios`) |
| POST | `/api/espacios` | Crea un espacio (`createEspacio`) |
| GET | `/api/espacios/[id]` | Detalle de un espacio |
| PUT | `/api/espacios/[id]` | Actualiza un espacio |
| DELETE | `/api/espacios/[id]` | Elimina un espacio (`204` sin body) |

Ejemplo real (`GET /api/espacios`, sin sesión):
```json
{ "error": "No autenticado" }
```
— con status `401`.

## Proyectos

`app/api/proyectos/route.ts`, `app/api/proyectos/[id]/route.ts`, `app/api/proyectos/[id]/credenciales/route.ts`

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/proyectos` | Lista proyectos |
| POST | `/api/proyectos` | Crea un proyecto |
| GET | `/api/proyectos/[id]` | Detalle de un proyecto |
| PUT | `/api/proyectos/[id]` | Actualiza un proyecto |
| DELETE | `/api/proyectos/[id]` | Elimina un proyecto (`204`) |
| GET | `/api/proyectos/[id]/credenciales` | Lista credenciales del proyecto |

## Casos de prueba

`app/api/casos/route.ts`, `app/api/casos/[id]/route.ts`, `app/api/casos/[id]/ejecutar/route.ts`, `app/api/casos/[id]/juego-de-datos/route.ts`, `app/api/casos/[id]/parametros/route.ts`, `app/api/casos/[id]/parametros/[paramId]/route.ts`

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/casos?proyectoId=` | Lista casos, filtro opcional por proyecto |
| POST | `/api/casos` | Crea un caso subiendo un script (`multipart/form-data`) |
| GET | `/api/casos/[id]` | Detalle de un caso |
| PUT | `/api/casos/[id]` | Actualiza un caso |
| DELETE | `/api/casos/[id]` | Elimina un caso |
| POST | `/api/casos/[id]/ejecutar` | Encola una ejecución (ver detalle abajo) |
| GET | `/api/casos/[id]/juego-de-datos` | Lista juegos de datos del caso |
| POST | `/api/casos/[id]/juego-de-datos` | Sube un juego de datos |
| GET | `/api/casos/[id]/parametros` | Lista parámetros del caso |
| PATCH | `/api/casos/[id]/parametros/[paramId]` | Actualiza un parámetro |

**`POST /api/casos`** — request real (`app/api/casos/route.ts`): `multipart/form-data` con campos `scriptFile` (archivo `.spec.ts`/`.test.ts`/`.spec.js`/`.test.js`), `codigo`, `nombre`, `responsableId`, `proyectoId`.

Errores reales:
- `400 { "error": "validation", "message": "Debes seleccionar un archivo de script" }` — falta el archivo.
- `400 { "error": "validation", "message": "El archivo debe ser .spec.ts, .test.ts, .spec.js o .test.js" }` — extensión inválida.
- `201` con el `CasoPrueba` creado.

**`POST /api/casos/[id]/ejecutar`** (`app/api/casos/[id]/ejecutar/route.ts`) — crea una fila `Ejecucion` en estado `pendiente`; el worker la recoge por polling, no la ejecuta esta ruta.

Respuestas reales:
```json
// 200
{ "ejecucionId": "…", "redirectTo": "/ejecuciones/…" }
```
- `401 { "error": "No autenticado" }`
- `404 { "error": "not_found" }` — caso no existe
- `409 { "error": "caso_inactivo" }` — el caso está desactivado

## Ejecuciones

`app/api/ejecuciones/route.ts`, `app/api/ejecuciones/[id]/route.ts`, `app/api/ejecuciones/[id]/acta/route.ts`, `app/api/ejecuciones/[id]/detener/route.ts`

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/api/ejecuciones` | Dispara una ejecución (`dispararEjecucion`) |
| GET | `/api/ejecuciones` | No implementa listado — devuelve `200 { "error": "Use /ejecuciones page for listing" }` (el listado real lo sirve la página como server component) |
| GET | `/api/ejecuciones/[id]` | Detalle completo: estado, pasos, subacciones, artefactos, acta |
| POST | `/api/ejecuciones/[id]/acta` | Genera el Acta (PDF) de la ejecución |
| POST | `/api/ejecuciones/[id]/detener` | Detiene una ejecución en curso |

**`POST /api/ejecuciones`** (`app/api/ejecuciones/route.ts`) — body `{ "casoPruebaId": "…" }`.

Errores reales:
- `400 { "error": "casoPruebaId es requerido" }`
- `409 { "error": "conflict", "message": "Ya existe una ejecución en curso para este caso" }`
- `403 { "error": "forbidden", "message": "Se requiere rol de superadmin" }`
- `404 { "error": "not_found", "message": "Caso de prueba no encontrado" }`
- `500 { "error": "Error interno" }`

**`GET /api/ejecuciones/[id]`** — devuelve el objeto completo con `pasos[].subacciones[]`, cada uno con sus capturas de referencia/actual (`capturaActual`, `capturaReferencia`) y el `acta` asociado si ya se generó. Ver el shape completo en `app/api/ejecuciones/[id]/route.ts`.

## Actas

`app/api/actas/[id]/download/route.ts`

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/actas/[id]/download` | Sirve el PDF del acta (`Content-Type: application/pdf`, `Content-Disposition: inline`) |

Errores reales: `401` sin sesión, `404 { "error": "not_found" }` si el acta no existe, `404 { "error": "file_missing", ... }` si el PDF no está en disco.

> Nota del propio código (comentario en el route handler): no hay chequeo de "owner" del acta — el modelo actual es single-tenant / single-usuario superadmin.

## Artefactos

`app/api/artefactos/[id]/route.ts`

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/artefactos/[id]` | Sirve el archivo del artefacto (video/captura/trace) |

`Content-Type` según `ArtefactoTipo`: `video` → `video/webm`, `captura` → `image/png`, `trace` → `application/json`, cualquier otro → `application/octet-stream`.

## Credenciales

Ver [Proyectos](#proyectos) — `GET /api/proyectos/[id]/credenciales`. No hay rutas de creación/edición de credenciales bajo `app/api/` (se gestionan vía Server Actions, no confirmado en este pase — ver `lib/credenciales/`).

## Usuarios y sesión

`app/api/usuarios/route.ts`, `app/api/logout/route.ts`

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/usuarios` | Lista usuarios |
| POST | `/api/logout` | Destruye la sesión (`iron-session`) y redirige a `/login` (`307`) |

## Grabador

`app/api/grabador/sesiones/*` — modo grabador (`lib/grabador/`, `lib/recorder/`).

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/api/grabador/sesiones` | Inicia una sesión de grabación |
| GET | `/api/grabador/sesiones/[id]` | Detalle de la sesión |
| PATCH | `/api/grabador/sesiones/[id]` | Actualiza la sesión |
| DELETE | `/api/grabador/sesiones/[id]` | Elimina la sesión |
| POST | `/api/grabador/sesiones/[id]/descartar` | Descarta la grabación |
| POST | `/api/grabador/sesiones/[id]/guardar` | Guarda la grabación (persiste `specCode` como `CasoPrueba.script`) |
| POST | `/api/grabador/sesiones/[id]/heartbeat` | Heartbeat para mantener viva la sesión (ver `RECORDER_HEARTBEAT_TIMEOUT_MS`) |
| POST | `/api/grabador/sesiones/[id]/pause` | Pausa la grabación |
| POST | `/api/grabador/sesiones/[id]/resume` | Reanuda la grabación |
| POST | `/api/grabador/sesiones/[id]/parametros` | Registra un parámetro capturado durante la grabación |
| POST | `/api/grabador/sesiones/[id]/pasos` | Agrega un paso grabado |
| PATCH | `/api/grabador/sesiones/[id]/pasos` | Actualiza pasos grabados en lote |
| PATCH | `/api/grabador/sesiones/[id]/pasos/[pasoId]` | Actualiza un paso puntual |
| DELETE | `/api/grabador/sesiones/[id]/pasos/[pasoId]` | Elimina un paso |

**`POST /api/grabador/sesiones`** (`app/api/grabador/sesiones/route.ts`) — wrapper HTTP de la Server Action `iniciarSesionGrabacion`. Auth: superadmin (verificado dentro de la action). Body: `NuevaGrabacionInput` (`lib/grabador/types.ts`). Respuestas documentadas en el propio código: `201` con `SesionGrabacionOut`, `400` validación, `401` sin sesión, `403` forbidden, `503` si el recorder-worker no responde.
