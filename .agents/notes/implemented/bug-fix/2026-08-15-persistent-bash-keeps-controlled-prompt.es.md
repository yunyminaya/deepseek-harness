# Agent Note: El bash persistente conserva el prompt controlado del backend

Status: implemented

[English](2026-08-15-persistent-bash-keeps-controlled-prompt.md) | Español

## Problema

`dsh-tool-bash-persistent` inicializaba su shell con `stty -echo; PS1='__DSH_PERSISTENT_BASH_PROMPT__ '`, sobrescribiendo el `PS1` que `dsh-terminal-bash` establece en el entorno de spawn. La preparación de prompt del backend exige que la cola imprimible tras el marcador OSC `133;D` sea exactamente igual al prompt controlado ([diseño](../feature/2026-07-16-persistent-pty-sessions.es.md)), así que tras la inicialización ningún envío podía asentarse a través de él. `PROMPT_COMMAND` sobrevivía a la sobrescritura, por lo que el marcador seguía llegando y cada envío pagaba el nivel de silencio más la gracia de traspaso: 3.5 s por llamada de herramienta con los valores por defecto de producción, 7.2 s para la primera llamada porque el envío de inicialización también se degradaba, y una cola extra de 3.5 s tras cada comando largo. macOS no tiene un nivel exacto de espera de stdin, y en Linux la sonda exacta no puede observar un comando de sub-intervalo de poll saliendo de su espera de stdin, así que la degradación se aplicaba a prácticamente todas las llamadas. Las pruebas del paquete lo enmascaraban configurando `idleSilenceMs: 100`.

La sobrescritura existía para dar a la herramienta un prompt conocido para dos consumidores: un respaldo por sufijo de viewport que detectaba «shell at a prompt without the end marker», y el recorte cosmético del texto del prompt de la salida parcial.

## Decisión

El backend es dueño de su protocolo de prompt y se lo repara él mismo: el `PROMPT_COMMAND` controlado vuelve a imponer `PS1` después de imprimir el marcador, de modo que cualquier sobrescritura de prompt dentro del shell —la antigua inicialización de esta herramienta, un comando del modelo, un script cargado— dura cero prompts. Esto también protege a los providers que no pueden informar del estado del foreground, donde el texto exacto del prompt es la única evidencia de preparación.

La herramienta deja de sobrescribir `PS1` (la inicialización es solo `stty -echo`) y sustituye su respaldo por sufijo de viewport por la señal existente del seam: un envío que se asienta como `stdin_read` sin el marcador final en el scrollback devuelve la salida parcial capturada. La constante privada del prompt y su recorte se eliminan; la salida parcial puede ahora terminar con el propio texto de prompt del backend, que la herramienta no puede ni debe conocer.

## Alternativas consideradas

**Arreglar solo la herramienta, dejando `PROMPT_COMMAND` sin cambios.** Rechazado porque el seam seguiría siendo silenciosamente frágil: cualquier consumidor o comando del modelo posterior que toque `PS1` reintroduce la degradación de 3.5 s sin ninguna señal de fallo, y los providers sin inspección del foreground pierden su único factor de preparación.

**Importar el prompt controlado en la herramienta.** Rechazado porque el prompt es una constante de protocolo de un provider; un Consumer que lo emparejara acoplaría la herramienta a `dsh-terminal-bash` específicamente, y cualquier otro backend montado la volvería a romper.

**Quitar el factor del texto del prompt de la preparación del backend.** Rechazado porque para los providers cuyo `inspectForeground` no informa de nada, marcador más texto es la defensa contra la salida de comandos que incrusta la secuencia cruda del marcador OSC; debilitarlo cambia una vía rápida por un riesgo de falso asentamiento.

**Ampliar en su lugar el ajuste de `handoffGraceMs`/`idleSilenceMs`.** Rechazado porque ningún valor de silencio arregla una vía rápida muerta; solo reequilibra cuánto paga de más cada llamada.

## Consecuencias

Medido en darwin con los valores por defecto de producción: los envíos crudos se asientan en ~86 ms con el prompt controlado intacto frente a ~3540 ms tras una sobrescritura; las llamadas de herramienta bajan de 7180/3560/3566 ms a 355/88/91 ms para spawn+init+echo, echo y pwd.

El respaldo `stdin_read` es comportamiento, no solo cosmética: tras `exec`, una interrupción o un hijo de foreground interactivo cuya espera de stdin el provider demuestra (el nivel exacto de Linux), la llamada devuelve ahora la salida parcial capturada en lugar de agotar el plazo del comando. Cuando ningún provider demuestra la espera (macOS), un hijo interactivo sigue ejecutándose hasta `timeoutMs` —registrado como limitación conocida en el README de la herramienta. La salida parcial puede portar el prompt final del backend; la salida completa delimitada por marcadores es idéntica byte a byte a la anterior, como confirman las instantáneas jsonrpc-agent sin clave.

La suite de composición del loader establece ahora `idleSilenceMs` por encima del límite de envío, de modo que el silencio no puede asentar nada y todo caso falla si la preparación de prompt retrocede; un caso con PTY real sobrescribe `PS1` dentro del shell y exige que el siguiente envío se asiente como `stdin_read` con el prompt sanado. La auto-reparación no puede sobrevivir a un comando que sobrescriba `PROMPT_COMMAND` en sí; el nivel de silencio sigue siendo el límite ahí, sin cambios respecto al diseño anterior.
