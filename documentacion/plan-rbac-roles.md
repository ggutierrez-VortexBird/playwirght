# Sistema de roles y permisos (RBAC) — superadmin / admin / tester

## Contexto

Hoy vorTest tiene un solo rol real: `Usuario.rol` es un `String` libre que
siempre vale `"superadmin"` (seed único `admin@admin.com`). El único guard de
autorización que existe es `requireSuperadmin(session)` en `lib/auth.ts`, y se
usa como primera línea de cada Server Action mutadora
(`lib/espacios/actions.ts`, `lib/proyectos/actions.ts`,
`lib/casos/actions.ts`, `lib/ejecuciones/actions.ts`,
`lib/grabador/actions.ts`). Las páginas del dashboard duplican, cada una por
su cuenta, la misma consulta a Prisma para calcular un booleano
`isSuperadmin`/`canEdit` (`app/(dashboard)/page.tsx`,
`app/(dashboard)/proyectos/page.tsx`,
`app/(dashboard)/espacios/[id]/proyectos/page.tsx`,
`app/(dashboard)/casos/page.tsx`,
`app/(dashboard)/proyectos/[id]/casos/page.tsx`). No existe UI para crear
usuarios (solo un `GET /api/usuarios` que lista `{id, email}`, consumido por
un selector de "responsable" de caso). La tabla `UsuarioEspacio` existe en el
schema pero no se usa en ningún lado del código de aplicación. No existe
ninguna relación Usuario↔Proyecto.

El objetivo es introducir tres roles con alcance jerárquico:

- **superadmin** (el `admin@admin.com` actual): acceso total. Único que crea
  Espacios, crea usuarios de cualquier rol (`admin` o `tester`), y asigna o
  quita el rol de administrador de un Espacio a un usuario. Único con acceso
  a Credenciales.
- **admin**: administra uno o varios Espacios que el superadmin le asignó
  (relación muchos-a-muchos: un admin puede tener varios espacios). Dentro de
  esos espacios puede crear/editar/borrar Proyectos, Casos y Ejecuciones, pero
  **no puede editar el Espacio en sí** ni ver/usar Credenciales. Puede crear
  usuarios nuevos, pero siempre con rol `tester` (no puede crear otros
  admins). Su único poder de gestión de personas: asignar y quitar testers
  *ya existentes* de los Proyectos de sus espacios — nada más.
- **tester**: no tiene acceso a nada hasta que un admin (o superadmin) lo
  asigna a un Proyecto puntual. Una vez asignado, puede hacer CRUD de Casos y
  Ejecuciones de ese Proyecto, pero no puede editar el Proyecto ni ver otros
  Proyectos/Espacios que no le fueron asignados. No ve Credenciales.

Decisiones ya confirmadas con el usuario (no rediscutir):
- Admin↔Espacio es muchos-a-muchos.
- Superadmin y admin pueden crear usuarios; superadmin puede darles rol
  `admin` o `tester`, admin solo puede darles rol `tester`.
- Solo el superadmin asigna/quita el rol de admin de un Espacio (ascenso
  tester→admin y vínculo a espacio).
- Un tester sin Proyecto asignado no debe ver ese Proyecto en absoluto
  (filtrado, no solo lectura).
- Credenciales queda 100% exclusivo de superadmin; admin y tester no la ven
  ni la usan.

## Cambios en el schema (`prisma/schema.prisma`)

1. Convertir `Usuario.rol` de `String @default("superadmin")` a un enum
   Prisma nuevo, consistente con el resto del schema (`EjecucionEstado`, etc.):
   ```prisma
   enum Rol {
     superadmin
     admin
     tester
   }
   ```
   `Usuario.rol Rol @default(tester)`. Migración: el único registro existente
   (`admin@admin.com`) ya tiene el valor literal `"superadmin"`, coincide con
   el enum sin necesidad de backfill.

2. Reutilizar `UsuarioEspacio` (hoy sin uso) como la tabla admin↔espacio: la
   presencia de una fila `UsuarioEspacio(usuarioId, espacioId)` significa
   "este usuario administra este espacio". No necesita columna de rol propia
   porque solo se crean filas para usuarios con `rol = admin`.

3. Nuevo modelo `UsuarioProyecto` (tester↔proyecto), análogo a
   `UsuarioEspacio`:
   ```prisma
   model UsuarioProyecto {
     usuarioId  String
     proyectoId String
     createdAt  DateTime @default(now()) @db.Timestamptz(6)

     usuario  Usuario  @relation(fields: [usuarioId], references: [id], onDelete: Cascade)
     proyecto Proyecto @relation(fields: [proyectoId], references: [id], onDelete: Cascade)

     @@id([usuarioId, proyectoId])
     @@index([proyectoId])
   }
   ```
   Agregar `usuarios UsuarioProyecto[]` a `Proyecto` y `proyectos
   UsuarioProyecto[]` a `Usuario`.

4. Nueva migración Prisma (`npx prisma migrate dev --name rbac_roles`).

## Capa de autorización (`lib/auth.ts`)

- Nuevo helper `getUsuarioActual(session)`: una sola consulta
  `{id, email, rol}`, para reemplazar las consultas duplicadas en cada page.
- Guards nuevos, junto al `requireSuperadmin` existente (que se mantiene tal
  cual para Espacios y Credenciales):
  - `requireEspacioAdmin(session, espacioId)`: pasa si `superadmin`, o si
    `admin` con fila `UsuarioEspacio` para ese `espacioId`.
  - `requireProyectoAccess(session, proyectoId)`: resuelve el `espacioId` del
    proyecto; pasa si `superadmin`, si `admin` de ese espacio, o si `tester`
    con fila `UsuarioProyecto` para ese `proyectoId`. Se usa para Casos y
    Ejecuciones (CRUD permitido a los tres roles con acceso).
  - `requireProyectoManage(session, espacioId)`: alias de
    `requireEspacioAdmin` — crear/editar/borrar el Proyecto en sí excluye a
    tester.
- Todos devuelven/lanzan `FORBIDDEN_ERROR` igual que hoy, para no romper el
  patrón de manejo de errores ya existente en las rutas API.

## Server Actions a rewire (mismo archivo, solo cambia el guard de entrada)

- `lib/espacios/actions.ts` (`createEspacio`, `updateEspacio`,
  `deleteEspacio`): sin cambios, siguen `requireSuperadmin`.
- `lib/proyectos/actions.ts` (`createProyecto`, `updateProyecto`,
  `deleteProyecto`): `requireSuperadmin` → `requireEspacioAdmin(session,
  espacioId)`.
- `lib/casos/actions.ts` (`createCaso`, `updateCaso`, `deleteCaso`):
  `requireSuperadmin` → `requireProyectoAccess(session, proyectoId)`.
- `lib/ejecuciones/actions.ts` (`dispararEjecucion`, `detenerEjecucion`):
  ídem `requireProyectoAccess`.
- `lib/grabador/actions.ts` (inicio de sesión de grabación): ídem
  `requireProyectoAccess`.
- Credenciales (buscar el archivo de actions correspondiente, no relevado
  aún en la exploración): se deja `requireSuperadmin` sin cambios.

## Gestión de personas (superficie nueva)

- `lib/usuarios/actions.ts` (nuevo): `createUsuario({email, password, rol},
  session)` — valida con Zod que `rol` sea `admin`/`tester`; si quien llama
  es `admin`, fuerza `rol = tester` sin importar lo que pida el body; si es
  `superadmin`, respeta el rol pedido. `listUsuarios(session)` — superadmin ve
  todos; admin ve todos los usuarios con `rol = tester` (para poder elegir a
  quién asignar).
- `lib/espacios/actions.ts` (agregar): `asignarAdminEspacio(espacioId,
  usuarioId, session)` / `quitarAdminEspacio(...)` — `requireSuperadmin`,
  valida que el usuario destino tenga `rol = admin`, crea/borra la fila
  `UsuarioEspacio`.
- `lib/proyectos/actions.ts` (agregar): `asignarTesterProyecto(proyectoId,
  usuarioId, session)` / `quitarTesterProyecto(...)` —
  `requireEspacioAdmin(session, espacioId-del-proyecto)`, valida `rol =
  tester` en el destino, crea/borra la fila `UsuarioProyecto`.
- `app/api/usuarios/route.ts`: extender `GET` para aplicar el filtro de
  `listUsuarios` según rol de quien pregunta; agregar `POST` → `createUsuario`.
- Rutas nuevas: `app/api/espacios/[id]/admins/route.ts` (POST/DELETE) y
  `app/api/proyectos/[id]/testers/route.ts` (GET/POST/DELETE).

## Páginas / UI

- Centralizar el patrón `isSuperadmin`/`canEdit` duplicado hoy en 5 páginas
  usando `getUsuarioActual` + los nuevos guards (misma lógica, un solo lugar).
- Filtrado por alcance en las consultas de listado (no solo ocultar botones):
  - `admin`: solo Espacios donde tiene fila `UsuarioEspacio`, y Proyectos de
    esos Espacios.
  - `tester`: solo Proyectos donde tiene fila `UsuarioProyecto`, y
    Casos/Ejecuciones de esos Proyectos. No debe ver la lista de Espacios en
    absoluto.
- Página nueva `app/(dashboard)/usuarios/page.tsx`: listado + formulario de
  alta (visible a superadmin y admin; el selector de rol solo aparece para
  superadmin, admin siempre crea `tester`).
- Dentro de la vista de detalle de Proyecto: panel "Testers del proyecto"
  (visible a superadmin y al admin del espacio) para asignar/quitar, usando
  `listUsuarios` filtrado a `tester`.
- Dentro de la vista de detalle/listado de Espacio: panel "Administradores"
  (solo superadmin) para asignar/quitar admins del espacio.
- `components/ui/sidebar-nav.tsx` / `NAV_ITEMS` en
  `app/(dashboard)/layout.tsx`: ocultar "Credenciales" para `admin`/`tester`;
  ocultar "Espacios" para `tester`; agregar ítem "Usuarios" para
  `superadmin`/`admin`.

## Verificación

- `npm run typecheck` y `npm run lint` en `playwright_vortex/`.
- Actualizar/extender tests existentes que hoy asumen "todo o nada"
  (`__tests__/lib/auth.test.ts`, `__tests__/lib/espacios/actions.test.ts`,
  `__tests__/lib/proyectos/actions.test.ts`, tests de `casos`/`ejecuciones`
  actions, `__tests__/app/api/usuarios/route.test.ts`,
  `__tests__/app/(dashboard)/layout.test.tsx`) para cubrir los tres roles y
  los casos de rechazo (admin de otro espacio, tester sin acceso al
  proyecto).
- `npm test` completo.
- Prueba manual end-to-end con `npm run dev`: como superadmin crear un
  Espacio, crear un usuario admin y asignarlo al Espacio; como ese admin,
  crear un Proyecto, crear un usuario tester y asignarlo al Proyecto; como
  ese tester, verificar que puede CRUD Casos/Ejecuciones del Proyecto
  asignado, y verificar que NO ve otros Proyectos/Espacios ni la sección de
  Credenciales.
