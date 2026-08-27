---
name: dsh-merging-stacked-prs
description: Úsalo al aterrizar una pila de PRs dependientes de GitHub (A ← B ← C, donde cada una se basa en la inferior) sobre master, al fusionar un PR cuya base es la rama de otro PR abierto, o siempre que una petición mencione “stacked PRs”, “PR stack”, “dependent PRs” o fusionar varios PRs relacionados en secuencia. Requiere que toda cadena de dependencias dentro del mismo repositorio use la funcionalidad oficial de stacked PRs de GitHub antes del aterrizaje para que GitHub posea las reglas de toda la pila, la CI, el orden, el retargeting y el estado de merge.
---

# Aterrizar una pila oficial de PRs de GitHub

[English](SKILL.md) | Español

Aterriza PRs dependientes mediante el objeto nativo de stack de GitHub y `gh stack merge`.
No reproduzcas la semántica de una pila fusionando y reapuntando PRs individuales con `gh pr merge` y `gh pr edit`.
El [AGENTS.md](../../../AGENTS.md) raíz posee los historiales permitidos de merge-forward y rebase; la [guía de review de stacks](../../../docs/cookbook/responding-to-pr-review-on-a-stack.es.md) posee la propagación de correcciones de review.

## Exige soporte nativo de stack

Ejecuta `gh stack --version` antes de cambiar estado en GitHub.
Detente por completo si la extensión oficial o la funcionalidad server-side de stack no están disponibles; no recurras a fusionar y reapuntar manualmente los PRs uno a uno.
Los stacks de GitHub requieren que cada rama head viva en el mismo repositorio, así que detente por completo ante una cadena cross-fork.

Usa un worktree dedicado y limpio.
Trae la metadata actual del PR y los OID exactos de head en vez de confiar en nombres de ramas o en un informe anterior:

```sh
gh pr view <pr> --json number,author,baseRefName,baseRefOid,headRefName,headRefOid,isCrossRepository,state,isDraft,reviewDecision,mergeStateStatus,statusCheckRollup
```

Consulta `PullRequest.stack` y `stackEntry.position` para al menos un PR en cada cadena aparente; este objeto oficial de GitHub, y no solo la inferencia por base-branch, es la autoridad de membresía del stack.
Pagina `entries` cuando `size` exceda la página devuelta:

```sh
gh api graphql -F owner=<owner> -F name=<repo> -F number=<pr> -f query='
query($owner: String!, $name: String!, $number: Int!) {
  repository(owner: $owner, name: $name) {
    pullRequest(number: $number) {
      number
      author { login }
      baseRefName
      headRefName
      stackEntry { position }
      stack {
        number
        baseRefName
        size
        entries(first: 100) {
          nodes {
            position
            pullRequest { number author { login } baseRefName headRefName state isDraft }
          }
        }
      }
    }
  }
}'
```

Establece el orden esperado de abajo arriba a partir de las bases vivas de los PRs: el de abajo apunta al trunk, y cada PR superior apunta a la rama head inmediatamente inferior.

## Enlaza miembros faltantes del stack

Primero compara cualquier entrada ya existente en el stack con la cadena esperada.
Un stack existente puede contener un subconjunto que conserve el orden de la cadena solicitada; múltiples números de stack, una entrada inesperada o un orden en conflicto requieren dirección del usuario antes de cualquier mutación.

Cuando cualquier PR dependiente aún no esté en ese stack oficial:

1. Compara exactamente cada `author.login`.
2. Si todos los autores coinciden, enlaza automáticamente la cadena en orden de abajo arriba:

```sh
gh stack link --base <trunk> <bottom-pr> <next-pr> ... <top-pr>
```

3. Si los autores difieren o falta algún autor, pregunta al usuario si quiere enlazarlos antes de cambiar estado en GitHub.
4. Vuelve a consultar GraphQL y exige un único número de stack, el trunk esperado, el conjunto completo de PRs y las posiciones y cadena de bases esperadas.

Nunca disuelvas, reordenes ni reconstruyas automáticamente un stack existente; `gh stack link` es aditivo y las entradas ya fusionadas o en queue no pueden quitarse del stack.

## Refresca solo cuando haga falta

No reescribas ramas solo porque exista un mecanismo de refresco.
Cuando el estado vivo de merge o las reglas del repositorio requieran un trunk actualizado, elige uno de los dos historiales permitidos:

- **Native cascading rebase:** haz checkout del stack remoto con `gh stack checkout <pr-or-stack>` cuando no esté rastreado localmente y luego ejecuta `gh stack sync`.
El comando puede hacer rebase y force-push protegido por lease a cada capa activa antes de la validación local.
Inspecciona inmediatamente el alcance reescrito, ejecuta las comprobaciones relevantes para cada capa afectada y no hagas merge ni afirmes que está listo hasta que pasen.
Si sync detecta un conflicto de rebase, usa `gh stack rebase`, resuélvelo y valídalo, y luego publícalo con `gh stack push`.
Si checkout o sync informan composiciones divergentes entre el stack local y el remoto, cancela y pregunta en vez de borrar o recrear automáticamente el stack remoto.
- **Incremental merge-forward:** fusiona el trunk en la rama inferior afectada y luego propaga cada padre actualizado a su hijo en orden de abajo arriba y haz push de forma normal.
Si la base avanza durante un merge en progreso, conserva ese checkpoint antes de fusionar la punta nueva, como especifica la [nota de retargeting incremental](../../notes/implemented/process/2026-07-26-incremental-pr-base-retargeting.es.md).

Cualquier reescritura de historia está permitida tras la review, pero invalida las suposiciones sobre commit OIDs.
Vuelve a traer los heads exactos y vuelve a auditar los hilos de review no resueltos, las aprobaciones, la mergeabilidad y las checks después del push.
Nunca uses `--force` a secas ni sobreescribas una cabeza remota que haya avanzado en paralelo.

## Haz el preflight del rango de merge

Vuelve a consultar el stack oficial inmediatamente antes de fusionar.
Exige que cada PR seleccionado esté abierto, no sea draft, esté en el orden esperado y cumpla los requisitos de review y checks del repositorio.
Trata el estado de cada PR de forma independiente; que la capa superior esté lista no prueba que sus dependencias también lo estén.

“Land the stack” selecciona el stack completo.
Un aterrizaje parcial requiere un PR límite explícito e incluye cada capa desde la inferior hasta ese límite.

## Fusiona a través de la API de stacks

Fusiona el stack completo por su número oficial de stack:

```sh
gh stack merge <stack-number> --yes --merge
```

Para un aterrizaje parcial solicitado explícitamente, fusiona a través del PR límite:

```sh
gh stack merge <boundary-pr> --yes --merge
```

No pases `--delete-branch`, no reapuntes dependientes manualmente ni emitas comandos de merge por PR individual.
GitHub fusiona el rango seleccionado de abajo arriba y reapunta/hace rebase de cualquier capa superior restante.
Un merge directo del stack es todo-o-nada; cuando el trunk usa merge queue, GitHub pone en cola el rango seleccionado junto, pero puede aterrizarlo en grupos separados.

No eludas los requisitos de merge.
Si el merge nativo informa un bloqueo, inspecciona y resuelve ese bloqueo a través del PR propietario o detente e infórmalo; nunca recurras a `gh pr merge`.

## Verifica el estado aterrizado

Espera a que cada PR seleccionado informe `MERGED`; una petición en queue no es un aterrizaje completado:

```sh
gh pr view <pr> --json number,state,mergedAt,mergeCommit,baseRefName,headRefName
```

Para un aterrizaje parcial, vuelve a consultar el stack oficial y verifica que cada PR restante siga enlazado en el orden esperado y apunte al trunk del stack o a la capa inferior.
Vuelve a comprobar los heads actuales, el estado de review y la CI porque GitHub puede haber hecho rebase de las capas restantes.

Borra ramas solo en una pasada final separada después de que los PRs correspondientes informen `MERGED`.
Antes de borrar cada rama, exige que GitHub informe que ningún PR abierto la siga usando como base:

```sh
gh pr list --state open --base <branch> --json number --jq length
```

Cualquier resultado distinto de `0` bloquea la eliminación.

## Checklist

- [ ] El soporte nativo `gh stack` está disponible; cada rama de PR vive en el mismo repositorio.
- [ ] Las bases vivas de los PRs y los heads exactos establecen una sola cadena de dependencias de abajo arriba.
- [ ] GraphQL informa un solo stack oficial con el trunk, las entradas y el orden esperados; una cadena sin stack del mismo autor que califique fue enlazada automáticamente.
- [ ] Cualquier capa reescrita pasó la validación relevante, y luego se reauditaron los hilos de review, aprobaciones, mergeabilidad y checks.
- [ ] El stack completo, o un prefijo acotado explícitamente, se envió mediante `gh stack merge --yes --merge`.
- [ ] Cada PR seleccionado informa `MERGED`; cualquier capa superior restante sigue formando el stack oficial esperado.
- [ ] La eliminación de ramas ocurrió solo tras verificar estado merged y cero dependientes.
