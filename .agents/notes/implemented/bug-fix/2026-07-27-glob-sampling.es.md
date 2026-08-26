# Agent Note: Muestrear los resultados glob que superan el tope a lo largo del árbol

Status: implemented

[English](2026-07-27-glob-sampling.md) | Español

## Problema

Al preguntarle a un agent qué contenía un workspace, describió una subcarpeta como si fuera todo el proyecto. El workspace tenía 22 entradas de nivel superior y 11.485 archivos. `glob {"pattern":"*"}` emparejó 10.030 rutas, pero las 100 rutas inline estaban todas bajo un subárbol desempaquetado recientemente, así que el modelo nunca vio las otras 21 entradas.

Tres comportamientos individualmente válidos compusieron la falsa impresión. Un glob sin `/` empareja nombres base a cualquier profundidad, así que `*` significa todos los archivos del árbol y no la expansión del directorio actual de la shell. El `--sort=modified` de ripgrep es ascendente, así que las marcas de tiempo antiguas restauradas de un archivo pusieron ese subárbol primero. La página inline tomó entonces la cabeza de ese orden sin decir que representaba solo un segmento concentrado.

## Decisión

Un resultado que cabe dentro de `globMaxResults` sigue siendo completo y ordenado byte a byte por tiempo de modificación. La configuración obligatoria `sampleOverCapGlobResults` no tiene valor por defecto: `false` conserva la cabeza por tiempo de modificación para un resultado sobre el tope, mientras que `true` muestrea por turnos (round-robin) entre las entradas de nivel superior del resultado completo. En modo muestreo, cada entrada recibe un hueco antes de que cualquiera reciba un segundo, los grupos agotados se retiran, el orden relativo permanece estable dentro de cada grupo, y la agrupación es relativa a la raíz de búsqueda real, incluida una `path` explícita.

En modo muestreo, el pie de página declara que la página es una muestra entre entradas y no la cabeza por tiempo de modificación, y cuántas entradas de nivel superior alcanza cuando ese dato añade información. Cuando existen más entradas de nivel superior que huecos inline, le indica al modelo acotar `path`. El modo cabeza conserva el pie de página ordinario de resultado con tope. Cuando el spill tiene éxito, ambos modos preservan la lista completa ordenada en el artefacto.

El prompt y el schema declaran el orden configurado para resultados sobre el tope, que un patrón sin `/` empareja a cualquier profundidad, y que glob devuelve archivos, nunca entradas de directorio. La composición CLI enviada selecciona explícitamente el modo cabeza; los despliegues que quieren páginas con tope representativas seleccionan el modo muestreo. La orientación por directorios sigue siendo trabajo ordinario de shell en los despliegues que exponen la herramienta bash visible para el modelo: usa `ls` para un directorio, y glob para un patrón de ruta de archivo con nombre a lo largo del árbol. `ctx.fs.listDir` sigue siendo una primitiva interna del provider usada por el descubrimiento de skills; esta decisión no añade ninguna herramienta `list` visible para el modelo.

## Alternativas consideradas

**Conservar la cabeza por tiempo de modificación como único comportamiento.** Rechazada tras medir la forma del fallo. Algunos despliegues necesitan el orden estable, pero un despliegue que valora la orientación en el workspace puede seleccionar explícitamente datos representativos en lugar de pedirle al modelo que desconfíe de las únicas rutas que recibió.

**Darle un valor por defecto a la elección de muestreo.** Rechazada. Ninguna evidencia de producto establece un orden como contrato implícito, así que cada composición selecciona uno y la mala configuración falla en la carga.

**Muestrear todos los resultados.** Rechazada. Un resultado completo no pierde nada por truncamiento, así que el orden por tiempo de modificación sigue siendo útil para preguntas orientadas a la antigüedad. El muestreo comienza solo cuando la cabeza deja de describir el todo.

**Cambiar al orden de más reciente primero.** Rechazada. Simplemente cambia qué subárbol concentrado puede dominar y elimina el contrato existente de más antiguo primero sin hacer representativa una página con tope.

**Muestrear solo pasado un umbral de sesgo.** Rechazada. Ninguna evidencia actual respalda un umbral para todo el despliegue, y el modelo no podría saber qué contrato de orden aplicaba. El tope existente es la transición explicable.

**Equilibrar recursivamente por debajo del nivel superior.** Diferida. El equilibrio por primer segmento corrige el fallo observado; un equilibrio más profundo necesita una política de profundidad-frente-a-amplitud que no está respaldada.

**Añadir una herramienta `list` visible para el modelo.** Rechazada tras la revisión de implementación. La composición de código por defecto ya expone bash general y el modelo entiende `ls`; una herramienta duplicada añadiría tokens permanentes de schema/prompt más contratos de orden, paginación, symlinks, escape, UI e instantánea sin un beneficio de seguridad o política distinto. Los despliegues ligeros sin una herramienta bash visible para el modelo no ganan orientación por directorios con este cambio.

**Rechazar `*` o anclar silenciosamente los patrones sin separador.** Rechazada. El mismo comportamiento de nombre base a cualquier profundidad hace útil `*.ts` a lo largo de un árbol. Documentar la regla conserva la semántica vigente de ripgrep.

## Consecuencias

Una página sobre el tope en modo muestreo ya no responde preguntas de orden por antigüedad desde sus rutas inline; su pie de página lo declara, y el artefacto de spill conserva la vista completa ordenada. El muestreo equilibra solo el primer segmento bajo la raíz de búsqueda, así que un subárbol caliente más profundo puede seguir dominando dentro de una entrada de nivel superior. El modo cabeza conserva el riesgo de concentración como intercambio explícito del despliegue.

La superficie de herramientas no crece. Cada composición debe fijar `sampleOverCapGlobResults`; cambiarla altera el prompt de glob, la descripción del schema y el renderizado Native de resultados sobre el tope. La salida canónica conserva `root` para que el modo muestreo pueda recuperar su base de agrupación, mientras que los resultados que caben permanecen sin cambios.

## Pruebas

Las pruebas del paquete fijan la configuración obligatoria, ambos modos sobre el tope, sus descripciones de prompt y schema, los resultados concentrados y planos, las raíces explícitas, más grupos que el límite de argumentos de JavaScript, los grupos agotados, menos huecos que grupos y las rutas fuera del directorio de trabajo. El escenario ACP `fs-glob-sampling` activa explícitamente el muestreo, arranca una composición real mínima de Loader/app/local-bash y ejecuta el plugin real de búsqueda contra un fixture determinista de proceso `rg`; su resultado abarca cuatro entradas de nivel superior en lugar de devolver la cabeza de un solo subárbol.
