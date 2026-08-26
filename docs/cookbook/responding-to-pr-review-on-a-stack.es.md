# Cómo responder a la revisión en una cadena de PRs apilados

[English](responding-to-pr-review-on-a-stack.md) | Español

Los comentarios de revisión pueden apuntar a varios PRs (pull request) de una pila dependiente (`A ← B ← C …`). Mantén esa cadena enlazada mediante la funcionalidad oficial de PRs apilados de GitHub. Esta guía es dueña de la colocación y la propagación de las correcciones de revisión; el skill [dsh-merging-stacked-prs](../../.agents/skills/dsh-merging-stacked-prs/SKILL.es.md) es dueño de las comprobaciones de enlace y del aterrizaje.

## Reglas básicas

1. **Un worktree por rama de PR.** Las correcciones de cada PR ocurren en el worktree propio de ese PR; las correcciones paralelas nunca comparten un checkout.
2. **El objeto stack de GitHub es autoritativo.** Las ramas base establecen el orden de dependencia esperado, mientras que `PullRequest.stack` y `stackEntry.position` prueban que GitHub lo reconoce. No trates una cadena de ramas coincidente como un stack oficial sin comprobar esos campos.
3. **Una corrección aterriza en el PR que INTRODUJO el problema y luego fluye hacia arriba en el stack.** Cuando un comentario en el PR `B` apunta a código que introdujo `B`, corrígelo en `B` y propaga `B` hacia `C` — incluso si `C` también lleva el archivo. Originar la corrección aguas abajo deja que `B` publique el código sin corregir y le oculta la corrección al revisor de `B`.
4. **Cada corrección de revisión sigue siendo un commit distinto.** Un rebase posterior puede cambiar su OID, pero no hagas amend de una corrección ya revisada fuera del historial de la rama. Haz amend solo de tu propio trabajo aún no enviado ni revisado.
5. **Elige deliberadamente entre merge-forward y rebase.** Tras la revisión se permiten ambos historiales. Un push reescrito debe estar protegido con lease y debe abortar en lugar de sobrescribir una cabeza remota avanzada de forma concurrente; el `--force` en bruto está prohibido.

## Resuelve los comentarios a lo largo del stack

1. Haz triage de cada comentario según sus méritos antes de actuar: verifica la afirmación contra el código — un revisor que señala el síntoma correcto puede seguir diagnosticando mal la causa.
2. Asigna cada hallazgo aceptado a su PR de origen y corrígelo allí.
3. Propaga la capa corregida a través de cada hijo afectado en orden:
   - **Merge-forward:** fusiona la rama padre corregida en su hijo, valida el hijo y continúa hacia arriba. Conserva cada punto de control en curso bajo la [decisión de redestino incremental](../../.agents/notes/implemented/process/2026-07-26-incremental-pr-base-retargeting.es.md).
   - **Rebase en cascada nativo:** usa `gh stack rebase`, valida las capas reescritas y luego publica con `gh stack push`; o usa `gh stack sync`, que puede publicar primero y por tanto exige una validación inmediata posterior a la sincronización bajo [dsh-pre-push-checks](../../.agents/skills/dsh-pre-push-checks/SKILL.es.md).
4. Trata las correcciones delegadas como «confiar pero verificar»: el informe de un subagente describe la intención, no necesariamente lo que aterrizó. Vuelve a ejecutar tú mismo los gates sobre el árbol real y, para un guard de regresión, demuestra que FALLA con el código sin corregir (introduce la regresión, observa el rojo, revierte) — un guard que pasa en ambos sentidos no protege nada. Un subagente que reformula un problema como ya resuelto es una señal para que profundices personalmente.
5. Responde en el hilo de la revisión (`gh api repos/{owner}/{repo}/pulls/{pr}/comments/{id}/replies`), no como comentario de nivel superior, indicando la corrección y el commit o head actual que la lleva.
6. Después de cualquier push reescrito, vuelve a leer los hilos sin resolver, las aprobaciones, la capacidad de fusión y las comprobaciones. Un OID de commit con force-push o una ancla en línea desactualizada no son evidencia actual de que el hallazgo siga resuelto.
7. Aterriza solo mediante el procedimiento oficial de stacks. Si los PRs aún no están enlazados, el skill de aterrizaje enlaza automáticamente una cadena del mismo autor, pregunta antes de enlazar autores mixtos y se detiene en seco cuando el soporte nativo de stacks no está disponible.

## Verificación

- El diff actual de cada PR corregido contiene la corrección prevista en la capa que introdujo el problema.
- GraphQL informa de un único stack oficial en el orden esperado, y el diff de cada hijo contra su padre muestra solo los cambios de ese hijo.
- Los hilos sin resolver, las aprobaciones, la capacidad de fusión y las comprobaciones se reauditaron después de cada push reescrito.
- Los gates pertinentes pasan en cada PR afectado del stack, no solo en el superior.
