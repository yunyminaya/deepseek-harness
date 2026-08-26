# @deepseek-ai/dsh-commands

[English](README.md) | Español

Registro de comandos humanos propiedad del plugin que consumen los adaptadores de UI interactivos. La [Agent Note de registro de comandos de plugins](../../../.agents/notes/implemented/feature/2026-07-19-plugin-command-registration.es.md) es dueña del límite y del contrato de despacho.

## Contrato del servicio

`ctx.commands.register(definition)` registra un nombre de comando en minúsculas, una descripción, un descriptor opcional de entrada no estructurada (`hint` más una bandera `images` que declara si los adjuntos de imagen del compositor pueden acompañar a una invocación), una política opcional de `recordInput` y un handler abortable. `recordInput` toma por defecto `true`; un comando cuyo evento de dominio autoritativo es dueño del payload lo pone a `false` para que `command/run` omita `args` en lugar de duplicar la entrada. Un comando registrado está disponible para todos los adaptadores de comandos compuestos; un plugin incompatible con un despliegue no se registra ahí. Un registro en contexto plano es global. Un plugin productor de comandos montado bajo `agent.ctx` declara su propia inyección de `commands` y crea una definición con ámbito exacto de agent; sombrea una definición global homónima. Esta forma de inyección de hijo conserva el ámbito del agent sin hacer que el agent loop del núcleo dependa de un servicio de UI. Los nombres duplicados dentro de una misma capa fallan durante el registro. Cada disposer es el disposer de effect exacto de Cordis, y el registro o la eliminación notifican a todos los observadores de `commands/change` para que los adaptadores vivos puedan refrescar el descubrimiento; los fallos de los observadores se registran y no pueden vetar la mutación del registro ni privar de recursos a observadores posteriores.

`list(agent)` devuelve descriptores inmutables ordenados por nombre tras el sombreado con ámbito (el descriptor lleva `input.images` para que los compositores puedan rechazar los envíos de imágenes a comandos no declarantes antes del despacho). `find(agent, name)` devuelve la definición correspondiente. `execute(agent, line, images, signal)` usa `parseCommand()` y ejecuta solo un comando conocido, devolviendo el `CommandExecution` asentado (el resultado normalizado más el emparejamiento de ciclo de vida `commandId`) o `undefined` para sintaxis inválida o nombres desconocidos. `images` lleva las imágenes del compositor codificadas en base64 del envío (`EncodedImageAttachment` de `@deepseek-ai/dsh-attachment/types`); el executor hace cumplir la declaración — las imágenes enviadas a un comando no declarante, un store de `attachments` ausente o un límite de lote superado se asientan cada uno como un resultado de error antes de que corra el handler, y un lote rechazado no publica ningún objeto duradero. Un lote admitido se confirma a través de `admitEncodedImages` y se entrega al handler como `ImageBlock`s congelados y ordenados en `invocation.attachments`; el handler es dueño de su uso visible para el modelo y devuelve un error cuando su gramática no puede usarlos, de modo que el compositor que despacha conserva los originales. El ciclo de vida de un comando resuelto se registra en la sesión del agent receptor como el par solo de log `command/run` (antes del handler, con un `commandId` acuñado, el nombre estructurado del parser, el `CommandSource` emisor y `args` salvo que `recordInput` sea `false`) y `command/done` (en el asentamiento, con el kind del resultado y el texto verbatim; un resultado correcto también puede nombrar un evento de dominio autoritativo anterior no relacionado con comandos a través de `sourceEventSeq`; un handler que lanza o se aborta se asienta como `kind: 'error'`). Los fallos de admisión no registran nada. Ambos son anexiones directas e independientes en la sesión del agent receptor: ningún turno los envuelve, y la persistencia los drena a través de los checkpoints y el teardown ordinarios.

`parseCommand()` reconoce una barra en el byte cero, un nombre en minúsculas que contenga letras, dígitos, `_` o `-`, y ya sea fin de entrada o un espacio en blanco. Devuelve todos los bytes posteriores al nombre como `rawInput`, incluido el espacio separador; los consumers son dueños de su gramática específica de comando y solo pueden normalizar lo que esa gramática permita.

Los handlers devuelven `success` o `error` más texto de UI opcional. Un handler correcto también puede devolver `sourceEventSeq` cuando un evento de dominio anterior es dueño de una presentación más rica; el invariante de ciclo de vida exige que esa referencia sea un evento anterior no relacionado con comandos en la misma sesión. Los resultados los renderiza directamente el adaptador y nunca entran en el historial del modelo. El registro nunca envía `rawInput` al agent implícitamente; un productor de comandos puede programar explícitamente trabajo visible para el modelo a través del `Agent` receptor, en cuyo caso ese productor es dueño del contrato de mensaje resultante. El registro compite la finalización del handler contra la señal de abort suministrada, pero un handler poco cooperativo puede continuar sus propios efectos secundarios externos después de que el llamador deje de esperarlo.

## Composición

La base incluida de `dsh` monta este servicio y el cliente web despacha a través de él. Los spines de demo sin UI y la automatización ACP no aportan un adaptador de comandos. Las composiciones interactivas personalizadas y los productores de comandos montan `@deepseek-ai/dsh-commands` explícitamente.

## Experiencia del modelo

### Comandos humanos directos

#### Qué ve el modelo

El registro en sí no envía nada. Los comandos de barra conocidos se ejecutan en el plano de comandos de la UI y su texto de `CommandResult` no se envía como mensaje de usuario. La entrada de comando de barra desconocido la rechazan los adaptadores incluidos en lugar de convertirse en un prompt de modelo. Un productor de comandos puede usar explícitamente el `Agent` receptor; por ejemplo, [`dsh-plan-mode`](../../plan/plan-mode/README.es.md#model-and-human-interactions) envía el mensaje opcional de `/plan [message]` después de seleccionar el modo plan. Los adjuntos de imagen siguen la misma regla: el executor solo los admite en objetos de adjunto duraderos, y un productor declarante decide si se convierten en contenido de mensaje visible para el modelo y cómo.

#### Efecto de tokens

El descubrimiento, la ejecución y la salida de UI de comandos no añaden tokens de modelo. El trabajo de agent explícito programado por un productor de comandos tiene el mismo efecto de tokens que la entrada de agent correspondiente.

#### Efecto en la caché KV

Los metadatos del registro, la entrada de comandos y la salida directa nunca entran en una solicitud de modelo y no afectan a su caché. Un dominio mutado es dueño de cualquier efecto de caché posterior.

## Limitaciones conocidas y trabajo diferido

- **Solo entrada de texto no estructurada** — los formularios, los schemas de autocompletado y los argumentos tipados siguen siendo asuntos de parseo propiedad del comando.
- **Cancelación cooperativa de efectos secundarios** — el despacho deja de esperar al abortar; los handlers deben respetar la señal para detener el trabajo que ya escapó a sistemas externos.
