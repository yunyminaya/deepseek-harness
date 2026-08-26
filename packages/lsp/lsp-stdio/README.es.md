# @deepseek-ai/dsh-lsp-stdio

[English](README.md) | Español

Un **backend genérico de servidor de lenguaje por stdio** para `ctx.lsp`. Una instancia de plugin acepta una tabla de servidores con nombre y registra un provider aislado por entrada. Lee a través de `ctx.fs` y lanza mediante `ctx.subprocess`, así que el servidor y el origen siempre habitan el mundo de ejecución montado. Es un host genérico, no un catálogo ni un instalador de servidores de lenguaje — los despliegues configuran comandos y mapeos explícitamente; los presets pertenecen a los overlays de `cordis.yml`.

Plugin de namespace (`name` / `inject` / `Config` / `apply`, sin export por defecto).

## Qué hace

- Resuelve todos los ajustes locales del servidor antes del registro; un mapeo inválido o un conflicto de registro revierte las entradas anteriores, de modo que una carga fallida no deja rutas de provider.
- Lanza de forma perezosa y con single-flight un proceso de servidor por `(id de servidor, destino de workspace canónico)`. Un error de servidor en vivo no se reproduce; si el transporte agrupado seleccionado falla antes o durante una consulta de solo lectura, el provider espera su liberación y reintenta esa consulta una vez en un proceso nuevo.
- Usa una secuencia **transient-open** de compatibilidad primero por consulta: resuelve y acota en bytes el origen mientras lo transmite por `ctx.fs`, `textDocument/didOpen` (versión 1, texto completo), la petición solicitada y luego `textDocument/didClose` en `finally`. Una escritura `didOpen` fallida o cancelada termina la instancia antes de que el pool pueda reutilizarla. Los documentos se cierran después de cada llamada, así que la primera versión no necesita `didChange`, caché de contenido ni LRU de documentos.
- Serializa cada ciclo de vida de lectura/apertura/consulta/cierre de origen a través de una cola anulable por workspace, de modo que las llamadas en cola leen el origen actual solo cuando les llega su turno; los workspaces distintos se ejecutan en paralelo. La liberación del provider anula el trabajo de filesystem y protocolo, espera las búsquedas de workspace que no han entrado en una cola y luego vacía todas las colas y servidores.
- Cuando el apagado del protocolo falla, termina el árbol descendiente del servidor a través del seam de subprocess (señalización de grupo de procesos POSIX; `taskkill /T /F` en Windows). La entrega del tree-kill está contenida como toda señal de grupo — compite con la salida del servidor — y la quiescencia se confirma con la espera de viveza del árbol del handle, no con el resultado del propio kill.
- Resuelve el ejecutable del servidor, el cwd, el proceso y los streams de protocolo a través de `ctx.subprocess`; `initialize.processId` es `null` porque otra máquina o namespace de PID no debe monitorizar el proceso del harness.
- Usa la contención canónica de `ctx.fs`, las URIs de archivo y la validación de texto transmitido, pero no emite `fs/observed`: solo el resultado LSP es visible para el modelo, así que una consulta no satisface la política de leer-antes-de-escribir.

## Configuración

La clave del registro `servers` es el id de provider estable reservado en `ctx.lsp`; cada valor tiene esta forma:

| Clave de servidor | Por defecto | Significado |
|---|---|---|
| `command` | (obligatorio) | Ejecutable a lanzar — absoluto, o resuelto en el PATH del hijo en la carga. El lanzamiento no usa shell. |
| `args` | `[]` | Argumentos pasados al ejecutable. |
| `env` | `{}` | Env adicional fusionado sobre el env ambiental depurado de credenciales (las vars que coinciden con `KEY`/`PASSWORD`/`SECRET`/`TOKEN` no se reenvían); una entrada explícita de `DSH_*` se fusiona después de la depuración que el seam hace de las ambientales. |
| `extensionToLanguage` | (obligatorio) | Extensión en minúsculas con punto inicial → id de lenguaje LSP (p. ej. `{ '.ts': 'typescript' }`). |
| `initializationOptions` | `null` | Opciones estáticas de `initialize` reenviadas al servidor. |
| `configuration` | `null` | Respuesta estática a cada elemento de `workspace/configuration`. |
| `maxMessageBytes` | `16000000` | Mayor mensaje enmarcado individual aceptado del servidor. |
| `maxStderrBytes` | `1000000` | Mayor cola de stderr conservada para diagnósticos. |
| `maxDocumentBytes` | `4000000` | Mayor archivo de origen que abrirá este host. |
| `shutdownTimeoutMs` | `5000` | Presupuesto de `shutdown`/`exit` elegante antes de la escalada. |
| `killGraceMs` | `2000` | Gracia para la cancelación de peticiones y para la escalada de SIGTERM→SIGKILL. |

`servers` debe contener al menos una entrada, y todo id debe ser no vacío. Los presupuestos de temporizador deben ser enteros positivos no mayores que el límite de temporizador de Node de `2_147_483_647` ms. Todos los ejecutables se resuelven en la carga después de la depuración de credenciales; una entrada mala posterior impide que se registre cualquier provider. Los procesos se lanzan de forma perezosa en la primera consulta coincidente.

## Comportamiento del protocolo

La inicialización anuncia `general.positionEncodings: ['utf-16']`, `workspace: { workspaceFolders: true, configuration: true }`, `textDocument.hover.contentFormat: ['markdown', 'plaintext']` y `linkSupport: true` para definición e implementación, sin registro dinámico. Las capacidades devueltas por el servidor son autoritativas: una operación no soportada, o una sincronización sin apertura/cierre transitorio, hace fallar la consulta. Un `positionEncoding` de servidor omitido toma `utf-16` por defecto; cualquier otro valor es un error de protocolo. El cliente responde a `workspace/configuration` desde la configuración estática, acepta las peticiones de contabilidad del ciclo de vida y rechaza `workspace/applyEdit` — nunca aplica ediciones ni ejecuta comandos. La navegación mapea `Location` directamente y `LocationLink` desde `targetUri` + `targetSelectionRange`; la normalización de hover toma el `MarkupContent.value` válido, preserva los `MarkedString` de cadena, renderiza los valores etiquetados con lenguaje como código con vallas y une los arrays con una línea en blanco. Los resultados ausentes, los rangos o posiciones malformados y las codificaciones de hover malformadas fallan como errores estructurados `LSP_MALFORMED_RESPONSE`.

## Límite de seguridad

El provider confía en su servidor configurado y no reclama confinamiento de sandbox. Delega la identidad canónica, la contención, el streaming de archivos regulares, la validación UTF-8 y la codificación de URI de archivo en `ctx.fs`; rechaza los orígenes de consulta ausentes, no regulares, no UTF-8, sobredimensionados o canónicamente fuera del workspace antes del arranque del servidor. La contención se evalúa antes de abrir el stream y no promete identidad de handle estable a través del reemplazo concurrente de rutas. Las ubicaciones de resultado pueden ser externas, pero una ruta externa no puede convertirse en origen de consulta. Un despliegue debe montar providers de filesystem y subprocess para el mismo mundo de ejecución; la composición de mundos divididos es inválida.

## Experiencia de modelo

Indirectamente, a través de `dsh-tool-lsp`, que expone los resultados normalizados de este provider; este host no aporta prompt ni schema por sí mismo.

#### Efecto de KV Cache

Sin invalidación directa; `dsh-tool-lsp` es dueño de los cambios de prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **Sin política de confinamiento** — este paquete confía en el servidor configurado y no mete su proceso en un sandbox; un despliegue restringido debe aportar providers de proceso/filesystem apropiados o un wrapper de sandbox del mismo mundo.
- **Suelo de compatibilidad transient-open** — los servidores cuya sincronización omite open/close (o anuncian `None`) no se soportan aunque las consultas de documento cerrado funcionaran; el e2e de TypeScript fijado establece un suelo de compatibilidad, no una afirmación multilingüe.
- **Latencia de serialización por servidor/workspace** — los agentes en paralelo comparten un servidor y la cola de workspace tras un único proceso; los procesos de workspace de larga vida consumen memoria hasta su liberación.
- **Un harness matado a la fuerza deja servidores de lenguaje huérfanos** — `initialize.processId: null` elimina la monitorización del PID de cliente por parte del servidor, así que los servidores solo se limpian con la liberación elegante del servicio; un harness con SIGKILL los deja corriendo hasta que salen por su cuenta.
