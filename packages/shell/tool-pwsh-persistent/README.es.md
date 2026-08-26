# @deepseek-ai/dsh-tool-pwsh-persistent

[English](README.md) | Español

`pwsh(command)` orientada al modelo respaldada por un único shell `ctx.terminals` con alcance de propietario. El paquete es dueño del contrato de herramienta y de la reutilización del shell; los despliegues seleccionan el backend de terminal (una instancia de `terminal-bash` configurada con `shellDialect: pwsh`) y la política de sandbox. Es la contraparte Windows de `tool-bash-persistent`: mismo contrato de estado persistente, dialecto PowerShell.

## Config

| Clave | Valor por defecto | Significado |
|---|---:|---|
| `backendType` | `shell` | Backend de terminal registrado usado para cada shell de Agent. |
| `timeoutMs` | `300000` | Límite de tiempo de pared para un comando; el timeout cierra el shell. |
| `maxOutputChars` | `16000` | Máximo de caracteres de salida de comando retenidos; los diagnósticos fijos se añaden después. |
| `description` | Persistent-shell description | Contrato de entorno orientado al modelo. |

## Experiencia del modelo

### Schema de herramienta

#### Lo que ve el modelo

El [schema `pwsh`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-pwsh-persistent) generado, incluida la `description` configurada. El plugin no contribuye ninguna sección independiente de system prompt; el despliegue es dueño de la persona y la guía de entorno.

#### Efecto en tokens

Costo fijo de schema mientras `pwsh` es visible.

#### Efecto en la caché KV

Estable por prefijo mientras la description configurada y el schema permanecen sin cambios.

### Resultados de herramienta

#### Lo que ve el modelo

Los comandos comparten un shell por Agent, así que cwd, las variables `$env:`, las funciones y los jobs en segundo plano persisten entre llamadas. Los resultados excluyen los marcadores privados de finalización, el prompt del shell y la línea de entrada reflejada (PSReadLine devuelve la entrada enviada al flujo; la extracción anclada al marcador y la tira de la fuente del wrapper la eliminan). Un comando envuelto distinto de cero añade `[exit code: N]` — el código de salida nativo exacto cuando el comando ejecutó un programa nativo, `1` para un error de PowerShell que termina. Un shell que sale antes de informar de ese estado añade en su lugar `[shell exited: code N]`, `[shell killed by signal: SIG]` o `[shell exited]` cuando el backend no aporta ninguno (la terminación forzada en Windows informa salida 1 sin señal), y luego se reinicia y le dice al modelo que la siguiente llamada empieza de cero. La salida larga conserva el prefijo retenido más antiguo más un aviso de recorte; si la terminal ya ha descartado ese prefijo, el resultado lo dice explícitamente. El timeout devuelve salida parcial acotada, cierra el shell incierto e informa del reinicio.

#### Efecto en tokens

Dependiente de los datos. `maxOutputChars` acota la salida de comando retenida; los diagnósticos fijos de recorte, prefijo perdido, estado, timeout y reinicio pueden extender el resultado.

#### Efecto en la caché KV

Los resultados de herramienta de solo añadido siguen el prefijo de petición reutilizable.

## Limitaciones conocidas y trabajo diferido

- La herramienta requiere un Agent propietario y un backend de terminal real con dialecto pwsh (ConPTY de Windows o un pwsh POSIX).
- **El eco de entrada es inevitable**: PSReadLine de PowerShell devuelve la entrada enviada al flujo de terminal y no existe equivalente de `stty -echo`. La extracción anclada al marcador excluye el eco en los resultados completos; la tira de la fuente del wrapper cubre las rutas de respaldo, pero un wrapper que envuelve al ancho de la terminal puede dejar un eco parcial en los resultados de salida parcial, acotado por `maxOutputChars`.
- Los caracteres ESC crudos dentro de los comandos del modelo no se soportan: PSReadLine los consume antes de la ejecución. El wrapper escapa los bytes de control que necesita (marcadores OSC construidos con `[char]27`, escapes de backtick para el cuerpo).
- Una redefinición por parte del modelo de la función `prompt` elimina el marcador de preparación; el shell se asienta entonces en el nivel de silencio en lugar de la ruta rápida del marcador.
- No hay stdin interactivo durante un comando: un comando en primer plano que lee entrada se bloquea hasta el timeout de preparación, que reinicia el shell.
- SIGTSTP/SIGHUP no están disponibles en Windows (rechazados por el backend); SIGINT se entrega como escritura de Ctrl-C a toda la consola, que en un prompt cancela la línea pendiente en lugar de señalar a un proceso.
- Bajo el modo read-only del sandbox ACL de Windows, pwsh arranca en ConstrainedLanguage, que puede denegar la fijación de codificación `[Console]::` y el marcador de prompt del bootstrap. Los comandos pueden seguir asentándose a través del prompt imprimible y el nivel de silencio, pero la salida no ASCII puede seguir la página de códigos del host.
- El marcador OSC terminado en BEL sigue siendo solo una señal de preparación; un canal de eventos BEL hacia el modelo sigue diferido, en línea con la implementación actual.
