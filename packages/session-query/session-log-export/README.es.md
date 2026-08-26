# @deepseek-ai/dsh-session-log-export

[English](README.md) | Español

Control Web de descarga de session logs sobre el endpoint ZIP transmitido por el host del que es dueño `dsh-host-apiproxy`. La mitad del Host registra `/export`; la mitad del navegador es dueña de una acción `Session log` de 111×32 en el Session Header, de un controlador de descarga y de un modal compartido por ese botón y el comando de barra. La generación del ZIP, las lecturas crudas JSONL/zstd, los descendientes, los attachments, la contrapresión y la semántica de errores HTTP siguen siendo propiedad de la [implementación de descarga de ApiProxy](../../host/apiproxy/README.md).

## Contrato del comando

| Entrada | Resultado |
|---|---|
| `/export` | Registra un ciclo de vida de comando humano; el navegador que lo envía recibe el acuse local de ejecución y descarga `GET /api/session.export?sessionId=<id>&includeDescendants=true`. |
| `/export <path>` | Devuelve un error. Las descargas del navegador eligen su destino mediante el comportamiento ordinario de descarga del navegador. |

El comando lo monta solo el bundle Web. El acuse local `command/executed` dispara la descarga de barra solo después de un resultado `/export` con éxito en el navegador que lo envió; otras pestañas siguen renderizando la fila durable del comando sin repetir el efecto secundario del navegador. El botón del Header llama directamente al mismo controlador. Ambas rutas de entrada emiten un preflight `HEAD` y luego entregan la URL GET al gestor de descargas del navegador sin almacenar el ZIP en JavaScript; comparten el colapso de las descargas en curso, la cancelación del preflight en la disposición del plugin, el manejo de errores de preparación, el comportamiento de guardado del navegador y el mismo Modal.

El endpoint de descarga del Host hace flush de una root Session en vivo antes de `readRaw`, así que un ZIP disparado por barra incluye el par `command/run` y `command/done` cuyo acuse inició la descarga. Las Sessions frías persistidas no requieren flush.

El modal informa de la preparación, del inicio de la descarga o del fallo. Cerrarlo no cancela una descarga en curso ni la vuelve a abrir cuando esa operación se resuelve después. Una Session admite una descarga activa a la vez; los gestos repetidos comparten esa operación.

## Composición

```yaml
- id: session-log-download
  name: '@deepseek-ai/dsh-session-log-export'
```

El bundle Web monta el paquete junto a `dsh-host-apiproxy`, `dsh-commands`, `dsh-client-ui-commands` y `dsh-client-ui-conversation`. El paquete aporta su botón y su modal a la lista alineada a la derecha `conversation.session.header.utilities`, independientemente de las entradas de modo, Subagent y Task adyacentes al título en `conversation.session.header.actions`; Trajectory no lleva ningún control de exportación.

## Experiencia del modelo

### Control humano de `/export`

#### Lo que ve el modelo

Nada. `/export` permanece en el plano de comandos humanos, y la descarga del ZIP no entra en el historial del modelo.

#### Efecto en tokens

Cero. El comando no crea ningún turno de modelo.

#### Efecto en la caché KV

Ninguno. El ciclo de vida del comando solo de log y la descarga del navegador no cambian el prefijo de petición derivado.

## Limitaciones conocidas y trabajo diferido

- El endpoint de descarga exige un backend de persistencia con un artefacto crudo por Session. El backend JSONL incluido soporta artefactos en texto plano y zstd; la exportación SQLite no está incluida en este cambio.
- Esto es una descarga de navegador, no un escritor de ruta del Host. El navegador elige el destino local; no se devuelve ninguna ruta del Host ni acción de carpeta nativa.
- El preflight informa de los fallos encontrados antes de que empiece la transmisión del ZIP. Un fallo de descendiente o attachment después de que el navegador acepte el GET lo informa el gestor de descargas del navegador, no el modal.
