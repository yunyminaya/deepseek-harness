# @deepseek-ai/dsh-permission-presets

[English](README.md) | Español

Presets de permiso visibles para el usuario a través de `ctx.permissionPresets` ([`PermissionPresetService`](src/index.ts)). Cada nombre configurado agrupa `sandbox/mode` con `approval/policy`; los valores por defecto son `workspace-write` (`workspace-write` + `ask`) y `danger-full-access` (`danger-full-access` + `never`). Los adaptadores de UI pueden exponer la tabla como un único selector, mientras que la ejecución de sandbox y la aprobación siguen consumiendo sus propios knobs.

`set(session, name)` registra una selección cambiada en un evento solo de log `permissionPresets/preset` y luego llama al setter de cada knob solo cuando su valor efectivo cambia. El evento de selección precede a los eventos de knob y conserva la intención del usuario cuando los presets comparten un bundle; una selección neta cero no anexa nada. `current(events)` prefiere una selección registrada que siga coincidiendo, luego la primera entrada de la tabla que coincida y, en caso contrario, devuelve `custom`. Los clientes pueden mostrar `custom` como el valor actual, pero no pueden seleccionarlo.

El servicio es dueño del namespace de Ajustes `permissionPresets`. Su `defaultPreset` es el valor por defecto para las sesiones futuras: la entrada de composición usa `Config.defaultPreset` o, cuando se omite, infiere el preset que coincide con los valores por defecto de sandbox y aprobación compuestos. Un cambio de Ajustes confirmado se lee cuando se crea la siguiente sesión; la creación fija `permissionPresets/preset`, `sandbox/mode` y `approval/policy` en esa sesión, de modo que los cambios posteriores nunca alteran una sesión existente. Una semilla reanudada, incluida una explícitamente vacía marcada por `session/end-seed`, conserva su permiso efectivo y recibe solo los hechos duraderos que faltan, no el último valor por defecto del usuario. Montar el servicio también barre las sesiones ya vivas, de modo que un reemplazo por HMR fija cualquier sesión creada mientras el plugin estaba ausente.

El servicio exige un executor de `ctx.shell` confinante y `ctx.approval`. Una entrada de tabla llamada `custom` lanza una excepción al cargar. Cuando los valores por defecto de la composición no coinciden con ningún preset, el plugin exige un `defaultPreset` explícito; una sesión sin eventos construida de forma independiente puede seguir derivando `custom`. Ver el [diseño de conmutación de sandbox](../../../.agents/notes/implemented/feature/2026-07-06-sandbox.es.md).

Dos hijos opcionales incluyen las superficies de producto sobre el mismo servicio: una unidad de proyección de sesión `permissions` (`src/types.ts` declara la clave; la unidad pliega los tres eventos de knob de valor completo y ve el select — opciones de la tabla más un `custom` solo actual — sobre los valores por defecto de la composición) y el comando `/permissionPresets` (la invocación desnuda informa del preset actual y de la tabla; un argumento de preset conmuta a través de `set`). Cada hijo se activa solo cuando su registro (`ctx.sessionProjections` / `ctx.commands`) está compuesto.

## Experiencia del modelo

Indirectamente, a través de `dsh-user-approval` y `dsh-tool-bash`, que renderizan el prompt de política de aprobación, el aviso de conmutación y los resultados de herramientas en sandbox seleccionados por los eventos de knob de este servicio; `permissionPresets/preset` en sí es solo de log.

#### Efecto en la caché KV

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio en el prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **Solo se incluyen dos knobs de mecanismo** — los presets seleccionan el modo de sandbox y la política de aprobación; una elección de agent/perfil aún no forma parte de `PresetSpec`.
- **`custom` es solo derivado** — los llamadores pueden alejarse de una combinación de knobs sin coincidencia, pero no pueden apuntar ni persistir un preset custom con nombre a través de este servicio.
- **La tabla de presets es a nivel de proceso** — la configuración es fija durante toda la vida del plugin; cambiar los presets disponibles exige recargar el plugin.
- **Los valores por defecto almacenados deben permanecer en la tabla de presets** — eliminar el preset referenciado hace fallar el registro de los ajustes de Permission hasta que la sección `permissionPresets` de `settings.yaml` se actualice o se restablezca.
