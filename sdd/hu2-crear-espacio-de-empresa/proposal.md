# Proposal: HU-2.1 — Crear espacio de empresa

## Intent

"As a superadmin user, I want to create an empresa/espacio with a name and a color identifier, so I can start organizing the work for a new client."

## Scope

### In Scope
- CRUD for espacios (create, read list, update, soft-delete)
- API routes: `GET /api/espacios` (list), `POST /api/espacios` (create), `PUT /api/espacios/[id]` (update), `DELETE /api/espacios/[id]` (soft-delete)
- UI: espacios listing page at `app/(dashboard)/espacios/page.tsx` with create form and inline edit
- Color picker using a predefined palette of 8–10 colors (not free color input)
- Sidebar navigation link to `/espacios`

### Out of Scope
- Role-based access (MVP: any authenticated user can manage espacios)
- Pagination (MVP: assume few espacios)
- Unique color constraint (multiple espacios can share a color)
- Proximity to proyectos listing (HU-2.2 handles projects inside spaces)

## Capabilities

### New Capabilities
- `espacio-crud`: Full CRUD for empresa espacios (list, create, edit, soft-delete)

### Modified Capabilities
- None

## Approach

- **API routes** under `app/api/espacios/` and `app/api/espacios/[id]/` returning JSON
- **UI page** at `app/(dashboard)/espacios/page.tsx` using React Server Components for data fetching + client components for forms
- **Soft-delete** — `DELETE` sets `activo=false` instead of hard delete (preserves historical data)
- **Color picker** — predefined palette rendered as clickable swatch grid; colors match design tokens (`--client`, `--seal`, `--stamp`, `--amber`, `--param`) plus additional ones
- **Prisma queries** directly in route handlers (MVP simplicity — no service layer)
- **Auth guard** inline in route handlers (MVP — no middleware role check yet)

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `prisma/schema.prisma` | Reference | Model already exists — no changes needed |
| `app/api/espacios/route.ts` | New | GET list + POST create |
| `app/api/espacios/[id]/route.ts` | New | PUT update + DELETE soft-delete |
| `app/(dashboard)/espacios/page.tsx` | New | UI for list + create + edit |
| `components/` | New | ColorPicker component |
| `app/(dashboard)/layout.tsx` | Modify | Add sidebar link to espacios |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| No authorization guard | Medium | Documented as MVP limitation; future HU will add roles |
| Duplicate color names | Low | Color picker uses fixed palette — no free input |
| Soft-delete cascade | Low | Proyekte/casos remain readable after espacio soft-delete |

## Rollback Plan

- Delete `app/api/espacios/` directory (all routes)
- Delete `app/(dashboard)/espacios/` directory
- Delete `components/ColorPicker.tsx` (if created as separate file)
- Revert `app/(dashboard)/layout.tsx` to remove sidebar link
- No DB migration needed (no schema changes)

## Dependencies

- Iron-session auth (HU-1.1) — already in place
- `Espacio` Prisma model — already migrated

## Success Criteria

- [ ] Can create an espacio with name and color from UI
- [ ] Cannot create espacio without name (validation error)
- [ ] Created espacio appears in listing
- [ ] Can edit existing espacio name and color
- [ ] Can soft-delete espacio (disappears from listing but stays in DB)
