# Proposal: HU-2.6 — Cambiar el proyecto activo desde el sidebar

## Intent

Permitir cambiar el proyecto activo desde el sidebar sin recargar la página, usando un `ProyectoSwitcher` dropdown que liste proyectos activos con su espacio y color. Sigue el patrón de `EspacioSwitcher` (HU-2.5).

## Scope

### In Scope
- Nueva acción `listProyectosActivos()` en `lib/proyectos/actions.ts` con `include: { espacio }`
- Nuevo componente `ProyectoSwitcher` en `components/ui/proyecto-switcher.tsx` (cliente, sigue patrón `EspacioSwitcher`)
- Integración de `ProyectoSwitcher` en `app/(dashboard)/layout.tsx` dentro del aside sidebar
- Lógica de single-project: si solo hay 1 proyecto, mostrar encabezado estático sin dropdown

### Out of Scope
- Crear nuevos endpoints API (se reutiliza el layout server component)
- Modificar routing — la navegación ya es URL-driven (`/proyectos/[id]/casos`)
- Cambios en el diseño visual del sidebar más allá del switcher

## Capabilities

### New Capabilities
- `proyecto-switcher`: Dropdown en sidebar para cambiar proyecto activo. Muestra nombre, espacio y color. Solo renderiza dropdown si hay >1 proyecto; si no, es encabezado estático.

### Modified Capabilities
- `dashboard-layout`: Ahora incluye `ProyectoSwitcher` además de `EspacioSwitcher` en el aside. Ambos coexisten.

## Approach

1. **`lib/proyectos/actions.ts`**: Crear `listProyectosActivos()` que haga `prisma.proyecto.findMany({ where: { activo: true }, include: { espacio: true }, orderBy: { createdAt: "desc" } })`.

2. **`components/ui/proyecto-switcher.tsx`**: Componente cliente `'use client'` que:
   - Reciba `proyectos: (Proyecto & { espacio: Espacio })[]`
   - Detecte proyecto activo desde URL (`proyectos/[id]/casos`)
   - Si `proyectos.length === 1`: renderiza encabezado estático (nombre + color)
   - Si `proyectos.length > 1`: renderiza dropdown clickeable, al seleccionar hace `router.push(/proyectos/${id}/casos)`
   - Reutiliza estilos `.sw-mark` y estructura visual de `EspacioSwitcher`

3. **`app/(dashboard)/layout.tsx`**: Importar `listProyectosActivos`, pasar proyectos a `ProyectoSwitcher`.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `lib/proyectos/actions.ts` | Modified | Agregar `listProyectosActivos()` |
| `components/ui/proyecto-switcher.tsx` | New | Dropdown switcher para proyectos |
| `app/(dashboard)/layout.tsx` | Modified | Integrar `ProyectoSwitcher` en aside |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Dropdown compite visualmente con `EspacioSwitcher` | Low | Position absolute, z-index 50, similar pero más pequeño |
| URL no contiene `proyectoId` en todas las páginas | Low | Fallback: buscar primer proyecto activo del espacio actual |

## Rollback Plan

1. Remover `<ProyectoSwitcher>` de `layout.tsx`
2. Eliminar `listProyectosActivos()` de `actions.ts` (o mantener si futuro HU-2.7 lo requiere)
3. Eliminar `components/ui/proyecto-switcher.tsx`

## Dependencies

- Prisma schema: modelo `Proyecto` con relación `espacio` (ya existe)
- `EspacioSwitcher` como patrón de referencia (ya implementado en HU-2.5)

## Success Criteria

- [ ] `listProyectosActivos()` retorna proyectos activos con espacio cargado
- [ ] `ProyectoSwitcher` muestra dropdown con nombre + espacio + color cuando hay >1 proyecto
- [ ] `ProyectoSwitcher` es encabezado estático cuando hay exactamente 1 proyecto
- [ ] Seleccionar proyecto navega a `/proyectos/[id]/casos` sin recarga completa
- [ ] Layout server component compila sin errores
