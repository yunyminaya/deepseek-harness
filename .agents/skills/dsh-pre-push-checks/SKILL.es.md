---
name: dsh-pre-push-checks
description: Úsalo antes de hacer push, force-push, marcar “ready for review” o afirmar que las checks pasan en una rama de deepseek-harness, e inmediatamente después de que gh stack sync publique ramas reescritas, para seleccionar las pruebas y checks más pequeñas que cubran el diff saliente o recién publicado sin ejecutar por reflejo toda la suite del repositorio.
---

# Checks previos al push en DSH

[English](SKILL.md) | Español

Usa este skill para ejecutar una sola vez evidencia local relevante antes de un push de `deepseek-harness`.
La única excepción de orden es `gh stack sync`, que puede publicar un rebase en cascada antes de que las capas reescritas puedan validarse; valídalas inmediatamente después y no hagas merge hasta que la evidencia pase.
Los hooks de git son deliberadamente estrechos: pre-commit corrige el lint staged, comprueba whitespace staged y protege la metadata del código vendorizado; pre-push ejecuta solo el typecheck incremental del repositorio.
La CI posee la cobertura exhaustiva y la matriz de plataformas.

## Inspecciona el cambio saliente

1. Confirma el checkout y la rama.

```sh
git status --short --branch
git rev-parse --show-toplevel
```

2. Verifica la base viva del PR o el parent del stack, trae esa ref e inspecciona el alcance completo contra ella.

```sh
pnpm --silent run change-scope --base <verified-base-ref>
```

El comando nunca adivina ni trae una base por sí solo.
Proporciona la ref verificada a partir del estado remoto o del stack actual; usa `--head <ref>` cuando inspecciones un commit distinto de `HEAD`.
Su JSON versionado registra rutas commiteadas relativas a la merge base resuelta, mientras que las rutas staged, unstaged y untracked describen el worktree actual.
Tras fusionar una base cambiada, vuelve a ejecutar el informe, reevalúa qué comportamiento puede afectar el alcance combinado y vuelve a ejecutar solo las checks invalidadas por ese merge.

## Selecciona evidencia relevante

No existe una baseline local universal más allá de los hooks.
Cada cambio de comportamiento necesita la prueba más estrecha disponible o la comprobación diseñada a propósito que fallaría ante esa regresión; añade checks más amplias solo para las superficies que el diff realmente toca.

- **Comportamiento de paquete o script:** ejecuta el archivo Vitest propietario o el nombre de prueba enfocado.
Añade tests de paquetes adyacentes cuando cambie un contrato compartido; deja la cobertura a nivel de repositorio a la CI salvo que el cambio sea genuinamente transversal o el usuario la pida.
- **Documentación, Agent Notes, catálogos o comentarios enlazados a docs:** ejecuta `pnpm run doc-sync`; ejecuta el lint completo cuando el flujo de documentación lo requiera.
- **Salida visible para modelo, editor, CLI o terminal:** ejecuta el snapshot sin key enfocado o el escenario real runnable-example que posea esa salida.
- **Package manifests, exports públicos, configuración de build, entradas de worker/bin o rutas de runtime ya construido:** ejecuta `pnpm run build`, las checks de higiene relevantes y el smoke del artefacto construido propietario.
- **Comportamiento real de provider o agent:** ejecuta el target relevante de `pnpm run test:e2e` cuando haya credenciales disponibles; nunca imprimas secretos.

No repitas manualmente una check que ya pasó solo porque luego vengan commit o push.
En particular, no ejecutes typecheck inmediatamente antes del push solo para duplicar el hook pre-push.

### Enfoca la cobertura de unidad en el código fuente afectado

La selección de tests y la selección de cobertura son cosas separadas.
Un filtro de archivo Vitest elige qué tests se ejecutan, mientras la configuración del repositorio mide por defecto todos los archivos `packages/*/*/src/**/*.ts`.
Cuando la cobertura de unidad sea relevante, nombra tanto los tests propietarios como los archivos fuente o el paquete cuya cobertura deben probar esos tests:

```sh
pnpm exec vitest run packages/<group>/<package>/tests/<behavior>.spec.ts \
  --coverage \
  --coverage.include='packages/<group>/<package>/src/**/*.ts'
```

Usa un archivo fuente exacto cuando el comportamiento de verdad esté confinado a un solo módulo.
Repite `--coverage.include` para varios archivos o paquetes afectados y pasa cada archivo de tests propietario necesario para ejercer ese alcance.
Los umbrales configurados de 100% por archivo siguen aplicándose dentro del alcance fuente seleccionado.

Cuando los tests propietarios no estén claros, usa el grafo de dependencias de Vitest para descubrir un conjunto candidato y luego inspecciona los tests seleccionados antes de tratarlos como evidencia:

```sh
pnpm exec vitest related packages/<group>/<package>/src/<changed>.ts \
  --run \
  --coverage \
  --coverage.include='packages/<group>/<package>/src/<changed>.ts'
```

`vitest related` no puede descubrir comportamiento alcanzado solo a través de configuración, carga dinámica, subprocesses, workers, artefactos construidos o providers externos; selecciona explícitamente esos tests propietarios.
No uses `--passWithNoTests`, no bajes umbrales de cobertura ni estreches `--coverage.include` solo para ocultar un archivo afectado sin cobertura.
Si un alcance de paquete seleccionado falla porque un test enfocado no lo cubre, añade sus otros tests propietarios relevantes o estrecha el alcance fuente solo cuando los módulos excluidos no puedan verse afectados por el cambio.

## Ensayo local completo

Ejecuta la aproximación local completa solo cuando el usuario la pida explícitamente, mientras diagnosticas un fallo de CI, o cuando el cambio abarque el repositorio tan ampliamente que ningún conjunto más estrecho sea creíble.
Usa el workflow actual y los scripts de paquete como inventario; no recrees el aggregate eliminado `check:pre-push`.

## Protege pushes que reescriben historia

Se permite rebase para ramas de PR independientes y apiladas, incluso tras la review.
Antes de una reescritura de historia en una rama independiente, trae la rama remota actual y registra su OID exacto; publícala con `--force-with-lease=<branch>:<observed-oid>` para que un update concurrente aborte el push.
`gh stack push` y `gh stack sync` ya aportan protección por lease para sus ramas gestionadas.
Nunca se permite `--force` a secas.

Después de cualquier push reescrito, vuelve a traer las heads vivas y reaudita los hilos de review no resueltos, las aprobaciones, la mergeabilidad y las checks.
Los commit hashes y anclas de comentarios inline anteriores a la reescritura ya no son evidencia actual.

### Validación post-sync

`gh stack sync` hace fetch, cascade-rebase y push como una sola operación, por lo que no puede insertar validación local entre reescritura y publicación.
Antes de ejecutarlo, exige worktree limpio y registra el orden oficial del stack y las heads remotas exactas.
Después de que termine:

1. Vuelve a consultar cada head de rama y el orden oficial del stack en GitHub.
2. Inspecciona el alcance cambiado de cada capa reescrita contra su base viva del PR.
3. Ejecuta la evidencia relevante seleccionada por este skill para cada capa afectada.
4. Mantén todos los PRs sin fusionar e informa la validación como pendiente hasta que pasen todas las checks seleccionadas.

Si la evidencia post-sync falla, deja en su sitio las heads publicadas protegidas por lease, repara el fallo, valida la reparación y publica la corrección.
No afirmes que el sync dejó el stack listo solo porque el comando tuvo éxito.

## Gestiona los fallos

Si una check relevante falla antes de un push ordinario, detente y corrige o explica el bloqueo.
No hagas push esperando que la CI sea distinta.
Para la excepción post-sync, bloquea el merge y sigue el procedimiento de reparación anterior.

Si un fallo parece específico del entorno, demuéstralo:

- Registra el comando exacto, la prueba fallida y la discrepancia específica de plataforma.
- Confirma la evidencia relevante no específica de plataforma.
- Prefiere corregir la no determinación cross-platform cuando la check sea requerida.
- Elude un hook local solo cuando el usuario lo pida o acepte explícitamente, e informa exactamente qué falló y por qué se espera que la CI difiera.

## Procedimiento de push

Para pushes ordinarios y pushes independientes con rebase:

1. Ejecuta una sola vez las checks relevantes seleccionadas.
2. Haz commit normalmente e inspecciona cualquier archivo cambiado por el fixer del pre-commit antes de continuar.
3. Haz push normalmente, o usa el lease exacto para una rama reescrita autorizada, de modo que se ejecute el hook incremental de typecheck.
4. Verifica que la ref remota coincida con el `HEAD` local.

```sh
git rev-parse HEAD origin/$(git branch --show-current)
```

Para PRs de GitHub, inspecciona la CI remota después del push:

```sh
gh pr checks
```

Informa las checks pendientes como pendientes.
Inspecciona los fallos antes de atribuirlos a la rama o al entorno.

Cuando `gh pr checks` informe "no checks reported" y `/actions/runs?head_sha=<sha>` devuelva `total_count: 0`, lee la mergeabilidad antes de sospechar del push o de un evento perdido de GitHub:

```sh
gh pr view <number> --json mergeable,mergeStateStatus
```

GitHub no crea ejecuciones de workflow `pull_request` mientras un PR esté `CONFLICTING`/`DIRTY`, así que la ausencia de señal es el conflicto, no la infraestructura.
Resolver el conflicto es la única solución; commits vacíos, pushes con `--allow-empty`, toggles de draft/ready y rebotes revert-and-restore dejan `total_count` en cero y añaden historia basura.
Confirma las rutas en conflicto con `git merge-tree --write-tree HEAD origin/<base>` cuando la rama aún no pueda fusionarse localmente.

Para `gh stack sync`, usa la secuencia de validación post-sync en lugar de fingir que el orden ordinario era posible.
