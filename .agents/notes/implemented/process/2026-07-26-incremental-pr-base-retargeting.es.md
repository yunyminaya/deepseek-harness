# Agent Note: Retargeting incremental de las bases de los PR

Status: implemented

[English](2026-07-26-incremental-pr-base-retargeting.md) | Español

## Problema

La base de un PR puede avanzar mientras su tip actual se está fusionando en la rama del PR. Reiniciar desde el tip más nuevo descarta la resolución de conflictos y la validación ya completadas. Reescribir un merge que ya está pusheado borra además historia revisable.

## Decisión

Cuando se elige merge-forward, cada tip de base observado recibe su propio checkpoint de merge. Si la base avanza durante el trabajo, termina y valida el merge ya en curso, haz commit de él y pushéalo cuando la tarea autorice un push. Solo entonces haz fetch y fusiona la base más nueva en un commit de merge separado. No abandones ni reescribas un checkpoint dentro de esa secuencia de merge-forward.

La [decisión de native-stack y rebase opcional](2026-08-02-native-github-stacks-and-optional-rebases.es.md) también permite un rebase protegido por lease para PR independientes o apilados, incluso tras la revisión. Esta nota es dueña solo de la ruta merge-forward. El [skill de aterrizaje de PR apilados](../../../skills/dsh-merging-stacked-prs/SKILL.md) elige cualquiera de las dos historias bajo el [AGENTS.md](../../../../AGENTS.md) raíz, y la [guía de revisión en stack](../../../../docs/cookbook/responding-to-pr-review-on-a-stack.es.md) es dueña de la propagación de correcciones a través de las capas dependientes.

## Alternativas consideradas

**Abortar y reiniciar desde la base más reciente.** Esto descarta los conflictos resueltos y la validación completada, repite trabajo y elimina un punto de recuperación útil.

**Plegar ambos tips de base en un solo merge reescrito.** Esto oculta el orden en que se resolvieron los conflictos y exige reescribir historia remota si el primer merge ya se había pusheado.

## Consecuencias

- Un PR puede llevar varios commits de merge de base cuando su base avanza repetidamente.
- El trabajo completado sigue siendo revisable y recuperable en lugar de descartarse.
- Fusionar una base más nueva cambia el árbol combinado, así que las verificaciones pertinentes vuelven a ejecutarse antes del siguiente push.
