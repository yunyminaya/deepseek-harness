# Agent Note: Proyectar el contenido inyectado verbatim, eliminando los envoltorios XML

Status: implemented

[English](2026-07-20-unwrap-injected-content-envelopes.md) | Español

## Problema

Dos familias de contenido de sesión inyectado se renderizaban en el transcript del modelo envueltas en envoltorios XML: `steering/message` como `<steering source="…">…</steering>` y `context/message` como `<context source="…">…</context>` (este último con un opt-out `'raw'` que se saltaba el envoltorio). Los envoltorios pretendían decirle al modelo «esto está inyectado, no habla el usuario».

Dos problemas:

- **Ningún modelo está entrenado con estas etiquetas.** `<steering>` y `<context>` son marcado arbitrario que a ningún modelo se le enseñó a leer, así que el encuadre añade tokens sin un efecto fiable y puede engañar activamente — los transcripts registrados muestran un modelo que trata una instrucción `<steering>` como metadatos de terceros, la rechaza y responde solo al prompt original.
- **La superficie de sesión es la capa equivocada para el encuadre.** La superficie proyecta el log durable en el transcript del modelo; decidir cómo se redacta el contenido no es su trabajo. Un llamador que quiere un marco concreto formatea su propio contenido antes de inyectarlo — lo que el único productor pesado (`agent-instructions`) ya hace, siendo propietario de su marco `<system-reminder>` completo y optando por no usar el envoltorio `<context>` con `envelope: 'raw'`. La maquinaria de etiquetas restante (`ContextEnvelope`, un campo `envelope` que atraviesa `InjectOptions`, `HookContext`, el evento `context/message` y el loop) servía a una distinción que pertenece al llamador.

## Decisión

El contenido de sesión inyectado se proyecta verbatim; el llamador es propietario de cualquier encuadre. `deriveEventMessage` renderiza los bloques de contenido de `user/message` al modelo sin cambios; `source` permanece en el log de eventos durable pero no se renderiza.

El tipo `ContextEnvelope` y todos los campos `envelope` se eliminan — `context/message` en `SessionEventMap`, `InjectOptions`, `HookContext` y la plomería `inject()`/`additionalContexts` en `dsh-agent-loop`. `agent-instructions` ya no solicita `'raw'`; su contenido auto-enmarcado se renderiza como antes. Los helpers `renderTagged`/`renderContextEnvelope` se eliminan. `context/message.meta` sigue llevando estado JSON durable oculto al modelo.

La atribución `source` que llevaban los envoltorios no se pierde — permanece en los eventos durables; simplemente ya no se renderiza en el transcript.

## Alternativas consideradas

- **Conservar el envoltorio `<context>`, desenvolver solo el steering** — deja viva la maquinaria `ContextEnvelope`/`envelope` para un bit de encuadre que ningún modelo lee, y conserva la inconsistencia de que el productor principal ya opta por no usarla.
- **Conservar el campo envelope solo para contenido de plugins** — divide una proyección en dos según `source.kind` sin beneficio observado; un plugin que dirige al agente (razones de continuación de hook-bridge) también quiere que la instrucción se siga, no que se etiquete.
- **Mover el desenvoltura a los adaptadores** — la proyección canónica es el contrato visible para el modelo («visible para el modelo ⟺ registrado»); la divergencia por adaptador en el encuadre haría el transcript derivado dependiente del adaptador. El encuadre que un llamador quiere de verdad pertenece al contenido del llamador, no a un adaptador.

## Consecuencias

- El steering a mitad de turno y el contexto inyectado llegan al modelo con el mismo peso que un prompt de usuario ordinario.
- El transcript ya no distingue el contenido inyectado de un mensaje de usuario; los consumidores que necesitan la distinción leen el log de eventos durable, que conserva intactos los tipos de evento, `source` y `meta`.
- Las instantáneas ACP `hook-{cc,codex}-stop-continue` se regrabaron: las grabaciones antiguas capturaban al modelo rechazando el steering como metadatos de terceros, exactamente el modo de fallo que el arreglo corrige.
- La cláusula de envoltorios etiquetados del [Agent Note de vocabulario de bloques de contenido](../architecture/2026-06-11-content-block-vocabulary.es.md) se enmienda para apuntar aquí.

## Diferido

`agent-instructions` ya enmarca su propio contenido: emite un bloque `<system-reminder>…</system-reminder>` completo como contenido del mensaje en lugar de apoyarse en un envoltorio a nivel de superficie. Ese patrón propiedad del llamador es el que hay que conservar — la superficie hace pasar el contenido verbatim, y cualquier encuadre vive en el contenido del propio productor.

Existían dos vías de encuadre — el encuadre horneado por el llamador (el `<system-reminder>` de `agent-instructions`) y el envolvimiento a nivel de superficie (`<context>`/`<steering>` añadidos por `deriveEventMessage`). Este cambio elimina la segunda, dejando solo el encuadre propiedad del llamador. Si se quiere de nuevo el encuadre etiquetado, unifícalo a través del mapa `meta` del evento — el campo de metadatos adjunto por el productor y oculto al modelo — consumido por un renderizador o adaptador dedicado, en lugar de re-hardcodear una etiqueta en `deriveEventMessage`. Un productor declara el marco que quiere en `meta`; un renderizador lo aplica; la proyección de superficie de sesión sigue siendo un paso-through verbatim.
