# HU-3 — Motor de ejecución Playwright (consolidada)

> **Origen**: esta HU consolida las cuatro HU de la Fase 3 del documento [`historias-usuario-playwright-vortex.md`](./historias-usuario-playwright-vortex.md):
> HU-3.1 (disparo de ejecución), HU-3.2 (concurrencia), HU-3.3 (errores de motor) y HU-3.4 (streaming de pasos).
> La consolidación se justifica porque las cuatro comparten el mismo dominio (ejecuciones de Playwright) y el mismo flujo de vida de una ejecución; mantenerlas separadas fragmenta el trabajo sin agregar trazabilidad real.
>
> **Cambio de modelo de datos a aplicar a esta HU**: el script de Playwright **ya no vive como ruta en disco**. La columna `rutaScript` fue eliminada en la migración `prisma/migrations/20260811201617_script_to_db/migration.sql`. El contenido del `.spec.ts` ahora se almacena como texto en `CasoPrueba.script` (campo `@db.Text`) y su nombre original en `CasoPrueba.scriptFileName` (opcional). Al ejecutar, el worker lee `script` desde la BD, lo escribe a un archivo temporal en `os.tmpdir()`, lanza Playwright contra ese archivo y luego lo borra. Esto cambia el sentido literal del AC original "el script asociado al caso no existe" → ahora se interpreta como "`CasoPrueba.script` está vacío o en blanco". El resto de los AC originales se preserva sin cambios.
>
> **Deuda pendiente**: `documentacion/convencion-scripts-playwright.md` (en `playwright_vortex/`) aún describe el modelo antiguo (`rutaScript`, `PLAYWRIGHT_SCRIPTS_ROOT`, `fs.access`). Debe actualizarse en una HU posterior — no es parte de esta HU-3.

---

## Historia

Como usuario superadministrador
Quiero disparar la ejecución de un caso de prueba desde la plataforma, con protección contra ejecuciones simultáneas del mismo caso, con errores del motor claramente diferenciados de los errores de la prueba, y viendo cada paso agregarse al detalle en vivo mientras la prueba corre
Para correr las pruebas reales sin usar la interfaz nativa de Playwright, evitar resultados inconsistentes o conflictos sobre el mismo entorno, diferenciar un error de infraestructura de un error real de la prueba, y entender el progreso real de la ejecución sin tener que esperar a que termine.

---

## Criterios de aceptación (los 11 originales de la Fase 3, preservados 1:1)

Cada AC conserva su origen entre paréntesis `(antiguo HU-x.y / AC-N)`. El orden y el texto son los del documento original; la única reinterpretación está en AC-7 (notada inline) por el cambio de modelo "script en BD".

- *(antiguo HU-3.1 / AC-1)* Dado un caso de prueba registrado, cuando presiono "ejecutar", entonces se crea una ejecución en estado "pendiente" y la petición responde de inmediato (sin esperar a que termine la prueba).
- *(antiguo HU-3.1 / AC-2)* Dado que la ejecución pasa a "corriendo", cuando actualizo la vista, entonces veo el estado cambiar sin recargar manualmente.
- *(antiguo HU-3.1 / AC-3)* Dado que la ejecución termina, cuando reviso su estado, entonces refleja correctamente "pasó" o "falló" según el resultado real de Playwright.
- *(antiguo HU-3.1 / AC-4)* Dado que estoy en la pestaña global **"Ejecuciones"** (`/ejecuciones`), cuando consulto el historial, entonces veo todas las ejecuciones agrupadas por proyecto, con su caso, estado, resultado y fecha.
- *(antiguo HU-3.2 / AC-5)* Dado un caso de prueba con una ejecución en curso, cuando intento dispararlo de nuevo, entonces el sistema me impide crear una segunda ejecución y me indica que ya hay una en curso.
- *(antiguo HU-3.2 / AC-6)* Dado que la ejecución en curso termina (con éxito o error), cuando intento ejecutar de nuevo, entonces el sistema lo permite sin restricciones.
- *(antiguo HU-3.3 / AC-7)* Dado que el script asociado al caso no existe al momento de ejecutar (interpretado en el modelo actual como "`CasoPrueba.script` está vacío o en blanco"), cuando el worker lo intenta correr, entonces la ejecución queda en un estado de error de motor (no "falló"), con un mensaje explicativo.
- *(antiguo HU-3.3 / AC-8)* Dado ese estado de error, cuando reviso el detalle de la ejecución, entonces no aparecen pasos falsos de la prueba (porque nunca llegó a correr).
- *(antiguo HU-3.4 / AC-9)* Dado que una ejecución está corriendo, cuando un paso termina, entonces aparece en la vista de detalle sin que yo tenga que recargar la página.
- *(antiguo HU-3.4 / AC-10)* Dado un paso recién agregado, cuando lo veo, entonces se distingue visualmente como "recién agregado" durante unos segundos (no se confunde con el resto del historial de la ejecución).
- *(antiguo HU-3.4 / AC-11)* Dado que la ejecución terminó, cuando consulto el detalle después, entonces el orden y contenido de los pasos refleja exactamente lo que se emitió durante la corrida (no se "rellena" información que no fue reportada).

---

## Criterios de aceptación complementarios

Estos ACs **no estaban en los originales**; agregan cobertura derivada del diseño actual y del modelo "script en BD". Se listan aparte para no contaminar la trazabilidad de los 11 originales.

- *(complementario, derivado de HU-3.2)* Dado dos casos de prueba distintos, cuando se ejecutan casi en simultáneo, entonces el sistema permite ambas ejecuciones en paralelo (el bloqueo aplica solo al mismo `casoPruebaId`).
- *(complementario, del modelo "script en BD")* Dado que el worker ejecuta un caso, cuando la ejecución termina, entonces el contenido del script que se ejecutó queda conservado en `CasoPrueba.script` aunque el archivo temporal del worker se elimine al terminar.
- *(complementario, del modelo "script en BD")* Dado que el worker lee el script desde la BD y lo escribe a un archivo temporal, cuando comienza la ejecución, entonces no se realiza ninguna lectura desde `PLAYWRIGHT_SCRIPTS_ROOT` ni validación previa con `fs.access` (no existe esa convención en el modelo actual).
- *(complementario, derivado del diseño)* Dado que el worker se detuvo dejando ejecuciones en estado `pendiente`, cuando el worker vuelve a arrancar, entonces retoma esas ejecuciones en su próximo ciclo de polling y las procesa normalmente (no quedan huérfanas).
- *(complementario, derivado de HU-3.3)* Dado que una ejecución queda en `errorMotor` por una causa ajena a la prueba (por ejemplo Playwright no disponible en el contenedor), cuando consulto el detalle, entonces el `errorMsg` distingue esa causa de la causa "script vacío o no ejecutable" del AC-7.
- *(complementario, coherente con HU-2.5)* Dado una ejecución en cualquier estado, cuando se muestra en la lista global o en su detalle, entonces su estado se representa con la `.pill` correspondiente del sistema visual (`p-idle`, `p-running`, `p-pass`, `p-fail`, `p-heal`) o con `.stamp` para `errorMotor`, manteniendo la dirección estética del mockup (`acta-mockups.html`).
- *(complementario, derivado de HU-2.4)* Dado que dejé la vista de detalle abierta mientras una ejecución corre y me fui a otra pestaña, cuando vuelvo a `/ejecuciones/[id]`, entonces el cliente reanuda el polling y refleja el estado y los pasos más recientes (no queda con datos viejos en memoria).
- *(complementario, refuerzo de HU-3.3)* Dado que una ejecución quedó en `errorMotor`, cuando la consulto desde el historial global o desde la pestaña del proyecto, entonces veo un indicador visible que explica la causa (vacío / parse error / Playwright no disponible / otro) sin tener que abrir el log del worker.

---

## Resumen de cobertura

| Bloque | Cantidad | Procedencia |
|---|---|---|
| ACs originales preservados 1:1 | 11 | HU-3.1 (4) + HU-3.2 (2) + HU-3.3 (2) + HU-3.4 (3) |
| ACs complementarios | 8 | Derivados del diseño y del modelo "script en BD" |
| **Total** | **19** | — |

---

## Modelo de datos relevante (referencia, no parte de la HU)

Esta HU **no introduce cambios de schema**. El modelo ya tiene lo necesario:

```prisma
// prisma/schema.prisma (extracto)
model CasoPrueba {
  id             String   @id @default(uuid())
  proyectoId     String
  codigo         String
  nombre         String
  script         String   @db.Text      // contenido completo del .spec.ts
  scriptFileName String?                 // nombre original del archivo
  responsableId  String
  activo         Boolean  @default(true)
  // ...
  ejecuciones Ejecucion[]
}

model Ejecucion {
  id           String          @id @default(uuid())
  casoPruebaId String
  estado       EjecucionEstado // pendiente | corriendo | paso | fallo | reparado | errorMotor
  inicioAt     DateTime?
  finAt        DateTime?
  duracionMs   Int?
  errorMsg     String?
  // ...
  pasos      PasoEjecucion[]
  artefactos Artefacto[]
  acta       Acta?
}

model PasoEjecucion {
  id          String              @id @default(uuid())
  ejecucionId String
  numero      Int
  descripcion String
  estado      PasoEjecucionEstado // paso | fallo | reparado
  duracionMs  Int?
  errorMsg    String?
  createdAt   DateTime            @default(now()) @db.Timestamptz(6)
  // ...
}
```

---

## Componentes principales a construir

Esto es **referencia** para el implementador, no parte de los ACs. Lista los elementos que la HU exige indirectamente:

- **Worker standalone** (`scripts/playwright-worker.ts`): polling cada 5s a `Ejecucion WHERE estado = 'pendiente'`, ejecuta Playwright, parsea eventos, inserta `PasoEjecucion`.
- **Reporter custom** (`scripts/playwright-reporter.js`): emite JSON por stdout línea a línea con `{type, numero, descripcion, estado, duracionMs, errorMsg}`.
- **Endpoint de disparo** (`POST /api/ejecuciones`): recibe `casoPruebaId`, ejecuta `dispararEjecucion()` con lock transaccional `FOR UPDATE NOWAIT` → 409 si hay conflicto.
- **Endpoint de detalle** (`GET /api/ejecuciones/[id]`): retorna ejecución + pasos ordenados por `numero`.
- **Página de detalle** (`app/(dashboard)/ejecuciones/[id]/page.tsx`): server component que carga el estado inicial + componente cliente con polling cada 2s.
- **Página global** (`app/(dashboard)/ejecuciones/page.tsx`): listado agrupado por proyecto.
- **Botón "Ejecutar"** en la fila del caso y en su vista de detalle.

---

## Artefactos SDD relacionados

Estos archivos ya existen en `playwright_vortex/sdd/hu3-motor-de-ejecucion-completo/` y proveen el detalle técnico que esta HU resume a nivel de criterios:

- `explore/exploration.md` — estado del arte al momento de plantear la HU.
- `proposal.md` — decisiones de diseño (worker standalone, polling HTTP, `FOR UPDATE NOWAIT`, reporter custom, validación lazy, `@playwright/test` como devDependency).
- `spec.md` — escenarios BDD detallados (11 ACs originales + escenarios end-to-end).
- `design.md` — arquitectura técnica completa.
- `tasks/tasks.md` — descomposición en tareas de implementación.
- `verify-report.md` — reporte de verificación adversarial.