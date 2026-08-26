# Agent Note: Los checkpoints de compactación usan un registro ingenieril en inglés

Status: implemented

[English](2026-07-31-english-compaction-checkpoints.md) | [中文](2026-07-31-english-compaction-checkpoints.zh.md) | Español

## Problema

Un checkpoint de compactación pasa a formar parte del prefijo durable de la siguiente solicitud al modelo. Cuando una conversación multilingüe lleva al compactador a preservar su material narrativo en el idioma de la conversación, el checkpoint puede introducir una gran cantidad de un idioma ausente del código, la salida de herramientas y el prefijo de razonamiento existente. Ese idioma persiste entonces a través de ciclos de compactación posteriores y puede influir en el registro de razonamiento del modelo de conversación.

## Decisión

`COMPACTION_INSTRUCTION` exige un checkpoint ingenieril interno en idioma inglés. Pide al modelo traducir el material narrativo fuente según sea necesario preservando los literales exactos, incluidas rutas, comandos, errores, identificadores, firmas y citas textuales cuando la exactitud importa. Los encabezados del checkpoint y sus viñetas ingenieriles tersas conservan el formato estructurado existente.

El requisito está integrado en la primera oración de la instrucción de compactación final. El prompt de sistema reproducido, las tools y el historial de conversación permanecen idénticos byte a byte con la solicitud enrutada, así que el cambio conserva la reutilización de prefix-cache propiedad de la [nota de prefix-cache del resumen de compactación](2026-07-21-compaction-summary-prefix-cache-reuse.es.md).

## Alternativas consideradas

- **Dejar el idioma del checkpoint a la conversación reproducida** — rechazada: el checkpoint es un prefijo durable de prompt, así que preservar un registro conversacional transitorio puede amplificarlo a través de compactaciones posteriores.
- **Restringir el idioma del modelo de conversación** — rechazada: la política es para un checkpoint interno, no para la conversación visible del usuario, y una regla para toda la conversación cambiaría innecesariamente la interacción normal.
- **Exigir salida solo ASCII** — rechazada: ASCII es una restricción de conjunto de caracteres más que una restricción de registro ingenieril y distorsionaría innecesariamente literales legítimos y material técnico.
- **Añadir al final una oración separada solo en inglés** — rechazada: declarar el requisito en el contrato de salida inicial de la instrucción es más corto y lo ata directamente al checkpoint que se solicita.

## Consecuencias

- Los checkpoints nuevos normalizan el contexto narrativo al inglés conservando las cadenas exactas de las que dependen el uso futuro de tools y el trabajo con código.
- La estructura existente del checkpoint, el enrutado de compactación y la alineación de caché quedan sin cambios; solo la instrucción de usuario final es diferente.
- La llamada directa de resumen permanece fuera de las instantáneas de transcript porque no emite eventos `assistant/chunk`. La regresión del bucle real afirma en su lugar la instrucción final exacta recibida por la solicitud de resumen.
