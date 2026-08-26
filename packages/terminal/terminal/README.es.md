# @deepseek-ai/dsh-terminal

[English](README.md) | Español

Seam de PTY persistente con ámbito de dueño. `TerminalSessionService` se registra como `ctx.terminals`, acuña ids de sesión opacos, enruta la creación a través de backends con nombre, valla cada operación al `Agent` vivo exacto y espera la quiescencia del backend cuando ese agent o el servicio se disponen.

## Contrato

- Los backends registran un `type` estable y devuelven una `TerminalBackendSession` no publicada; un setup fallido o cancelado debe limpiar los recursos parciales, y una limpieza fallida rechaza con `TerminalBackendCleanupError` para que el registro pueda conservarla a través de la cancelación.
- La cancelación del spawn conserva la razón de aborto exacta de quien llama. La disposición del servicio y la pérdida del dueño siguen siendo fallos distintos encaminables por la máquina tras el setup del backend.
- La disposición del dueño y del servicio aborta el setup no publicado mediante una señal propiedad del servicio y espera el asentamiento del backend más el rollback antes de retornar.
- Un cierre de rollback o un fallo de limpieza de arranque notificado por el backend rechaza el ciclo de vida en disposición en lugar de declarar quiescencia. La cancelación disparada por quien llama sigue recibiendo su razón exacta; el fallo de rollback disparado por el ciclo de vida también rechaza el spawn pendiente.
- Un fallo de limpieza del backend posterior a la cancelación de quien llama sigue siendo actividad del dueño hasta que la disposición del dueño o del servicio lo consume y lo notifica, de modo que la política de ciclo de vida no pueda confundir una limpieza fallida con quiescencia.
- `hasOwnerActivity(owner)` abarca desde el setup no publicado hasta el cierre final, de modo que la política de ciclo de vida pueda vallar al dueño exacto sin una carrera de publicación.
- Un spawn exitoso publica una `TerminalSessionId`. El `name` opcional es metadatos de visualización locales al dueño, nunca autoridad.
- Una sesión acepta como máximo una operación de envío viva. Las lecturas y señales pueden observarla; otro envío falla hasta que la operación se asienta.
- `TerminalSendResult.waitReason` y `sessionStatus` son independientes. `session_exit` describe el proceso PTY de nivel superior, no un comando arbitrario en primer plano.
- `kill()` y la disposición se resuelven solo tras la quiescencia del árbol de procesos capturado por el backend. Un fallo de limpieza rechaza en lugar de declarar éxito y limpia las barreras del backend y del registro correspondientes para que un cierre posterior pueda reintentar sin perturbar un intento más nuevo.

El seam no contiene ninguna política de `node-pty`, sandbox, schema de herramienta, prompt, tarea o renderizado de terminal. Las implementaciones son dueñas de la mecánica del terminal; los consumidores son dueños de la presentación al modelo y del registro opcional de trabajos en segundo plano.

## Model Experience

### Consumidor indirecto

#### Lo que ve el modelo

Nada directamente. Este paquete no registra prompt ni herramienta; `@deepseek-ai/dsh-tool-terminal` es dueño de los schemas visibles y del texto de resultado.

#### Efecto de tokens

Ninguno directamente. El estado vivo de la sesión permanece local al proceso hasta que un consumidor devuelve un resultado acotado.

#### Efecto de caché KV

Sin invalidación directa; el consumidor con nombre es dueño de los cambios del prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- Las sesiones son locales al proceso y no se restauran tras un reinicio del harness.
- La compartición entre agents está ausente deliberadamente; un diseño futuro de sesión compartida necesita un contrato de autoridad separado.
