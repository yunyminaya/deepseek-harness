# @deepseek-ai/dsh-plan-mode

[English](README.md) | Español

Estado de colaboración de plan registrado en el log y propio de cada agent, con guía propiedad del despliegue, comandos directos de entrada `/plan [message]` y de salida `/plan off`, y la salida revisada `exit_plan_mode`. El plan mode es guía blanda; el sandbox mode y la política de aprobación imponen las restricciones de forma independiente y no leen ni escriben el estado de plan.

## Estado durable

`plan/mode` (`{ active: boolean }`) es un miembro de `SessionEventMap` solo de log y de reemplazo de valor completo. `foldPlanMode(events)` devuelve el último valor registrado o `false`, así que reanudar, bifurcar y compactar recuperan el estado de plan directamente del log de la sesión. Las UIs observan los cambios confirmados a través de `session/event`.

`ctx.planMode.set(agent, active)` añade el evento independiente `plan/mode` inmediatamente cuando el agent está inactivo, porque no hay ningún pre-step dentro del turno antes del siguiente prompt. Mientras el agent se está ejecutando, mantiene una selección pendiente para el siguiente pre-step dentro del turno aceptado. Devuelve cuál ocurrió (`committed`/`queued`), una reversión `cancelled` o un `noop`. `get(agent)` devuelve `{ active, pending? }`, separando el estado registrado usado para ensamblar el paso actual de una selección del usuario a mitad de turno. Los pre-steps iniciales y de continuación aplican ambos las selecciones pendientes; un reintento de recuperación de solicitud del mismo paso reutiliza su ensamblaje congelado y deja la selección pendiente para el siguiente pre-step. Una selección de usuario cambiada aporta un aviso `user/message` de fuente de plugin cuando la última cabecera de solicitud registrada describía el otro estado (ambas rutas de confirmación).

## Interacciones del modelo y humanas

Mientras está activo, `plan:policy` renderiza la `section` configurada. El plugin registra siempre `exit_plan_mode`, manteniendo estables los schemas de herramienta a través de la transición; su ruta de ejecución acepta solo plan mode activo y solo lo abandona tras una aprobación exacta del usuario a través de `ctx.userQuestions`.

La pregunta de revisión declara la intención de presentación `plan-review`, nombrando `Approve` como la etiqueta que la aprueba, de modo que una UI capaz presenta el plan como una decisión en lugar de una pregunta genérica; la respuesta que lee la herramienta es la misma en ambos casos. Una revisión descartada — el usuario cerrando la solicitud de hablar en su lugar — se informa al modelo como tal, diciéndole que permanezca en plan mode y espere el mensaje; cualquier otro fallo de revisión conserva el mensaje propio del seam.

Cuando `ctx.commands` está compuesto, el paquete registra `/plan [message]` y reserva el argumento exacto `off` para la salida directa. `/plan` a secas selecciona el plan mode; cualquier otro argumento no vacío lo selecciona primero y después se envía mediante `agent.steer()`, de modo que se convierte en el mensaje de usuario ordinario registrado del siguiente paso bajo la guía de plan. `/plan off` selecciona inactivo sin enviar entrada al modelo; también cancela una entrada pendiente antes de que el plan mode alcance una solicitud. El comando declara `input.images`: los attachments de imagen del composer viajan en el mensaje dirigido (steered), delante de su bloque de texto. `/plan` a secas con imágenes dirige un mensaje de usuario solo de imágenes, mientras que `/plan off` con imágenes devuelve un error directo antes de cualquier cambio de modo para que el composer las conserve.

El cliente Web consume el comando `/plan` propiedad del plugin; otros puntos de entrada pueden conducir el mismo servicio directamente sin definir un segundo vocabulario de modos.

## Proyección de sesión

Cuando la composición monta `ctx.sessionProjections` ([`@deepseek-ai/dsh-session-projection`](../../session/session-projection/README.es.md)), este paquete registra la unidad de proyección `plan` bajo un hijo inyectado. Un registro `command/run` llamado `plan` con `args` registrados inicia un objetivo candidato (`off` → inactivo, cualquier otra cosa → activo); su `command/done` emparejado retiene una selección exitosa y descarta un error; `plan/mode` confirma el estado registrado y limpia la selección retenida. Cualquier otro evento devuelve la misma referencia de estado. `view` deriva `{ active, pending }`, donde `pending` es true solo mientras una selección sin resolver o exitosa difiere del estado registrado. Esto sigue siendo una cantidad de repetición pura, así que los reinicios del host, otras pestañas y las lecturas en frío la recuperan solo del log, y un `/plan off` rechazado con imágenes no puede dejar una salida pendiente. La clave se fusiona en `SessionProjectionMap` desde `src/types.ts` (servida a los consumidores del host mediante `./types` y a los agregados del cliente mediante `./client`); el framework conduce la unidad y los carriers sirven el valor en la página de cola del historial y en el frame de push `session/projection`. Las composiciones sin el registro no se ven afectadas.

## Configuración

```yaml
- id: plan-mode
  name: '@deepseek-ai/dsh-plan-mode'
  config:
    section: |
      You are in plan mode. Explore and design before presenting the complete
      plan through exit_plan_mode.
```

`section` es obligatoria y no vacía. Las claves desconocidas fallan en la carga. El paquete no acepta modos con nombre arbitrarios, filtros de herramientas, ajustes de sandbox ni política de aprobación.

Diseño: [estado de colaboración específico de plan](../../../.agents/notes/implemented/simplification/2026-07-22-plan-specific-collaboration-state.es.md).

## Experiencia del modelo

### Prompt de política de plan

#### Qué ve el modelo

Mientras el plan mode está activo, el modelo ve el texto exacto de la `section` del despliegue en el orden de prompt 50; el modo inactivo no aporta texto.

##### Ejemplo de configuración

```markdown
You are in plan mode. Explore and design before presenting the complete plan through exit_plan_mode.
```

#### Efecto de tokens

El modo inactivo no añade tokens; el modo activo añade la sección configurada a cada solicitud.

#### Efecto de KV Cache

La sección es estable dentro del plan mode, pero entrar o salir cambia el system prompt a partir del orden 50.

### Comando humano

#### Qué ve el modelo

`/plan`, `/plan off` y sus resultados terminales permanecen fuera del historial del modelo. Un sufijo no vacío distinto del argumento exacto `off` se convierte en un mensaje de usuario mediante `agent.steer()` después de seleccionar el plan mode: los attachments de imagen admitidos como bloques de imagen iniciales y después el bloque de texto recortado. `/plan` a secas con imágenes admitidas dirige un mensaje de usuario que contiene solo esos bloques de imagen. Una selección activa de `/plan off` aporta el aviso estándar registrado de cambio de usuario solo cuando la última cabecera de solicitud describía el plan mode; cancelar una entrada pendiente no aporta ninguno porque ninguna solicitud la observó.

#### Efecto de tokens

El mensaje opcional cuesta los mismos tokens de historial que enviar ese contenido por separado. `/plan` a secas sin imágenes y `/plan off` no añaden ninguno; `/plan` a secas con imágenes tiene el coste normal de prompt de imagen. Una salida activa narrada añade el pequeño aviso de cambio retenido.

#### Efecto de KV Cache

El bloque de usuario es crecimiento de conversación solo de adición. Entrar o salir del plan mode cambia la sección de política anterior; un aviso de salida narrado se añade después del prefijo de solicitud reutilizable.

### Schema de la herramienta de salida y el intercambio de revisión

#### Qué ve el modelo

El [schema de `exit_plan_mode`](../../../docs/tool-catalog.es.md#deepseek-aidsh-plan-mode) permanece disponible en ambos estados; la ejecución fuera del plan mode falla, mientras que una revisión aprobada dentro del modo devuelve el valor canónico `{ approved: true }` y renderiza el texto de confirmación existente. El rechazo sigue siendo una llamada fallida con retroalimentación de revisión, y una revisión descartada, una llamada fallida que nombra la toma de control del usuario.

#### Efecto de tokens

El schema estable se paga según el modo de ToolRuntime, y cada argumento de plan y resultado de revisión permanecen en el historial de la conversación.

#### Efecto de KV Cache

Las transiciones de modo no cambian el catálogo de herramientas; los argumentos de plan y los resultados de revisión extienden la conversación con normalidad.

## Limitaciones conocidas y trabajo diferido

- El plan mode guía en lugar de imponer; los despliegues que necesitan restricciones impuestas deben configurar los controles de sandbox y de aprobación de forma independiente.
- Una selección hecha después del pre-step final aceptado del turno se pierde si el proceso sale antes de otro pre-step aceptado dentro del turno, así que la UI debe volver a aplicarla.
- Los agents bifurcados heredan el estado de plan registrado, mientras que los agents recién lanzados comienzan inactivos; no hay opción de plan en el momento de la creación.
- Un hijo vivo propiedad de otro agent no puede abrir la revisión de `exit_plan_mode`. La llamada fallida le dice al hijo que incluya la decisión sin resolver en su resultado final; el linaje durable de bifurcación por sí solo no impide que una sesión reanudada como raíz de runtime abra la revisión.
- Solo la UI Web tiene un renderizador especializado de `plan-review`; otro provider de interacción puede presentar la misma solicitud a través de su flujo genérico de opciones.
