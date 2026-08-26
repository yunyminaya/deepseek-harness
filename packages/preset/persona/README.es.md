# dsh-persona

[English](README.md) | Español

La persona del agent como una row componible. Puede hacer shadow a la persona de despliegue o ser dueña del prompt de sistema completo.

[`dsh-system-prompt`](../../core/system-prompt/README.es.md) es dueño de la persona de despliegue como config propia y registra esa sección incondicionalmente, así que un proceso tiene exactamente una. Un [agent preset](../agent-presets/README.es.md) no puede montar el registro de prompts por sí mismo — sin una row propia, un preset podría cambiar las herramientas de un agent pero nunca su identidad. Este paquete es esa row.

## Solo con scope

Montar esta row fuera de un scope de agent colisiona con el registro `deployment:persona` del propio registro y falla de forma ruidosa. No es una limitación que haya que rodear: la persona de despliegue ya tiene dueño, y el sentido mismo de esta row es hacerle shadow para un solo agent. Móntala dentro de una composición de preset, donde el montaje del preset aporta el scope del agent.

## Config

| Campo | Valor por defecto | Significado |
|---|---|---|
| `text` | requerido | Prosa de la persona renderizada como la sección `deployment:persona` |
| `complete` | `false` | Restaura esta persona tras el ensamblaje como la única sección del prompt de sistema |
| `includeRuntimeContext` | `true` | Incluye las instantáneas dinámicas de runtime-context para este scope de agent; con `false` se suprime toda contribución de contexto sin deshabilitar los servicios que la poseen |

`text` es una plantilla, como cualquier sección de prompt: los grupos `{{…}}` completos se resuelven estrictamente contra las variables de prompt registradas cuando el prompt se renderiza, no cuando se ensambla. El texto vacío sigue ocupando el slot, así que hace shadow a la persona de despliegue por completo y luego desaparece en el render. Con `complete: true`, el ensamblaje sigue resolviendo contextos, herramientas, variables y listeners cooperativos, y después el registro de prompts restaura esta persona exacta como única sección; ninguna identidad, guía de herramientas ni listener puede añadir texto al prompt. Con `includeRuntimeContext: false`, los context providers no se evalúan para este scope y se descartan los contextos añadidos por los listeners del ensamblaje.

## Model Experience

### La sección de la persona

#### Qué ve el modelo

La sección `deployment:persona` en el orden 0, inmediatamente después del opener de identidad del harness, con exactamente el `text` configurado de esta row y las variables de prompt resueltas. Para un agent cuyo preset monta esta row, reemplaza cualquier persona que haya configurado el despliegue. En modo completo, el modelo ve solo esta sección renderizada como su prompt de sistema. El runtime context sigue habilitado por defecto. Al deshabilitarlo, un agent recién creado no recibe ninguna instantánea de runtime-context de la sandbox policy, la approval policy, la delegación ni de ningún otro context provider del prompt de sistema.

#### Efecto de tokens

Fijo para un preset dado: los tokens propios de la persona en cada petición que haga ese agent, y ninguno para cualquier otro agent. El texto vacío no contribuye nada. El modo completo elimina todos los demás tokens del prompt de sistema para ese agent.

#### Efecto de KV Cache

Estable en el prefijo durante toda la vida de un agent — la row se monta una vez, antes de que el agent se publique y por tanto antes de su primera petición, y su texto nunca cambia mientras el agent se ejecuta. Dos agents en presets distintos establecen prefijos diferentes a partir de esta sección; ninguno puede invalidar la reutilización del otro.

## Limitaciones conocidas y trabajo aplazado

- **Sin montaje global** — el registro de prompts es dueño del slot de persona sin scope, así que esta row solo se puede usar desde una composición con scope. Un cambio de persona para todo el despliegue pertenece a la config de la propia row `system-prompt`.
