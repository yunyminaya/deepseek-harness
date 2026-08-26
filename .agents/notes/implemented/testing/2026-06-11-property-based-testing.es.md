# Agent Note: Pruebas basadas en propiedades para código con forma de protocolo

Status: implemented

[English](2026-06-11-property-based-testing.md) | Español

> La suite de propiedades encontró un bug real de `block-end` duplicado de BlockAssembler en su primera ejecución.

## Problema

Las pruebas basadas en ejemplos fijan los casos que se nos ocurrieron. El núcleo del harness tiene forma de protocolo — flujos de chunks, registros de eventos, conversión de schemas, programación de inbox — donde el espacio de entrada es combinatorio y los bugs interesantes viven en intercalados para los que nadie escribió un ejemplo. La evidencia motivadora: un bug de ordenación del ensamblado de bloques sobrevivió una vez al 100 % de cobertura de línea de los caminos felices. La cobertura del 100 % por archivo demuestra que cada línea se ejecutó, no que cada intercalado sea correcto.

## Decisión

`fast-check` (una devDependency raíz) alimenta un `tests/properties.spec.ts` por paquete con forma de protocolo, con generadores ajustados para entradas *realistas pero adversariales* (no ruido uniforme) y `numRuns` contenido para que la suite siga muy por debajo de ~10 s en local. Los fallos imprimen una seed reproducible. (No se entrega un trabajo de CI nocturno que ejecute 100× las iteraciones — la suite de propiedades corre solo en el CI normal de `push`/`pull_request`; un trabajo programado de alta iteración sigue siendo trabajo futuro posible.)

- **dsh-llm / BlockAssembler:** flujos de chunks arbitrarios (válidos + malformados: índices duplicados, rezagados, `block-start` faltante). Invariantes: el recuento de `blocks()` ≤ índices distintos vistos; el reensamblado es idempotente (`blocks()` es estable entre llamadas repetidas y `message().content` lo refleja); `blocks()` nunca lanza y produce solo etiquetas de bloque de contenido válidas; `finish` refleja el último chunk `finish`, con valor por defecto `{kind:'stop'}` cuando no llega ninguno.
- **dsh-session:** registros de eventos arbitrarios. Invariantes: `deriveMessages` determinista; reproducción desde la semilla idéntica; seq estrictamente monótono; los eventos que no son mensajes nunca afectan a la historia derivada; el contenido derivado está desacoplado del registro.
- **dsh-tools:** `ParameterSchemaSpec` arbitrario. Invariantes: el `required` de JSON Schema equivale a las claves `required:true` en cada nivel; la conversión es total para declaraciones válidas; **y la composición con la [validación de argumentos en runtime](../architecture/2026-06-11-runtime-arg-validation.es.md)** — los argumentos generados que satisfacen un spec pasan `validateArgs`, y las corrupciones dirigidas (clave requerida eliminada, nivel superior no objeto) se rechazan. Los casos enfocados cubren cada raíz de valor, solapamiento exacto-uno/sin coincidencia, apertura explícita, valores por defecto crudos y JSON con pérdida. Esto cierra el riesgo de deriva entre compilador/validador/`InferArgs`.
- **dsh-agent-loop:** programas de envío arbitrarios contra un adaptador que nunca se agota, conducidos por la señal de settle de `agent/status` (sin sleeps de reloj de pared). Invariantes: ningún mensaje perdido; los números de turno aumentan estrictamente; las transiciones de estado permanecen en la máquina legal.

## Consecuencias

- La calidad de los generadores es la palanca de valor — los generadores sesgan hacia pools de índices pequeños y cadenas cortas para que las colisiones y los intercalados sean comunes.
- **Ya dio sus frutos:** el flujo de BlockAssembler encontró un bug real — un `block-end` duplicado en el mismo índice reescribía un bloque completado. Corregido (la primera coincidencia gana, en línea con la regla existente de rezagados) con una prueba de regresión dedicada.
- Una falla de propiedad por timeout es un hallazgo, no algo que haya que reintentar. Las propiedades del loop son deterministas por construcción (settle en `agent/status`), así que un cuelgue es un defecto real.
- Las pruebas de propiedades complementan, no reemplazan, las pruebas de ejemplo que fijan ramas específicas para la puerta del 100 % de cobertura.

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
