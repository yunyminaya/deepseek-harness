# @deepseek-ai/dsh-tool-workflow

[English](README.md) | [中文](README.zh.md) | Español

La herramienta **`workflow`** orientada al modelo: ejecuta un script de orquestación en JavaScript que hace fan-out de subagentes y devuelve el valor final del script. Este paquete es dueño del schema orientado al modelo y del ciclo de vida de la ejecución sobre [`ctx.workflowEngine`](../workflow/README.es.md); el análisis, la ejecución, los topes y la cancelación del script viven tras el seam, mientras que el consumer conserva la propiedad del schema y del sobre de resultados orientados al padre.

## Lo que ve el modelo

Tres parámetros: `meta` (datos de identidad obligatorios: `name`, `description` y anotaciones de progreso opcionales), `script` (cuerpo JavaScript simple obligatorio — sin declaración `export const meta`; la descripción de la herramienta lleva el contrato de autoría completo) y `args` (objeto JSON opcional expuesto al script como el global `args`; envuelve una lista desnuda en un campo para que el schema del cable siga siendo honesto). El plugin también aporta una sección de prompt de sistema `tool:<toolName>` con la política de uso — usa la herramienta solo ante una petición explícita del usuario de un flujo de trabajo / orquestación grande; prefiere llamadas de subagente simples para una o dos delegaciones — conforme a la convención de que la guía de la herramienta viaja con el plugin de la herramienta, nunca en la persona del despliegue.

## Ciclo de vida

La recolección es síncrona (como [`dsh-tool-subagent`](../../subagent/tool-subagent/README.es.md)): `execute` inicia una ejecución y espera `run.result` dentro de un `try/finally` que siempre dispone la ejecución, así que el script y sus hijos alcanzan la quietud en todos los caminos. `exec.signal` se tiende un puente hacia `run.cancel()` (incluido el caso ya-abortado-antes-de-arrancar). Una razón de parada distinta de `completed` se mapea a un resultado `isError` que reporta la razón — nunca salida parcial como éxito; un fallo de análisis/meta lanzado síncronamente por `start()` se convierte en un `isError` que el modelo puede corregir. La finalización devuelve el canónico `{ runId, agentsStarted, result }`; el renderizador Native preserva el nombre del meta, el recuento de agents y el valor JSON, truncando solo esa proyección en `maxResultChars`.

Para una ejecución de transporte de raíz (`exec.parent` ausente), la herramienta también proyecta la ejecución en la Session del Agent llamador: run-start después de que `start()` vuelva, inicios y finales de miembros coincidentes filtrados por `run.id`, y luego run-end solo después de que `run.result` esté disponible y `dispose()` haya alcanzado la quietud. Las llamadas anidadas de transporte se ejecutan con normalidad pero no escriben ningún registro de flujo de trabajo. El primer append fallido a la Session deshabilita la grabación posterior de esa ejecución, emite una advertencia y deja o bien ningún registro o un prefijo continuo legal sin cambiar el resultado o la limpieza de la herramienta.

El subpath `@deepseek-ai/dsh-tool-workflow/types` apto para navegador es dueño de estas cuatro cargas de eventos de solo-log y de su declaración `SessionEventMap`. El invariante del paquete rechaza inicios duplicados, miembros sin pareja, eventos terminales con miembros abiertos y actualizaciones tras el run-end tanto en carga fría como en append en vivo, a la vez que acepta sufijos terminales ausentes.

## Intención de renderizado

Decidida de antemano (según la [Agent Note de render-intent](../../../.agents/notes/implemented/architecture/2026-07-02-tool-render-intent-union.es.md)): una tarjeta `generic` titulada `workflow: <meta.name>`, leída directamente de `args.meta.name` (la presentación es una función pura de los argumentos y no pide al motor que analice); el texto del script viaja como `rawInput`. El resultado conserva la tarjeta genérica.

## Configuración

| Clave | Predeterminado | Significado |
|---|---|---|
| `toolName` | `workflow` | El nombre de herramienta orientado al modelo a registrar. |
| `maxResultChars` | `50000` | Tope del resultado renderizado; el JSON más largo se trunca con aviso. |

## Experiencia del modelo

### Prompt del sistema

#### Lo que ve el modelo

Toda petición padre dentro del ámbito de registro de este plugin recibe la guía de flujo de trabajo siguiente. Una restricción con ámbito de herramienta puede ocultar el schema sin eliminar esta guía registrada de forma independiente.

##### Guía de flujo de trabajo

```markdown
Use the <toolName> tool ONLY when the user explicitly asks for a workflow or for large multi-agent orchestration: you write a JavaScript script (the tool description documents the exact format) that fans work out across many subagents with phases and structured results. For one or two delegations, prefer plain subagent calls.
```

#### Efecto de tokens

Pequeño coste fijo de guía por petición mientras el plugin está activo.

#### Efecto de KV Cache

Prefijo-estable mientras el ámbito del plugin y el texto de la guía no cambien. La activación o la disposición puede invalidar la reutilización de esta sección del prompt.

### Schema de la herramienta

#### Lo que ve el modelo

Cuando es visible, el schema [`workflow`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-workflow) predeterminado generado lleva el contrato completo de hooks y metadatos de JavaScript; `toolName` puede renombrar la definición, y el modelo envía script, metadatos y args opcionales.

#### Efecto de tokens

Coste fijo sustancial de schema en cada petición donde la herramienta es visible.

#### Efecto de KV Cache

Prefijo-estable mientras `toolName`, la definición y la visibilidad no cambien. El renombrado, el ciclo de vida del plugin o las restricciones con ámbito pueden invalidar la reutilización de este schema.

### Historial de llamadas y resultado

#### Lo que ve el modelo

El script, los metadatos y los args completos escritos por el modelo permanecen en la llamada de herramienta del asistente. El éxito es exactamente `workflow "<name>" completed (<count> agent<optional-s>).`, salto de línea, `Return value:`, salto de línea y JSON pretty-printed dependiente de los datos; un tope añade `… [truncated: <omitted> more characters]` en una línea nueva. Los fallos son exactamente `Error: workflow run was cancelled`, con sufijo opcional ` (<error>)`, `Error: workflow run failed: <error-or-unknown error>`, o defensivamente `Error: workflow run ended abnormally (<reason>)`; una llamada sin agent propietario se convierte en `Error: workflow tool requires a calling agent (exec.agent was undefined)`. Los mensajes intermedios de los hijos se omiten.

#### Efecto de tokens

Los tokens de la llamada pueden ser grandes y permanecen hasta la compactación. El renderizado del resultado está acotado por `maxResultChars`; los tokens de los modelos hijos son independientes del contexto retenido del padre.

#### Efecto de KV Cache

Solo append; el contenido recién visible sigue al prefijo reutilizable de la petición y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **El turno del padre se bloquea hasta que todo el flujo de trabajo se asienta** — no hay API de arranque/sondeo en segundo plano, y la cancelación descarta la salida parcial como error.
- **`args` debe ser un objeto y el texto del resultado Native está acotado** — los llamadores envuelven arrays/escalares de nivel superior en un campo; el resultado canónico del flujo de trabajo permanece completo, mientras que el JSON por encima de `maxResultChars` se trunca en la proyección orientada al modelo en lugar de almacenarse tras un handle de recuperación.
- **La política de flujo de trabajo es fija por registro de herramienta** — la selección de provider, los topes y el nombre de la herramienta son configuración de despliegue, no argumentos de llamada del modelo.
- **Los registros durables son de nivel superior y observacionales** — los despachos anidados de Code Mode no se graban, y un fallo de grabación se degrada intencionadamente a un prefijo incompleto en lugar de cambiar la ejecución.
