# Tasks: HU-2.1 — Crear espacio de empresa

## Review Workload Forecast

| Field | Value |
|-------|-------|
| Estimated changed lines | 300–450 |
| 400-line budget risk | Medium |
| Chained PRs recommended | No |
| Suggested split | Single PR |
| Delivery strategy | single-pr |
| Chain strategy | pending |

Decision needed before apply: No
Chained PRs recommended: No
Chain strategy: pending
400-line budget risk: Medium

### Suggested Work Units

| Unit | Goal | Likely PR | Notes |
|------|------|-----------|-------|
| 1 | All: API routes + ColorPicker + UI page + sidebar nav | PR 1 | Tests included |

## Phase 1: Infrastructure / Types

- [x] 1.1 Define TypeScript types for Espacio API responses in `types/espacio.ts` (Espacio, CreateEspacioInput, UpdateEspacioInput, ApiError)
- [x] 1.2 Verify Prisma schema has `Espacio` model with fields: id, nombre, color, activo, createdAt, updatedAt

## Phase 2: API Routes

- [x] 2.1 Create `app/api/espacios/route.ts` — GET returns all activo=true espacios ordered by createdAt DESC; POST creates new espacio with nombre + color, returns 400 if missing
- [x] 2.2 Create `app/api/espacios/[id]/route.ts` — PUT updates nombre and/or color; DELETE sets activo=false (soft-delete), returns 404 if not found
- [x] 2.3 Wire iron-session auth guard on all four handlers (GET, POST, PUT, DELETE) — return 401 if no session

## Phase 3: UI Components

- [x] 3.1 Create `components/ui/color-picker.tsx` — 8 predefined swatch buttons (palette from design: #C9822F, #0E6B4F, #A8322A, #A9741A, #3F3A7A, #2F6FA8, #5C8A3A, #6B7C8D) with selected state
- [x] 3.2 Create `app/(dashboard)/espacios/page.tsx` — RSC page listing espacios as color-coded rows; inline create form (nombre input + ColorPicker); inline edit on row click

## Phase 4: Navigation

- [x] 4.1 Add sidebar nav link to `/espacios` in `app/(dashboard)/layout.tsx` — place between "Proyectos" and "Casos" with dot indicator using existing nav pattern

## Phase 5: Testing

- [x] 5.1 Write Jest tests for API route handlers: GET /api/espacios (empty + populated list), POST /api/espacios (success + validation 400), PUT /api/espacios/[id] (success + 404), DELETE /api/espacios/[id] (success + 404)
- [x] 5.2 Write Playwright E2E test: create espacio, verify it appears in list, edit it, delete it and verify gone
