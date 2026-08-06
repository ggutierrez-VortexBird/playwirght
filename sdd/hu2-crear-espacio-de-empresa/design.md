# Design: HU-2.1 Crear espacio de empresa

## Technical Approach

Next.js 15 App Router with React Server Components for data fetching. API routes under `app/api/espacios/` return JSON for future extensibility. Prisma queries directly in route handlers (no service layer — MVP simplicity). iron-session for auth from HU-1.1.

## Architecture Decisions

### Decision: Fixed color palette over free color input

**Choice**: Predefined palette of 8 colors rendered as clickable swatches
**Alternatives considered**: Native `<input type="color">`, third-party color picker library
**Rationale**: MVP scope — predefined palette avoids color validation logic, ensures brand-consistent colors, and eliminates dependencies. Native color input is ugly and misaligned with the design system.

Color palette (matches design tokens + additions):
- `#C9822F` (client/orange)
- `#0E6B4F` (seal/green)
- `#A8322A` (stamp/red)
- `#A9741A` (amber/yellow)
- `#3F3A7A` (param/purple)
- `#2F6FA8` (blue)
- `#5C8A3A` (green-alt)
- `#6B7C8D` (neutral)

### Decision: Soft-delete over hard-delete

**Choice**: DELETE sets `activo=false` instead of removing the row
**Alternatives considered**: Hard delete (permanent removal)
**Rationale**: Audit trail and data integrity — if an espacio has associated projects (future HU-2.2), soft-delete prevents accidental data loss and allows recovery.

### Decision: API-first with RSC page

**Choice**: API routes handle all mutations; page fetches via RSC + client form components
**Alternatives considered**: Server actions only (no API routes)
**Rationale**: API routes needed for future SPA/mobile consumers. Separation of concerns — API is transport-agnostic, UI is a client.

### Decision: No pagination in MVP

**Choice**: Return all active espacios in a single array
**Alternatives considered**: Cursor or offset pagination
**Rationale**: MVP assumes few espacios per instance. Pagination deferred to future HU when scale becomes a concern.

## Data Flow

```
User → app/(dashboard)/espacios/page.tsx (RSC)
     → GET /api/espacios → Prisma → Postgres
     → POST/PUT/DELETE → API routes → Prisma → Postgres
```

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `app/api/espacios/route.ts` | Create | GET (list active) + POST (create) |
| `app/api/espacios/[id]/route.ts` | Create | GET + PUT (update) + DELETE (soft-delete) |
| `app/(dashboard)/espacios/page.tsx` | Create | RSC page listing espacios with create/edit form |
| `app/(dashboard)/layout.tsx` | Modify | Add sidebar nav link to /espacios |

## API Contracts

### GET /api/espacios
Returns all active (`activo=true`) espacios ordered by `createdAt` DESC.

Response 200:
```json
[{ "id": "abc", "nombre": "Acme", "color": "#C9822F", "activo": true, "createdAt": "...", "updatedAt": "..." }]
```

### POST /api/espacios
Request: `{ "nombre": "string", "color": "string" }` — both required

Response 201: Created espacio object
Response 400: `{ "error": "validation", "message": "nombre is required" }`

### PUT /api/espacios/[id]
Request: `{ "nombre": "string", "color": "string" }` — both optional (partial update)

Response 200: Updated espacio object
Response 404: `{ "error": "not_found" }`

### DELETE /api/espacios/[id]
Soft-deletes by setting `activo=false`.

Response 200: `{ "success": true }`
Response 404: `{ "error": "not_found" }`

## UI Design

- Page at `/espacios` inside `(dashboard)` route group (shares sidebar layout)
- Full-width card listing all espacios as color-coded rows
- "Crear espacio" form inline at top of page (expandable/collapsible)
- Edit form appears inline on the same row when editing
- Color picker: 8-swatch grid using Tailwind bg-* classes matching design tokens
- Sidebar nav item: `Espacios` link between "Proyectos" and "Casos" with dot indicator

## Testing Strategy

| Layer | What to Test | Approach |
|-------|-------------|----------|
| Unit | Prisma queries in isolation | Jest + Prisma mock |
| Integration | API routes with real DB | Jest + test database |
| E2E | Full CRUD flow from UI | Playwright |

## Migration / Rollout

No migration required. Prisma schema already has the `Espacio` model from HU-0.1.

## Open Questions

- [ ] Should non-superadmin users be able to view espacios (read-only)?
- [ ] Does the sidebar "Espacios" link appear for all roles or superadmin only?
