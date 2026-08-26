# Agent Note: Descubrimiento difuso de comandos slash en Web

Status: implemented

[English](2026-08-04-web-slash-command-fuzzy-discovery.md) | Español

## Problema

El menú de comandos web exigía un prefijo del nombre del comando, así que el descubrimiento fallaba cuando un usuario recordaba las letras significativas pero no sus posiciones exactas. Ampliar la coincidencia del menú podía facilitar el descubrimiento, pero la ejecución de comandos debe seguir siendo exacta y determinista: una línea aproximada nunca debe ejecutar un comando cercano.

## Decisión

La fuente de comandos `/` coincide difusamente la consulta tecleada contra los nombres de comando como una subsecuencia ordenada insensible a mayúsculas. Los prefijos exactos forman la clase de mayor rango. Dentro de cada clase, la puntuación de alineación más fuerte premia las fronteras de separador y los caracteres adyacentes mientras penaliza los caracteres iniciales y los huecos; las puntuaciones iguales conservan el orden de directorio del host y de contribución del cliente. El filtrado posicional sigue eliminando los comandos que toman argumentos de los menús en línea antes de la clasificación.

El puntuador usa programación dinámica en tiempo `O(query length × name length)` y memoria `O(name length)` por candidato. La puntuación de candidatos permanece en el cliente y examina solo nombres; las descripciones no afectan a la coincidencia. La selección del menú sigue despachando el nombre exacto seleccionado, mientras que la adjudicación de espacio y Enter siguen exigiendo un token de comando exacto.

## Alternativas consideradas

**Mantener la coincidencia solo por prefijo.** Rechazada porque preserva el fallo de recall que motiva la funcionalidad; `/cpt` no puede descubrir `/compact`.

**Coincidir caracteres desordenados o descripciones.** Rechazada porque las coincidencias desordenadas son difíciles de predecir, mientras que las coincidencias por descripción pueden mostrar comandos cuyos nombres visibles no explican por qué clasificaron ahí.

**Usar una dependencia general de búsqueda difusa.** Rechazada porque esta superficie necesita una sola regla de subsecuencia restringida sobre un catálogo pequeño de comandos; un índice de búsqueda configurable añadiría peso al bundle y comportamiento de clasificación que el producto no usa.

## Consecuencias

Los usuarios pueden descubrir un comando desde letras recordadas en orden, y la clasificación permanece estable a través de catálogos idénticos. La puntuación es deliberadamente heurística: una coincidencia alineada con separadores puede superar a una con un tramo crudo más corto. Las pruebas del paquete fijan cada factor de clasificación y los empates estables, mientras que la instantánea de replay Web ensamblada fija que `/cpt` resuelve a `/compact`. La semántica de ejecución exacta no cambia.
