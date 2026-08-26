# @deepseek-ai/dsh-tmux-context

[English](README.md) | Español

Contexto duradero opt-in que nombra la sesión, la ventana y el panel de tmux en los que se ejecuta este proceso de agent, además del árbol de paneles de la ventana. Se muestrea una vez por turno durante la preparación de la solicitud al modelo y no forma parte de la composición Web/headless distribuida. Registro de decisión: [la Agent Note de tmux-context](../../../.agents/notes/implemented/feature/2026-07-27-tmux-location-context.md).

## Configuración

```yaml
- id: tmux-context
  name: '@deepseek-ai/dsh-tmux-context'
  config:
    refreshIntervalMs: 60000 # optional; omit or set to 0 to inject on every changed turn
```

`refreshIntervalMs` debe ser un entero seguro no negativo. Si se omite o se pone a `0`, inyecta siempre que el estado de tmux haya cambiado desde la última inyección. Un valor positivo suprime además las inyecciones que caigan dentro de esa cantidad de milisegundos desde la última.

## Cómo lee tmux

El plugin antepone un listener de `agent/pre-step` que solo se ejecuta en el primer paso de cada turno. Cuando corresponde, ejecuta un único comando de solo lectura a través del servicio ejecutor `ctx.shell`:

```sh
[ -n "$TMUX_PANE" ] || exit 1
self_tty=$(ps -o tty= -p <pid> | tr -d ' ')
pane_tty=$(tmux display-message -t "$TMUX_PANE" -p '#{pane_tty}') || exit 1
[ "$pane_tty" = "/dev/$self_tty" ] || exit 1
exec tmux display-message -t "$TMUX_PANE" -p '<format>'
```

`$TMUX_PANE` por sí solo no basta: una terminal lanzada desde un shell de tmux (una terminal integrada de VS Code, un lanzador del escritorio) **hereda** `$TMUX` y `$TMUX_PANE` de ese ancestro, así que las variables están presentes aunque el proceso no viva en ese panel. Por eso el comando compara también el `#{pane_tty}` del panel con la terminal de control del propio proceso (`ps -o tty=` de su pid): un panel real es dueño de la tty de este proceso, mientras que un entorno heredado nombra la tty de algún otro panel. Ejecutarse a través de `ctx.shell` aplica el sandbox y la política del despliegue; el plugin no posee código de subprocesos. Cuando falta `ctx.shell`, el proceso no está en un panel de tmux real (`$TMUX_PANE` sin definir, o la tty no coincide ⇒ salida distinta de cero), o la lectura está malformada, el intento es un no-op, nunca un error. La ubicación es opcional, así que un rechazo del ejecutor — una negativa de política de `resolve()` o un fallo de infraestructura de `run()` — se contiene y se registra como advertencia en lugar de hacer fallar el turno.

El estado se extrae en cada turno elegible — un panel movido, renombrado o redistribuido se detecta sin ningún hook de tmux ni proceso en segundo plano. El plugin solo reinyecta cuando el estado de tmux renderizado difiere de su última inyección, así que una ubicación sin cambios no añade nada.

## Semántica de temporización

El plugin antepone un listener de `agent/pre-step`. Cuando una inyección corresponde y la decisión posterior entra en el paso propuesto, antepone un `UserMessage` con fuente al lote devuelto. AgentLoop registra ese contexto después de `step/start` con la fuente `{ kind: 'plugin', plugin: 'tmux-context' }`. La supresión de cambios y la programación de intervalos escanean los eventos duraderos de sesión sin procesar en busca de la última inyección de esta fuente, de modo que la programación sobrevive a la compactación y a los procesos reanudados sin estado de caché local al proceso; las sesiones se programan de forma independiente. Un listener de pre-step posterior que rechace o falle impide que la lectura se registre.

## Experiencia del modelo

### Ubicación de tmux en el momento de la preparación

#### Qué ve el modelo

En cada turno cuyo estado de tmux haya cambiado, un mensaje de contexto etiquetado con su fuente con las tres líneas siguientes. `<window-layout>` es la descripción compacta del árbol de paneles de tmux; los tamaños en píxeles de paneles y ventanas se excluyen a propósito, y el contenido de los paneles hermanos nunca se captura.

##### Lectura del turno cambiado

```markdown
tmux location (turn <turn>):
session <session>, window <index> "<name>", pane <index> <pane-id>
window active=<0|1>, pane active=<0|1>, layout <window-layout>
```

#### Efecto de tokens

Cada lectura de dos líneas se acumula hasta que la compactación la sombrea. Las ubicaciones sin cambios y la supresión por intervalo no añaden nada.

#### Efecto de KV Cache

De solo anexión; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas de KV Cache existentes.

## Limitaciones conocidas y trabajo diferido

- **Solo el primer paso** — un panel movido o redimensionado a mitad de turno se refleja en el siguiente turno, no entre pasos.
- **Solo la propia ubicación** — el plugin nunca captura el texto visible de los paneles hermanos.
- **Distribución, no tamaño** — las dimensiones en píxeles de paneles y ventanas se omiten; solo se notifican el árbol de distribución y las banderas de activo.
- **Campos separados por tabulaciones** — un nombre de ventana de tmux que contenga la secuencia literal de dos caracteres `\t` dividiría mal la lectura y se omitiría como malformada; los nombres ordinarios no se ven afectados.
- **Detección de panel basada en tty** — el proceso se considera «en tmux» solo cuando su terminal de control coincide con el `#{pane_tty}` de `$TMUX_PANE`. Esto excluye deliberadamente las terminales que heredaron `$TMUX`/`$TMUX_PANE` de un ancestro de tmux (p. ej. una terminal integrada de VS Code). `ps -o tty=` es POSIX; la comprobación es un no-op dondequiera que ella o `#{pane_tty}` no estén disponibles.
