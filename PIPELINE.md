# Estándar de CI/CD — todos los proyectos Logwell

> Establecido 2026-08-22, a partir de la auditoría de consumo de Actions en
> `hubwell_react`. Aplica a todo repo nuevo y a toda migración.

## El problema que resuelve

`hubwell_react` gastaba **~2,100 minutos facturados cada 5 semanas** contra una
cuota de 2,000/mes, y podía fundir un mes en un día de trabajo intenso. Las
causas, medidas y en orden de tamaño:

| Causa | Costo medido | Regla que la mata |
|---|---|---|
| Tres workflows corriendo el mismo typecheck+build | ~1,480 min | **R1** — un solo CI |
| `push` **y** `pull_request` en el mismo workflow | ×2–3 | **R2** — CI solo en PR |
| Jobs de 4–13 s con VM propia | ~650 min | **R3** — un job por workflow |
| Dependabot sin agrupar | ~450 min | **R4** — mensual y agrupado |
| Rebuild en frío en cada deploy | 4.5 min/deploy | **R5** — cache en registro |
| Construir en el camino a producción | 6m33s de reloj | **R6** — promover, no construir |
| Imágenes GHCR sin retención | drenaje invisible | **R7** — conservar 10 |

## Las siete reglas

### R1 · Un solo workflow de CI por repo
Un `ci.yml`, no `web-ci` + `api-ci` + `quality-gate` haciendo lo mismo. Si dos
workflows corren `tsc --noEmit` sobre el mismo código, uno sobra.

### R2 · El CI corre **solo** en `pull_request`
El PR ya prueba exactamente el árbol que se va a mergear. Volver a correrlo en
el push a la rama de integración y otra vez en el push a producción es probar
el mismo commit tres veces. La red de seguridad de producción es el
**health-gate + rollback automático** del deploy, no un cuarto typecheck.

### R3 · Un solo **job** por workflow
**GitHub factura cada job redondeado hacia arriba al minuto.** Siete jobs de
40 s cuestan 7 minutos; el mismo trabajo secuencial en un job cuesta 3. Esta
regla es contraintuitiva —parece que el paralelismo es gratis— y era la mitad
del gasto. Nunca un job para correr un `if` o un `date`.

Corolario: nada de `services:` a nivel de job para dependencias que solo usa
una parte del CI. Se levantan con `docker run` dentro del paso que las
necesita, para no pagarlas en los PRs que no las tocan.

### R4 · Nada corre si no cambió
Detectar qué cambió con la API de GitHub (`pulls/N/files`, `compare/A...B`) —
no con `fetch-depth: 0`, que en un repo de 100 MB cuesta ~30 s por run— y
saltar por app. Un PR de solo documentación debe terminar en ~30 s.

### R5 · Cache de build **en el registro**, nunca `type=gha`
`cache-from: type=gha` se desaloja a los **7 días** sin uso y compite por un
tope de 10 GB por repo que dos imágenes en `mode=max` saturan desalojándose
entre sí. Medido en hub el 2026-08-22: no quedaba un solo layer de buildx. Usar:

```yaml
cache-from: type=registry,ref=ghcr.io/OWNER/IMG:buildcache
cache-to:   type=registry,ref=ghcr.io/OWNER/IMG:buildcache,mode=max
```

No caduca, no compite por ese tope, y lo comparten todas las ramas.

### R6 · Producción **promueve**, no construye
Éste es el que baja el deploy de 6m33s a ~1 min.

```
rama de trabajo ──PR──► development ──FF──► main
   ci.yml (~2 min)      build.yml            deploy.yml
   typecheck/tests      construye y publica  pull + up + health
                        las imágenes          (SIN build)
```

El FF conserva el SHA, así que cuando el commit llega a `main` su imagen **ya
está en GHCR**. `deploy.yml` solo hace pull. Si el SHA no tiene imagen (hotfix
directo a `main`), construye como respaldo con el mismo cache.

La imagen que no cambió no se reconstruye: se le **copia el manifiesto** al tag
nuevo (`docker buildx imagetools create`, ~3 s, no mueve bytes), que preserva
la invariante de desplegar todos los servicios al mismo tag.

### R7 · Retención en GHCR
El almacenamiento de packages privados se cobra contra **el mismo spending
limit que los minutos**. Sin retención, el presupuesto se drena por un lado que
no aparece en la factura de "Actions". Conservar 10 versiones + todos los `v*`.
Ver `gc-packages.yml`. **No borrar versiones sin tag**: el `:buildcache`
depende de ellas.

## Presupuesto por feature

| | Antes | Ahora |
|---|---|---|
| Push a la rama del PR | 9 min | 2–3 min |
| Merge a `development` | 9 min | 3–4 min (construye) |
| Promoción a `main` | 23 min | 2 min |
| **Total por feature** | **~41 min** | **~8 min** |

## Workflows reutilizables

- `ci-node.yml` — CI de un job (R1+R3). Llamar solo desde `pull_request` (R2).
- `gc-packages.yml` — retención GHCR (R7).
- `deploy-prod.yml`, `rollback.yml`, `e2e-smoke.yml` — sin cambios.
- ~~`ci-node-next.yml`~~ — **obsoleto**: cinco jobs, viola R3. Migrar a `ci-node.yml`.

## Al migrar un repo

1. Fusionar los CI en `ci.yml`, `on: pull_request` únicamente.
2. Un solo job; detección de cambios por API.
3. Separar `build.yml` (rama de integración) de `deploy.yml` (producción).
4. Cambiar `type=gha` por `type=registry` en todos los `cache-from/to`.
5. `.github/dependabot.yml`: `interval: monthly` + `groups` + límite de PRs.
6. Programar `gc-packages`.
7. **Actualizar la protección de ramas al nombre del nuevo job** — si quedan
   required checks de workflows borrados, ningún PR vuelve a mergear jamás.
