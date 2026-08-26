# @deepseek-ai/dsh-session-title-llm

[English](README.md) | Español

Política de implementación compartida para los providers de títulos de sesión respaldados por modelo. Resuelve la ruta auxiliar, enmarca como JSON los mensajes humanos seleccionados exactos, registra la petición despachable exacta, aplica una instrucción de título sensible al idioma, hace cumplir los presupuestos de entrada y salida, compone el timeout y la cancelación del llamador, ensambla el stream y devuelve texto normalizado con los seq de origen exactos más la ruta provider/model usada para generarlo.

Este paquete es una librería, no un plugin de Cordis. Los plugins provider llaman a `registerSessionTitleLlmProvider()` con su cadencia y su selector de mensajes; la función valida la configuración compartida y delega cada revisión en `generateSessionTitleWithLlm()`, de modo que el comportamiento de registro, ruta, prompt, cancelación y validación no puede divergir entre ellos.

## Contrato de ruta y fallo

Los overrides `provider` y `model` son opcionales, pero deben suministrarse juntos como cadenas no vacías. Sin ese par, el helper usa la ruta provider/model exacta capturada del `request/header` registrado de la sesión en curso; por tanto, un refresh explícito antes de que exista ruta alguna necesita overrides. El helper mide el prompt de usuario final enmarcado en JSON — incluidos los campos seq, los envoltorios y el escape JSON — contra `maxInputBytes` antes de registrarlo o despacharlo, en lugar de truncarlo. El timeout y la cancelación del llamador se vuelven a comprobar mientras se consume el stream y después de que termine, de modo que un resultado tardío con éxito no puede aceptarse aunque un interceptor o adaptador ignore el abort. La salida malformada o vacía, las llamadas de herramienta y las razones de fin distintas de stop también se rechazan; el servicio de títulos de sesión decide si ese rechazo es una advertencia automática o un fallo explícito del llamador.

Tras la validación de ruta y entrada, el helper anexa un evento `session/title-llm-request` solo de registro directamente a través de `Session` antes del despacho al modelo. Contiene el id del title-provider, los seq de origen exactos, la ruta, el prompt de sistema, la lista de mensajes y el tope de tokens de salida que usa la llamada. La persistencia observa el registro de forma eager; la anexión no necesita marcador específico de título, cast, cola de resolución ni flush. El sobre despachado está congelado en profundidad, lleva `purpose: 'session-title'` y carece deliberadamente de la identidad de petición local al proceso de dsh-agent-loop. Los interceptores permanecen alineados con el registro, mientras que los observadores de reconstrucción solo de loop no lo comparan con la cabecera de conversación. El adaptador de DeepSeek mapea ese purpose a pensamiento deshabilitado (thinking-disabled) para que el pequeño presupuesto de salida se reserve para el texto visible del título; otros adaptadores son dueños de su comportamiento específico de purpose. Un fallo posterior del modelo deja intacto el registro de la petición; los fallos de validación que nunca llegan a ser peticiones despachables no crean ninguno. El evento permanece fuera del historial de modelo derivado.

## Configuración

Todos los campos son obligatorios salvo el override de ruta en par; la librería no tiene valores predeterminados.

| Clave | Contrato |
|---|---|
| `targetWords` | Recuento objetivo de palabras positivo para títulos que no son CJK. |
| `targetCjkCharacters` | Recuento objetivo de caracteres positivo para títulos en chino, japonés o coreano. |
| `maxInputBytes` | Techo de bytes UTF-8 positivo para el prompt de usuario final enmarcado en JSON. |
| `maxOutputTokens` | Tope de tokens positivo para la generación auxiliar. |
| `timeoutMs` | Plazo positivo de extremo a extremo dentro del límite de temporizador del runtime. |
| `provider`, `model` | Ruta explícita opcional; ambos o ninguno. |

## Model Experience

### Solicitud de título auxiliar

#### Lo que ve el modelo

El modelo de títulos recibe una instrucción de sistema fija para devolver un título único, conciso y sin adornos en el idioma de entrada, incluidos los objetivos configurados de palabras y de caracteres CJK. Su único mensaje de usuario contiene un array JSON de los mensajes humanos seleccionados exactos y sus seq.

#### Efecto en tokens

La petición auxiliar consume tokens según el tamaño de la entrada seleccionada y `maxOutputTokens`. Es independiente de la petición principal del agent y no añade texto de título ni enmarcado al historial del agent. Las llamadas de título de DeepSeek deshabilitan el pensamiento (thinking); la conversación principal conserva su modo de pensamiento configurado.

#### Efecto de KV Cache

Sin invalidación de la petición principal. La reutilización de la caché auxiliar es específica del provider; la instrucción fija es reutilizable mientras el array JSON de mensajes cambia con cada revisión.

## Limitaciones conocidas y trabajo diferido

- El helper acepta solo salida de texto y rechaza las llamadas de herramienta; no se exponen adaptadores de salida estructurada ni variantes de prompt específicas del provider.
- Aplica un techo de bytes para todo el prompt de usuario enmarcado en lugar de recortar mensajes individuales o aplicar una política de retención.
