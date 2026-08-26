# Agent Note: Composición de respuesta personalizada en multiselección

Status: implemented

[English](2026-07-30-multi-select-custom-answer-composition.md) | Español

## Problema

El vocabulario de resultados de user-questions lleva las etiquetas de opciones seleccionadas y el texto personalizado opcional en campos separados, pero su semántica original los hacía mutuamente excluyentes para toda pregunta. En una pregunta de multiselección, abrir o teclear la respuesta personalizada descartaba etiquetas que el usuario ya había seleccionado. La TUI devolvía solo el texto personalizado, y el host Web rechazaba una respuesta del cliente que preservara ambos campos.

## Decisión

Para una pregunta con `multiSelect: true`, un elemento de respuesta puede contener tanto un array `selected` no vacío como texto `custom` no vacío. Los borradores Web preservan ambos valores sin importar si el usuario selecciona una opción o teclea primero el texto personalizado; la TUI retiene el texto personalizado pendiente a través de los cambios de modo opción/personalizado y lo proyecta con las etiquetas marcadas desde cualquiera de los modos de envío; y el host Web acepta la respuesta combinada tras aplicar su validación existente de id, etiqueta, unicidad, lote y texto no vacío.

Las preguntas de selección única y sin opciones conservan la semántica exclusiva: el texto personalizado sobrescribe cualquier opción seleccionada. La forma del resultado sigue siendo `{ id, selected, custom? }`, así que no cambian ni el schema de wire ni el de salida de herramienta.

## Alternativas consideradas

**Codificar el texto personalizado como otra etiqueta `selected`.** Rechazada porque borraría la distinción entre etiquetas de opción proporcionadas por el llamador y texto escrito por un humano, debilitando la validación y forzando a los consumidores a inferir qué valor era personalizado.

**Permitir `selected` y `custom` juntos para toda pregunta.** Rechazada porque una pregunta de selección única representa una sola respuesta; permitir una opción seleccionada más texto personalizado haría ambigua su cardinalidad. La forma combinada se limita a preguntas que se adhieren explícitamente a respuestas múltiples.

## Consecuencias

Las UIs de multiselección pueden representar la respuesta completa del usuario sin descartar ninguna de las dos fuentes. Los providers y los consumidores conservan el DTO existente, mientras que los validadores conscientes de la petición interpretan la combinación permitida a partir de `multiSelect`. La cobertura del componente Web y del navegador ensamblado, la cobertura de TUI, la cobertura de respuesta del host y la cobertura de proyección de herramienta fijan el resultado combinado. Las coberturas Web, de TUI y de proyección de herramienta también retienen respuestas de solo etiquetas; la cobertura ensamblada de TUI sin clave fija el flujo combinado de terminal, y la cobertura del host para selección única fija la regla de exclusividad restante.
