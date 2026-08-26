# @deepseek-ai/dsh-tool-bash-persistent

[English](README.md) | Español

`bash(command)` orientada al modelo respaldada por un único shell `ctx.terminals` con alcance de propietario. El paquete es dueño del contrato de herramienta y de la reutilización del shell; los despliegues seleccionan el backend PTY y la política de sandbox.

## Config

| Clave | Valor por defecto | Significado |
|---|---:|---|
| `backendType` | `shell` | Backend PTY registrado usado para cada shell de Agent. |
| `timeoutMs` | `300000` | Límite de tiempo de pared para un comando; el timeout cierra el shell. |
| `maxOutputChars` | `16000` | Máximo de caracteres de salida de comando retenidos; los diagnósticos fijos se añaden después. |
| `description` | Persistent-shell description | Contrato de entorno orientado al modelo. |

## Experiencia del modelo

### Schema de herramienta

#### Lo que ve el modelo

El [schema `bash`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-bash-persistent) generado, incluida la `description` configurada. El plugin no contribuye ninguna sección independiente de system prompt; el despliegue es dueño de la persona y la guía de entorno.

#### Efecto en tokens

Costo fijo de schema mientras `bash` es visible.

#### Efecto en la caché KV

Estable por prefijo mientras la description configurada y el schema permanecen sin cambios.

### Resultados de herramienta

#### Lo que ve el modelo

Los comandos comparten un shell por Agent, así que cwd, las variables exportadas, los entornos activados, las funciones y los jobs en segundo plano persisten entre llamadas. Los resultados excluyen los marcadores privados de finalización. Cuando el shell vuelve a leer stdin sin haber impreso el marcador de finalización — tras `exec`, una interrupción o un hijo interactivo en primer plano cuya espera de stdin el provider demuestra — la llamada devuelve la salida parcial capturada, que puede terminar con el propio texto de prompt del backend. Un comando envuelto distinto de cero añade `[exit code: N]`; un shell que sale antes de informar de ese estado añade en su lugar `[shell exited: code N]`, `[shell killed by signal: SIG]` o `[shell exited]` cuando el backend no aporta ninguno, y luego se reinicia y le dice al modelo que la siguiente llamada empieza de cero. La salida larga conserva el prefijo retenido más antiguo más un aviso de recorte. Si el PTY ya ha descartado ese prefijo, el resultado lo dice explícitamente en lugar de presentar una cola como salida completa. El timeout devuelve salida parcial acotada, cierra el shell incierto e informa del reinicio.

#### Efecto en tokens

Dependiente de los datos. `maxOutputChars` acota la salida de comando retenida; los diagnósticos fijos de recorte, prefijo perdido, estado, timeout y reinicio pueden extender el resultado.

#### Efecto en la caché KV

Los resultados de herramienta de solo añadido siguen el prefijo de petición reutilizable.

## Limitaciones conocidas y trabajo diferido

- La herramienta requiere un Agent propietario y un backend PTY real.
- Un hijo interactivo en primer plano (por ejemplo un REPL) vuelve pronto con salida parcial solo donde el provider de subprocesos demuestra su espera de stdin; en el resto, la llamada se ejecuta hasta `timeoutMs`.
- El `exit` explícito y el timeout descartan el estado del shell. La cancelación también reinicia y descarta el resultado, incluso cuando ya se observa un marcador de estado completo; la siguiente llamada inicia un shell nuevo.
- Los hechos de entorno como el acceso a red y los espejos de paquetes pertenecen a la `description` configurada, no al valor por defecto de este paquete.
