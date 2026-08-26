# @deepseek-ai/dsh-command-goal

[English](README.md) | Español

Control humano `/goal` sobre [`ctx.goals`](../goal/README.es.md). El plugin registra un comando global mediante [`ctx.commands`](../../interaction/commands/README.es.md), de modo que todo adaptador de comandos compuesto lo descubre y lo ejecuta sin un turno del modelo. La [Agent Note del comando de objetivo humano](../../../.agents/notes/implemented/feature/2026-07-19-human-goal-command.es.md) es la dueña de las decisiones de UX y de composición.

## Contrato del comando

| Entrada | Resultado |
|---|---|
| `/goal` | Muestra el objetivo actual, la fase duradera, el recuento/tope de rondas, la activación local del proceso y los siguientes comandos válidos; un objetivo bloqueado muestra además su código de política y su explicación, mientras que la ausencia de objetivo muestra el uso. |
| `/goal <objective>` | Crea y arma un objetivo, o reemplaza un objetivo completado con una identidad nueva. Un objetivo sin terminar nunca se reemplaza sin un clear explícito. |
| `/goal edit <objective>` | Edita el objetivo actual sin cambiar su fase ni su activación. Editar un objetivo completado crea un objetivo activo nuevo. |
| `/goal pause` | Pausa un objetivo activo y desarma la continuación. |
| `/goal resume` | Reanuda un objetivo detenido o rear ma un objetivo activo después de un resume/fork de sesión, sujeto a su tope de rondas restante. |
| `/goal clear` | Limpia el puntero actual conservando su historial duradero y su tumba. |

Las palabras de control no distinguen mayúsculas solo cuando ocupan la entrada completa. Cualquier otro sufijo no vacío es un objetivo, así que `/goal pause after verification` crea ese objetivo literal. El dominio de objetivo recorta y valida los objetivos. Como el plano de comandos genérico no tiene editor modal ni primitiva de confirmación, `edit` toma su reemplazo en línea y un reemplazo sin terminar devuelve un error directo que indica al usuario que edite o haga clear.

El comando declara `input.images`, de modo que los adjuntos de imagen del compositor pueden acompañar a una invocación. Los adjuntos solo acompañan a un objetivo: en un create o edit con éxito, el productor envía un followup de usuario que lleva los bloques de imagen admitidos más el texto fijo `Reference images for the goal objective.`, de modo que las rondas de objetivo posteriores los leen del historial de sesión ordinario sin que el dominio de objetivo almacene estado de adjuntos. Cualquier otro subcomando, y cualquier create o edit rechazado, devuelve un error directo y no envía nada, así que el compositor que despacha conserva las imágenes.

Los rechazos de dominio esperados se convierten en errores directos estables de comando sin exponer ids ni revisiones con marca. Los fallos de implementación inesperados siguen rechazando el despacho para que los adaptadores puedan informarlos como fallos de comando. El texto y la salida genéricos del comando siguen siendo estado de UI en vivo; `dsh-goal` persiste cada mutación aceptada mediante su propio evento duradero `goal/change`.

## Composición

El productor inyecta `commands` y `goals`. Una app personalizada monta sus propietarios más este plugin; la continuación automática sigue siendo una elección independiente:

```yaml
- id: commands
  name: '@deepseek-ai/dsh-commands'
- id: goal
  name: '@deepseek-ai/dsh-goal'
- id: command-goal
  name: '@deepseek-ai/dsh-command-goal'
```

La base `dsh` distribuida habilita la pila de objetivos persistentes y este comando; el cliente Web aporta su adaptador interactivo. La app de automatización ACP habilita el dominio y las herramientas de modelo sin adaptador de comando; `goals: false` elimina esa pila. El `agent-spine-demo` sin UI exige un `goals: {}` explícito para que los llamantes headless de un solo uso no cambien en silencio de un turno físico a una operación de varias rondas.

## Experiencia de modelo

### Control humano `/goal`

#### Lo que ve el modelo

La entrada de barra, la mutación y la salida directa de estado/error están ausentes de las peticiones al modelo. El dominio de objetivo registra la mutación como `goal/change`; un driver de misma sesión habilitado puede exponer el estado resultante en un prompt de continuación posterior. El texto de presentación nunca se registra. Cuando un create o edit lleva adjuntos de imagen, el modelo ve un mensaje de usuario ordinario: los bloques de imagen seguidos del texto `Reference images for the goal objective.`; precede a la siguiente ronda de objetivo en el historial de sesión.

#### Efecto en tokens

Leer el estado, mutar un objetivo o recibir un error directo de comando no añade tokens de modelo. Un driver de misma sesión habilitado puede añadir prompts de ronda posteriores. Los adjuntos de imagen de un objetivo añaden un mensaje de usuario facturado como cualquier prompt de imagen.

#### Efecto en la KV cache

El descubrimiento de comandos, las mutaciones y la salida directa no afectan a la cache. Los prompts de continuación posteriores siguen el historial de peticiones ordinario del driver.

## Limitaciones conocidas y trabajo pendiente

- **Solo interacción de texto plano** — el registro de comandos genérico no tiene formulario de edición modal ni callback de confirmación de reemplazo; la edición en línea y el clear explícito mantienen la intención destructiva determinista entre adaptadores.
- **Sin argumento de tope de rondas por comando** — `defaultMaxGoalRounds` sigue siendo configuración de despliegue, mientras que una petición humana directa puede pedir al modelo que edite `max_goal_rounds` a través de la herramienta de objetivo autorizada por separado.
- **Sin widget de estado continuo** — el `/goal` pelado es la API de observación portátil; las insignias específicas de adaptador y la salida de comando reconectable siguen siendo trabajo futuro de UI.
- **Adaptador de comando web solo en las apps distribuidas** — los adaptadores headless, de automatización ACP y JSON-RPC no consumen `ctx.commands`. Los prompts ordinarios aún pueden autorizar las herramientas de objetivo orientadas al modelo cuando están compuestas.
