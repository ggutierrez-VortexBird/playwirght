# Exploration: HU-2.1 — Crear espacio de empresa

## Current State

### Modelo de datos — Espacio (Listo ✅)
The `Espacio` model is already defined in `prisma/schema.prisma` (lines 57–67) and the migration `20260806145203_init` has been applied:

```
Espacio {
  id        String   @id @default(uuid())
  nombre    String
  color     String
  activo    Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  proyectos Proyecto[]
  miembros  UsuarioEspacio[]
}
```

Fields: `nombre` (String, required), `color` (String, required — hex color like `#C9822F`), `activo` (Boolean, default true). The model has `@@index` on `Espacio` implicitly via the FK from `UsuarioEspacio` and `Proyecto`. No unique constraint on `nombre` (multiple espacios can share a name).

### API routes — Vacío 🔴
Only `/api/logout` exists. No API routes for espacios at all. The following need to be created:
- `GET    /api/espacios`        — list all espacios (filtered by user membership via `UsuarioEspacio`)
- `POST   /api/espacios`        — create espacio (requires superadmin role)
- `GET    /api/espacios/[id]`   — get single espacio
- `PUT    /api/espacios/[id]`   — update espacio (nombre, color)
- `DELETE /api/espacios/[id]`   — soft-delete (set activo=false) or hard-delete

### UI — Parcial 🟡
- `(dashboard)` route group exists with `proyectos`, `casos`, `ejecuciones`, `credenciales` pages.
- All existing pages are stubs — no real data fetching.
- No `/espacios` page exists yet.
- The sidebar (layout.tsx) shows nav items but no espacios entry. The switcher/sidebar brand area in the mockup (`acta-mockups.html`) shows a client-color switcher — this is conceptually the "espacios" UI.
- **Decision needed**: Where does espacios UI live? Options: (a) `/espacios` page as its own section, (b) a modal/dialog on top of proyectos, (c) a dropdown switcher in the sidebar header. Based on mockup, the switcher in the sidebar rail suggests a space-switcher pattern. But the acceptance criteria require listing, creating, and editing — a full page makes more sense.

### Auth — En lugar ✅
Iron-session is set up with `getSession()`, `saveSession()`, `destroySession()`. Session contains `{ userId, email }`. Dashboard layout checks `session.userId` and redirects to `/login` if absent. No role-checking middleware exists yet — role checks must be done inline in each action/route.

### Estructura de la app ✅
- `app/(dashboard)/` — route group with shared sidebar layout (auth-protected)
- `app/api/` — only `logout/route.ts`
- `app/login/` — login form + server action
- `lib/auth.ts` — iron-session wrapper
- `lib/db.ts` — Prisma singleton
- `tailwind.config.ts` — custom design tokens matching mockup CSS vars (`--client`, `--seal`, `--stamp`, `--amber`, `--param`)

### Prisma migrations ✅
Migration `20260806145203_init` applied — includes `Espacio`, `UsuarioEspacio`, `Proyecto`, `CasoPrueba`, `Ejecucion`, `PasoEjecucion`, `Artefacto`, `Acta`, `Credencial`, `ConsecutivoAnual`.

---

## Affected Areas

| File/Directory | Why affected |
|---|---|
| `app/(dashboard)/espacios/page.tsx` | New — main espacios listing + create/edit UI |
| `app/api/espacios/route.ts` | New — GET list, POST create |
| `app/api/espacios/[id]/route.ts` | New — GET, PUT, DELETE single espacio |
| `lib/auth.ts` | May need a helper like `requireSuperadmin()` or role check |
| `prisma/schema.prisma` | No change needed — model exists |
| `app/(dashboard)/layout.tsx` | Needs espacios nav item in sidebar |
| `tailwind.config.ts` | Color tokens already defined (client, seal, etc.) |
| `app/globals.css` | Design tokens referenced here |

---

## Approaches

### 1. Page + Server Actions (Recommended)
Create `app/(dashboard)/espacios/page.tsx` with a list and a create/edit form. Use React Server Components for data fetching. Create `app/api/espacios/` API routes for JSON API responses (for future SPA/mobile consumption). Server actions handle form submissions.

- **Pros**: Next.js idiomatic, RSC handles data fetching cleanly, good for progressive enhancement
- **Cons**: API routes + server actions = some duplication; form state management needs care
- **Effort**: Medium

### 2. API Routes + Client Components Only
Skip server actions. All mutations go through API routes. Page is a thin RSC that fetches data server-side.

- **Pros**: Single patterns for all mutations (API routes only)
- **Cons**: Loses progressive enhancement; more boilerplate
- **Effort**: Medium

### 3. Standalone `/espacios` route group (outside dashboard)
Create `app/espacios/page.tsx` with its own mini-layout. Useful if espacios is a top-level standalone concept (before selecting a space).

- **Pros**: Cleaner separation, doesn't require prior espacio selection
- **Cons**: Duplicates auth check, different layout from the rest of the app
- **Effort**: High

---

## Recommendation

**Approach 1** — `app/(dashboard)/espacios/page.tsx` + API routes under `app/api/espacios/`.

Rationale:
- The espacios section belongs inside the `(dashboard)` group since it requires auth and uses the same sidebar layout.
- API routes are needed anyway for non-form consumers (future mobile app, integrations).
- Server actions handle the create/edit form with `useActionState` or `useFormStatus`.
- Soft-delete (`activo=false`) is the right default for `DELETE` — spaces hold critical historical data (proyectos, casos, ejecuciones).

**Naming convention** for API routes: `app/api/espacios/route.ts` and `app/api/espacios/[id]/route.ts` (plural, lowercase).

**Color picker**: Use a predefined palette of 8–10 colors (matching the design tokens in tailwind: `client`, `seal`, `stamp`, `amber`, `param` + additional ones) rendered as a clickable swatch grid. Free hex input as advanced option.

---

## Risks

1. **Color uniqueness not enforced** — Multiple espacios can have the same color. This is by design (mockup shows same color for same client), but no validation prevents identical colors from different clients.
2. **No role guard on API routes** — The `superadmin` role check for creating espacios must be added to every route handler. Easy to forget.
3. **No pagination** — `GET /api/espacios` returns all espacios. If many espacios exist this could be slow. Should add pagination early.
4. **Sidebar UX gap** — Adding "Espacios" to the sidebar requires deciding what happens to "Proyectos" — does it move under a specific espacio or remain as a separate view?
5. **Soft-delete cascade question** — If a espacio is soft-deleted (`activo=false`), should its proyectos/casos still appear? Likely yes (read-only), but this needs explicit handling.

---

## Ready for Proposal

**Yes** — The schema is ready, auth is in place, and the scope is clear. The next step is `sdd-propose` / `sdd-spec` to formalize the acceptance criteria into concrete technical specs.

Key decisions to lock in before spec:
1. Where does the espacios nav live? (top of sidebar rail vs. full page vs. both)
2. Does `DELETE` on espacio soft-delete or hard-delete?
3. Is there a limit on espacios per user?
4. Should the color picker be restricted palette or free-form?
