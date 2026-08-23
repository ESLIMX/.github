# Estándar de CI/CD — todos los proyectos Logwell

> Establecido 2026-08-22, revisado 2026-08-23 a partir de medir el consumo real
> de Actions en `hubwell_react`. Aplica a todo repo nuevo y a toda migración.

## De dónde sale

`hubwell_react` gastaba **~2,100 min facturados cada 5 semanas** contra una
cuota de 2,000/mes. Al desglosar las 185 corridas de un periodo de 5 semanas
apareció que el gasto **no venía del trabajo real**:

| Origen | Corridas | |
|---|---|---|
| PRs de Dependabot | 78 | 42 % — cada rebase del bot relanzaba el CI entero |
| `push` a `main` | 38 | 21 % — re-probaba lo que el PR ya había probado |
| `push` a `development` | 31 | 17 % — lo mismo, otra vez |
| **PRs de features reales** | **~17** | **9 %** |

Nueve por ciento. Todo lo demás era duplicación o ruido del bot. La lección
general: **antes de optimizar un pipeline, desglosa quién lo dispara** — es
probable que el trabajo real sea la minoría.

## Las ocho reglas

### R1 · Un solo workflow de CI por repo
Si dos workflows corren `tsc --noEmit` sobre el mismo código, uno sobra.

### R2 · El CI corre **solo** en `pull_request`
El PR ya prueba el árbol que se va a mergear. Repetirlo en el push a la rama de
integración y otra vez en el de producción es probar el mismo commit tres
veces. La red de seguridad de producción es el **health-gate + rollback
automático** del deploy, no un cuarto typecheck.

### R3 · Se **compila una sola vez** por feature, y el resultado es el artefacto de producción
La regla que más trabajo elimina. `next build` y `nest build` **ya
typechequean**; un `tsc --noEmit` aparte compila el proyecto dos veces. Y si el
CI compila para "verificar" y luego el deploy vuelve a compilar para "empaquetar",
son tres compilaciones del mismo código.

**El `docker build` del CI *es* el gate de tipos, y lo que produce es la imagen
que se va a desplegar.**

### R4 · Etiquetar las imágenes por **hash de contenido**, no por SHA de commit
Un script único calcula el hash de las entradas de build (`git ls-tree` sobre
las rutas del context, que es direccionado por contenido). Esto compra dos
cosas:

1. El merge por **squash** crea un commit nuevo con SHA distinto, pero el
   **árbol** es el mismo ⇒ mismo hash ⇒ producción reutiliza tal cual la imagen
   que construyó el PR. Sin esto habría que reconstruir al integrar.
2. Si el hash ya existe en el registro, **no se construye nada**: un rebase o un
   push que solo toca `docs/` no cuesta un minuto.

El script debe ser **uno solo**, compartido por CI y deploy. Si cada uno
calcula el hash por su lado, cualquier divergencia hace que producción no
encuentre la imagen y la reconstruya — justo lo que se quería evitar.

### R5 · Producción **promueve**, no construye
```
rama ──PR (ready)──► integración ──FF──► producción
      ci.yml                             deploy.yml
   compila, prueba y                   retag + pull + up
   PUBLICA la imagen                   ~1 min, sin build
```
Deploy con respaldo: si falta la imagen (trabajo integrado sin PR), se
construye ahí. El deploy sigue siendo correcto, solo tarda más — y ése es el
incentivo para abrir el PR.

Cuando las imágenes tienen hashes distintos entre sí y el wrapper de deploy
espera un solo tag, se les pone un tag común por commit copiando el manifiesto
(`docker buildx imagetools create`): ~3 s, no mueve bytes, y deja un tag
inmutable por deploy para el rollback.

### R6 · Iterar en **PR borrador**; el CI corre al marcarlo listo
```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
jobs:
  plan:
    if: github.event.pull_request.draft != true
```
Un draft no gasta nada. Se pushea sin miedo (el push es respaldo) y el CI corre
**una vez**, cuando el trabajo está listo. Es el mayor recorte disponible y no
cuesta calidad: lo que se deja de correr son las verificaciones de estados
intermedios que nadie iba a desplegar.

### R7 · Nada corre si no cambió — y nada arranca "por si acaso"
Un job `plan` decide y los caros ni arrancan: **un job saltado por `if:` cuesta
0**. Detectar cambios con la API (`pulls/N/files`, `compare/A...B`), no con
`fetch-depth: 0`.

Corolario que se pasa por alto: **no pongas `services:` a nivel de job**.
Arrancan siempre, incluso cuando el paso que los usaba está saltado. En
`hubwell_react` había Postgres + Redis en cada corrida para 34 specs que son
unitarios con mocks y **ninguno instancia PrismaClient**. Lo que de verdad
necesite una base real se levanta con `docker run` dentro de su propio job.

### R8 · Retención en el registro
El almacenamiento de packages **privados** se cobra contra el **mismo spending
limit que los minutos**, así que el presupuesto se drena por un lado que no
aparece en la factura de "Actions". Conservar 10 versiones + todos los `v*`
(`gc-packages.yml`). **No borrar versiones sin tag**: el `:buildcache` depende
de ellas.

## Sobre el paralelismo entre jobs

**GitHub factura cada job redondeado hacia arriba al minuto.** Siete jobs de
40 s cuestan 7 minutos; el mismo trabajo secuencial en uno cuesta 3. Por eso el
default es **un job por workflow**, y por eso nunca se dedica un job a correr
un `if` o un `date`.

Pero es un default, no un dogma. Cuando el consumo baja al ~10 % de la cuota,
el recurso escaso deja de ser el minuto y pasa a ser el **tiempo de espera**;
ahí conviene separar los builds pesados en jobs paralelos y pagar el redondeo.
La decisión se toma con el número de consumo en la mano, no por costumbre — y
si el consumo vuelve a apretar, lo primero que se colapsa es esa separación.

## Presupuesto por feature

| | Antes | Ahora |
|---|---|---|
| Iterar en el PR | 9 min por push | **0** (draft) |
| CI del grupo, una vez | — | ~5 min |
| Merge a integración | 9 min | **0** |
| Promoción a producción | 23 min | ~2 min |
| **Total** | **~41 min** | **~7 min** |
| **Espera de "listo" a producción** | ~10 min | **~1 min** |

## Workflows reutilizables

- `ci-node.yml` — CI de un job para repos sin imagen Docker. Llamar solo desde
  `pull_request` (R2).
- `gc-packages.yml` — retención en el registro (R8).
- `deploy-prod.yml`, `rollback.yml`, `e2e-smoke.yml` — sin cambios.
- ~~`ci-node-next.yml`~~ — **obsoleto**: cinco jobs, viola R1 y el default de
  un job. Migrar a `ci-node.yml`.

Para repos que construyen imagen, `hubwell_react` es la referencia:
`ci.yml` + `deploy.yml` + `scripts/image-hash.sh`.

## Al migrar un repo

1. Fusionar los CI en `ci.yml`, `on: pull_request` únicamente, con `draft` fuera.
2. Quitar todo `tsc --noEmit` que duplique el build (R3).
3. Quitar los `services:` que no use el 100 % de las corridas (R7).
4. `scripts/image-hash.sh` + tags `src-<hash>` (R4); deploy que promueve (R5).
5. `type=gha` → `type=registry` en todos los `cache-from/to`.
6. `.github/dependabot.yml`: `monthly` + `groups` + límite de PRs. Es el
   disparador #1 en repos con Dependabot suelto.
7. Programar `gc-packages` (R8).
8. **Actualizar la protección de ramas al nombre del nuevo job** — si quedan
   required checks de workflows borrados, ningún PR vuelve a mergear jamás.
