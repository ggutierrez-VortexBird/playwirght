# Contribuir

## Tabla de contenidos

- [Antes de un commit](#antes-de-un-commit)
- [Convenciones de código](#convenciones-de-código)
- [Flujo de ramas](#flujo-de-ramas)

## Antes de un commit

Correr, desde `playwright_vortex/`:

```bash
npm run lint
npm run typecheck
npm test
npx playwright test e2e/
```

- `npm run lint` → `next lint`
- `npm run typecheck` → `tsc --noEmit`
- `npm test` → suite unitaria/integración con Jest (`jest.config.mjs`)
- `npx playwright test e2e/` → suite E2E (requiere la app corriendo, ver [`SETUP.md`](./SETUP.md))

## Convenciones de código

Basadas en la estructura real del código en `lib/`:

- TypeScript en todo el proyecto (`tsconfig.json`, `strict` según configuración del proyecto).
- Lógica de negocio organizada por dominio bajo `lib/<dominio>/`, con un archivo `actions.ts` para las Server Actions/mutaciones del dominio (ej. `lib/casos/actions.ts`, `lib/proyectos/actions.ts`, `lib/espacios/actions.ts`, `lib/ejecuciones/actions.ts`, `lib/grabador/actions.ts`).
- Los route handlers de `app/api/**/route.ts` son wrappers delgados: validan sesión, parsean el request y delegan la lógica a la Server Action correspondiente en `lib/`. Ver `app/api/grabador/sesiones/route.ts` como ejemplo explícito (comentario "Wrapper HTTP que invoca la Server Action").
- Errores de negocio se lanzan como objetos con `status`/`body` y se capturan en el route handler (patrón repetido en `app/api/**/route.ts`: `catch (err: any) { if (err.status) { return NextResponse.json(err.body, { status: err.status }) } throw err }`).
- Componentes React organizados por dominio en `components/<dominio>/` (`components/casos`, `components/ejecuciones`, `components/grabador`, `components/proyectos`) más `components/ui` para primitivas compartidas.
- Nombres de modelos/campos en español (`CasoPrueba`, `Ejecucion`, `PasoEjecucion`), consistente con el dominio del producto — mantener el idioma al agregar campos nuevos al schema.

## Flujo de ramas

El flujo de ramas, checkpoints y proceso de ejecución de historias de usuario está documentado en [`plan_ejecucion_hus.md`](./plan_ejecucion_hus.md). Es el documento vigente para cómo se organiza el trabajo por HU en este repositorio.
