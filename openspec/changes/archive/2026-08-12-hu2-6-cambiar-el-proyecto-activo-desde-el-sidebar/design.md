# Design: HU-2.6 — ProyectoSwitcher in Sidebar

## Technical Approach

Add a `ProyectoSwitcher` client component to the dashboard sidebar that lists active projects with their espacio and color. Follows the existing `EspacioSwitcher` pattern: server component fetches data in `layout.tsx`, client component renders dropdown and handles navigation via `useRouter`.

## Architecture Decisions

### Decision: Data Fetching Location

**Choice**: Fetch `listProyectosActivos()` in the dashboard layout server component
**Alternatives considered**: Fetch in client component with SWR, fetch in parent page components
**Rationale**: Layout already fetches `espacios` for `EspacioSwitcher`; adding `proyectos` is consistent. Server component avoids client-side loading states for initial render.

### Decision: Client/Server Split

**Choice**: `ProyectoSwitcher` is a `'use client'` component that receives pre-fetched proyectos
**Alternatives considered**: Pure server component with server actions, client-only fetch
**Rationale**: `EspacioSwitcher` uses this exact pattern; maintains consistency. Dropdown open/close state and `useRouter` navigation require client context.

### Decision: Single-Project Static Mode

**Choice**: When `proyectos.length === 1`, render static header instead of dropdown button
**Alternatives considered**: Always show dropdown (even with 1 item), hide component entirely
**Rationale**: Matches `EspacioSwitcher` behavior philosophy — presence of the switcher provides context even when selection is trivial.

## Data Flow

```
layout.tsx (Server)
  │
  ├── listProyectosActivos() ──→ Prisma ──→ returns (Proyecto & { espacio: Espacio })[]
  │
  └── <ProyectoSwitcher proyectos={proyectos} />
            │
            ├── useSelectedLayoutSegments() ──→ extracts [proyectos, id] from URL
            │
            ├── isOpen state ──→ dropdown visibility
            │
            └── router.push(/proyectos/${id}/casos) ──→ Next.js navigation
```

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `playwright_vortex/lib/proyectos/actions.ts` | Modify | Add `listProyectosActivos()` with `include: { espacio }` |
| `playwright_vortex/components/ui/proyecto-switcher.tsx` | Create | Client component following `EspacioSwitcher` pattern |
| `playwright_vortex/app/(dashboard)/layout.tsx` | Modify | Import `listProyectosActivos`, render `<ProyectoSwitcher>` in aside |

## Interface Definitions

```typescript
// New type in types/proyecto.ts
export interface ProyectoWithEspacio extends Proyecto {
  espacio: Espacio;
}
```

## New Action: listProyectosActivos

```typescript
export async function listProyectosActivos() {
  return prisma.proyecto.findMany({
    where: { activo: true },
    include: { espacio: true },
    orderBy: { createdAt: "desc" },
  });
}
```

## Component: ProyectoSwitcher

Props: `proyectos: (Proyecto & { espacio: Espacio })[]`

Key behaviors:
- URL detection: `useSelectedLayoutSegments()` returns segments; if `segments[0] === 'proyectos'` then `segments[1]` is active project ID
- Navigation: `router.push(\`/proyectos/${id}/casos\`)` on selection
- Single-project: renders static header with `proyectos[0].nombre` and `proyectos[0].espacio.color`
- Multi-project: renders dropdown button + listbox, same visual style as `EspacioSwitcher`

## Testing Strategy

| Layer | What to Test | Approach |
|-------|-------------|----------|
| Unit | `listProyectosActivos()` returns correct shape | Direct Prisma mock call |
| Unit | `ProyectoSwitcher` renders static vs dropdown | Shallow render with different `proyectos` arrays |
| Unit | URL detection extracts correct proyecto ID | Segment mocking |
| Integration | Selecting project navigates to correct URL | `router.push` spy |

## Migration / Rollout

No migration required. Pure additive feature — no schema changes, no data transformation.

## Open Questions

None — all decisions resolved by following existing `EspacioSwitcher` pattern.
