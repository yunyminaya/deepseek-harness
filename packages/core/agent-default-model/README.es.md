# @deepseek-ai/dsh-agent-default-model

[English](README.md) | Español

El valor por defecto del despliegue que se usa cuando un punto de entrada crea un Agent sin selección de modelo local a la sesión. `AgentDefaultModelConfig` proporciona `ctx.agentDefaultModel`; los puntos de entrada directos como `dsh --profile headless` y los puntos de entrada respaldados por Host como ApiProxy leen el mismo servicio en lugar de ser dueños de valores por defecto paralelos de provider/modelo.

La config del plugin exige `{ provider, model }`. Esa entrada de composición es la base de la sección de ajustes de `agent-default-model`; un provider de ajustes montado superpone la elección del usuario sobre ella y los cambios se ven en la siguiente lectura de `currentSelection()`. `reasoningEffort` pertenece a la sección de ajustes pero deliberadamente no a la config del plugin: una selección guardada completa puede limpiar un effort cuando el siguiente modelo seleccionado no tiene ninguno, mientras que un valor de composición se heredaría otra vez.

- `ctx.agentDefaultModel.currentSelection()` devuelve una selección independiente `{ provider, model, reasoningEffort? }` para un Agent recién creado.
- `ctx.agentDefaultModel.saveSelection(selection)` guarda la selección completa del usuario. Sin un provider de ajustes es un no-op y la entrada de composición sigue vigente.

El servicio no valida la pertenencia al catálogo. Una ruta de provider puede atender un modelo no anunciado, y el Consumer que realmente abre una solicitud de modelo es dueño de los diagnósticos de disponibilidad.

## Experiencia del modelo

Indirectamente, a través de la selección de provider/modelo suministrada a un punto de entrada; el ensamblaje de la solicitud y los adaptadores son dueños de la solicitud visible para el modelo.

#### Efecto de KV Cache

Cambiar el valor por defecto solo afecta a los Agents que posteriormente lo resuelven. Una sesión existente cuyo registro de solicitudes ya nombra una selección conserva esa selección, así que este servicio no invalida su prefijo establecido.

## Limitaciones conocidas y trabajo diferido

- El servicio es dueño de un único valor por defecto a nivel de proceso; la selección por sesión sigue siendo responsabilidad del punto de entrada.
- Sin un provider de ajustes, `saveSelection()` no puede retener una selección para un Agent posterior.
