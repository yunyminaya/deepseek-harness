# Agent Note: Devolución del razonamiento de DeepSeek en cada turno razonado

Status: implemented

[English](2026-08-19-deepseek-reasoning-passback-every-turn.md) | Español

## Problema

`dsh-llm-deepseek` reproducía `reasoning_content` en el historial solo en turnos de assistant que también portaban llamadas de herramienta. La guía del modo de pensamiento de DeepSeek exige el campo ahí y lo ignora en cualquier otro sitio, así que retenerlo en turnos planos compraba tokens de entrada sin nada observable perdido frente a `api.deepseek.com`.

Ese endpoint no es el único al que sirve este adapter. `Config.baseURL` lo apunta a cualquier endpoint compatible con OpenAI, incluido un gateway que recodifica una conversación de chat-completions de DeepSeek para otro proveedor. Un gateway así no tiene ranura de wire para la firma de pensamiento ascendente y la recupera calculando el hash de la cadena de pensamiento reproducida. Un turno que el modelo respondió sin llamar a una herramienta llegaba por tanto al gateway sin texto de razonamiento alguno, la búsqueda de firma no encontraba nada y la conversación reconstruida divergía de la registrada. Las ejecuciones de agent llaman a herramientas en la mayoría de los turnos, así que la pérdida solo aparecía en turnos de respuesta plana y parecía intermitente.

## Decisión

`serializeAssistant` emite `reasoning_content` para todo turno de assistant cuyo contenido portara razonamiento, con independencia de las llamadas de herramienta. Un bloque de razonamiento ausente sigue sin emitir campo, de modo que un turno sin pensamiento no cambia.

El texto reproducido es byte exacto con lo que transmitió el provider: `translate.ts` acumula todo el canal `reasoning_content` de una respuesta en un único bloque de razonamiento, de modo que la unión en `serializeAssistant` concatena un miembro, y un hash calculado sobre la reproducción coincide con un hash calculado sobre la entrega original.

## Alternativas consideradas

- **Un conmutador `Config` que seleccione la política de devolución.** Los dos comportamientos de endpoint son reales, pero el campo es inerte donde no se necesita, así que el conmutador solo compra de vuelta la cadena de pensamiento de un turno en tokens de entrada —contra un ajuste erróneo que vuelve silenciosamente irreconstruible una sesión, sin error en ninguno de los dos extremos al que atribuirlo. Una perilla cuya posición errónea falla en silencio es peor que los tokens.
- **Decidir desde `baseURL`.** Si un endpoint reenvía a otro proveedor no se puede leer de su host: un endpoint interno puede hacer de proxy directo de DeepSeek y uno público puede reenviar. El adapter estaría adivinando sobre un despliegue que no puede ver a través.
- **Portar la firma de forma durable en su lugar, como hace `dsh-llm-pi-ai`.** Ese adapter persiste `thinkingSignature` por bloque en su estado de replay porque sus providers ponen la firma en el wire. DeepSeek chat-completions no expone ninguna, así que este adapter no tiene nada que persistir y el texto reproducido es el único canal.

## Consecuencias

Cada turno razonado sin llamada de herramienta cuesta ahora su cadena de pensamiento en tokens de entrada en las peticiones posteriores. El texto añadido se sitúa en la posición de ese turno y es idéntico en cada petición subsiguiente, de modo que el prefijo ensamblado permanece estable y solo la primera petición que cruza el cambio pierde la reutilización de caché a partir de ese punto.

`WireAssistantMessage.reasoning_content` documenta ambos comportamientos de endpoint, y el README del paquete enuncia la regla de devolución en las notas del formato Wire y en las secciones de tokens y caché de Model Experience.

## Pruebas

`tests/serialize.spec.ts` fija las tres formas de assistant: razonamiento junto a texto sin llamada de herramienta, razonamiento junto a una llamada de herramienta, y un turno solo de razonamiento cuyo contenido permanece `""`. Los turnos sin razonamiento siguen sin emitir campo, lo que cubren los casos sin contenido y solo con llamada de herramienta.
