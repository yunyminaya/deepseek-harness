# Agent Note: Comandos de estado de revisión de PR dirigidos por eventos

Status: implemented

[English](2026-08-10-event-directed-pr-review-status.md) | [中文](2026-08-10-event-directed-pr-review-status.zh.md) | Español

## Problema

El estado del Issue Project registra quién es dueño del siguiente paso del trabajo en resolución. El estado agregado de revisión del pull request responde a si GitHub considera fusionable el pull request, pero no puede representar ese traspaso: una revisión anterior de `CHANGES_REQUESTED` puede seguir siendo efectiva después de que el autor corrija el código y vuelva a solicitar revisión.

Una proyección monótona tampoco puede devolver una Issue propiedad de la automatización de `In review` a `In progress` cuando un revisor solicita cambios. Reconstruir rondas de revisión o bloqueos de revisores añadiría estado que el contrato requerido de dos eventos no necesita.

## Decisión

El flujo de trabajo del ciclo de vida de la Issue trata los webhooks de revisión como comandos. `pull_request.review_requested`, incluida una solicitud repetida, apunta a `In review`. `pull_request_review.submitted` apunta a `In progress` solo cuando `review.state` es `changes_requested`; el evento submitted sigue siendo necesario porque un revisor puede solicitar cambios sin un evento previo de solicitud de revisión. Las presentaciones aprobadas y con comentarios ejecutan su trabajo de ciclo de vida pero no hacen nada (nunca llegan al paso de token del Project), mientras que las revisiones descartadas no están suscritas.

Los eventos ordinarios suscritos de pull request siguen siendo señales de implementación solo hacia adelante: pueden mover `Inbox`, `Backlog` o `Ready` a `In progress`, pero no pueden mover `In review` hacia atrás. Los comandos de solicitud de revisión pueden mover cualquier estado activo anterior a `In review`. Los comandos de cambios solicitados pueden mover estados activos anteriores hacia adelante a `In progress` y pueden mover `In review` hacia atrás solo cuando el evento de estado más reciente del Project destino fue escrito por el actor de ciclo de vida configurado. Un actor humano o desconocido como último actor conserva el estado actual.

El handler resuelve solo referencias exactas de `Fixes`, `Closes` o `Resolves` dentro del mismo repositorio. No altera estados terminales, no añade una Issue sin estado de Project, no depende de la validez de los metadatos del PR, no consulta `reviewDecision`, no reconstruye rondas de revisión, no busca pull requests desde Issues ni ejecuta un reconciliador programado.

El [ciclo de vida de la Issue](../../../../.github/workflows/issue-lifecycle.yml) sigue sin suscribirse a `pull_request.ready_for_review`; ninguno de los comandos de evento depende de esa acción. La [política de la Issue](../../../../.github/workflows/issue-policy.yml) conserva `ready_for_review` porque es dueña de la aplicación de las comprobaciones requeridas cuando un pull request humano entra en revisión.

## Verificación

Las [pruebas de gestión de Issues](../../../../.github/issue-management/policy.test.mjs) fijan el mapeo evento-a-comando, la transición de solicitud de revisión repetida después de un comando de cambios solicitados, la regresión de cambios solicitados, la protección terminal y la preservación del override humano. Las [pruebas de flujos de trabajo](../../../../scripts/ci-workflow.spec.ts) fijan los eventos suscritos, la ausencia de `if` a nivel de trabajo más la puerta a nivel de paso sobre los pasos de token/board (de modo que las revisiones aprobadas y con comentarios pasan sin acuñar un token), y el disparador de política separado de `ready_for_review`.

## Alternativas consideradas

**Derivar el estado de `reviewDecision` o de una ronda de revisión reconstruida.** El agregado de GitHub puede seguir siendo `CHANGES_REQUESTED` después de una solicitud de revisión repetida, mientras que un reductor de rondas introduce semántica de revisores y de orden más allá de los dos traspasos explícitos.

**Conservar la proyección solo hacia adelante.** El avance monótono protege los estados posteriores, pero deja una Issue en `In review` mientras el autor implementa los cambios solicitados.

**Aplicar cada comando de revisión incondicionalmente.** Es el handler de eventos más pequeño, pero permite que la automatización sobrescriba un estado de Project propiedad de un humano. El actor del estado más reciente del Project destino guarda por tanto la única transición hacia atrás.

**Restaurar `ready_for_review` o añadir una cola de debounce.** El estado de ready no lleva ningún traspaso de revisión, mientras que otra cola añade latencia y estado de plano de control sin cambiar ninguno de los dos comandos.

## Consecuencias

Una solicitud de revisión repetida mueve una Issue en resolución gestionada por automatización a `In review` incluso mientras GitHub sigue informando de una revisión de bloqueo más antigua. Una revisión posterior de cambios solicitados la devuelve a `In progress`; la aprobación, los comentarios, el descarte, los pushes y la eliminación de revisores dejan inalterado el estado del comando más reciente.

La proyección sigue siendo dirigida por eventos y no repara un evento que nunca se ejecuta. Reproducir una ejecución antigua del flujo de trabajo puede reproducir su comando antiguo, y ProjectV2 sigue sin ofrecer compare-and-swap atómico entre la lectura del estado más reciente y la mutación. La concurrencia de flujo de trabajo por pull request y la guardia de titularidad humana reducen estas carreras sin introducir estado de ciclo de vida duradero.
