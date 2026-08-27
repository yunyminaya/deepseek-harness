---
name: dsh-prose-standard
description: Úsalo al redactar, revisar, restaurar, recortar o auditar prosa en el repo deepseek-harness, incluido decidir dónde se requiere documentación o comentarios en Markdown, JSDoc, comentarios de código y tests, prompts, descripciones, diagnósticos y cadenas de CLI o UI.
---

# Estándar de prosa de DeepSeek Harness

[English](SKILL.md) | Español

Escribe lo suficiente para preservar el contrato y luego elimina reasoning transcripts, repetición y decoración.
Un contrato es una obligación, invariant, precondition, postcondition o promesa de compatibilidad de la que depende un caller, callee, implementer, producer o consumer.
Este skill posee el juicio editorial y la cobertura obligatoria de prosa; usa [dsh-doc-standards](../dsh-doc-standards/SKILL.es.md) para ubicación, budgets, pares bilingües y documentation gates, y [dsh-trim-cot-leakage](../dsh-trim-cot-leakage/SKILL.es.md) para encontrar y corregir fugas de reasoning transcript.
Es guía, no script.

Trata `contract`, `boundary`, `shape`, `surface`, `seam`, `gate` y `vocabulary` como términos que debes comprobar antes de usar, no como palabras prohibidas.
Primero pregúntate si la regla exacta, API, conjunto de campos, tipo, validación, punto temporal, división de componentes o fallo expresa mejor el hecho.
Conserva un término cuando nombre el sujeto técnico exacto, incluidos los contratos caller/callee y los límites de seguridad/proceso.

Los comentarios describen contratos o justificaciones no obvios que el código no puede expresar; no reformulan lo que el código ya implica.

## Entradas y exclusiones

Exige un `scope` explícito.
Si falta, informa la entrada requerida y detente; no infieras un alcance a nivel de repositorio ni inicies una entrevista.

Acepta `mode: automatic | interactive`; por defecto usa `automatic`.
Entra en modo interactivo solo cuando el usuario pida explícitamente preguntas o calibración.

`mode` controla las preguntas, no la autoridad de escritura.
Las tareas de review y auditoría informan hallazgos sin editar; las tareas que pidan explícitamente escribir, corregir o recortar aplican cambios claros.

Excluye siempre `vendor/` del descubrimiento, la review y las ediciones, incluso cuando el alcance solicitado sea todo el repositorio.
No sigas un symlink hacia allí.
Pon las exclusiones después de los globs de inclusión para que una inclusión posterior no pueda readmitirlo: por ejemplo, termina los comandos ripgrep con `--glob '!vendor/**'` y da a los comandos Git un pathspec explícito `:(exclude)vendor/**`.
Si el alcance solicitado contiene solo `vendor/`, informa que no quedan archivos elegibles.

Excluye también `.agents/notes/archived/` de la review y edición de prosa.
Las Archived Agent Notes son instantáneas congeladas; inspecciona un objetivo exacto solo para entender una cita histórica entrante, nunca para modernizar su prosa o sus enlaces salientes.

Trata los catálogos generados, snapshots y fixtures como derivados.
Edita primero la fuente o escenario propietario y luego regenera el artefacto.
Cuando un generador extraiga un resumen de la prosa propietaria, haz que la frase extraída quede completa para esa superficie.
Los pares bilingües no tienen propietario permanente: cualquiera de los idiomas puede ser el lado redactado de una actualización.
Sigue la [ruta rutinaria ligera](../../../docs/AGENTS.es.md#writing-rules), actualiza mínimamente la contraparte y vuelve a registrar el par.

## Conserva la proposición completa

Antes de editar, identifica cada proposición del pasaje.
Conserva cada elemento relevante:

- actor y acción;
- condición, momento y orden;
- modalidad como must, may o never;
- garantía negativa y excepción;
- propiedad, efecto lateral, modo de fallo y consecuencia.

Elimina adjetivos, repetición y narración solo cuando sobreviva cada cláusula factual y el resultado sea más claro.
Un recuento de palabras menor por sí solo no es una mejora.

Conserva un contrato local completo en el punto de uso: comportamiento, fallo, propiedad y consecuencia que un caller o maintainer necesita ahí.
Enlaza agresivamente al documento propietario para arquitectura, justificación, algoritmos, historia o ejemplos ampliados.
Una explicación tiene un solo hogar; los hechos esenciales del contrato pueden repetirse localmente.

Conserva la justificación no obvia cuando omitirla podría causar plausiblemente un mal uso o una simplificación incorrecta.
En caso contrario, declara la consecuencia y enlaza al hogar de la justificación.

## Cobertura obligatoria por ubicación de la prosa

No es un pase unidireccional de recorte.
Añade o restaura prosa cuando el código, los tipos y la estructura no comuniquen un contrato obligatorio de los siguientes.
No añadas un comentario cuando esos hechos ya resulten obvios localmente.

- **JSDoc público:** documenta distinciones caller-visible de retorno, throws o rejections, efectos laterales, propiedad, timing, cancelación y durabilidad.
- **Comentarios internos:** orienta estructura no local y estructura local obviamente complicada, incluidos invariants, orden de carreras, propiedad, límites de seguridad y comportamiento sorprendente ante fallos.
Elimina la narración del flujo de control y la reformulación del código.
- **Comentarios de módulo:** declara el rol del módulo, sus dependencias, responsabilidades y decisiones de arquitectura no obvias; enlaza las decisiones de arquitectura a su explicación propietaria.
- **Tests:** explica solo diseño de tests no obvio — por qué hace falta un fixture, assertion, adaptación por plataforma, ruta real de entrada u observación indirecta.
Elimina walkthroughs e inventarios.
- **Cookbooks:** incluye prerrequisitos, acciones requeridas, ruta real de entrada, verificación observable y advertencias concisas.
- **READMEs:** incluye el contrato del consumer: configuración, semántica, fallos, limitaciones, puntos de extensión y efectos model-visible.
Cita texto estable model-visible cuyo propietario sea el paquete; enlaza catálogos generados y propietarios cross-package.
Conserva lagunas duraderas y trampas para maintainers, no inventarios ordinarios de limpieza.
Sigue los [requisitos del README de paquete](../../../docs/cookbook/adding-a-package.es.md#4-write-the-package-readme).
- **Agent Notes:** conserva justificación única, mecanismos, alternativas, consecuencias, evidencia enviada de verificación y lagunas de cobertura con nombre.
Las Agent Notes implementadas declaran la realidad enviada en tiempo presente; elimina checklists de planificación, no la evidencia de lo que fija la decisión.
- **Postmortems:** conserva la secuencia del incidente, la evidencia, la cadena causal, el impacto y la prevención.
Elimina persuasión repetida o detalle de implementación que no establezca causalidad.
- **Skills e instrucciones de agentes:** declara guardarraíles de comportamiento y límites explícitos de alcance como “guía, no script/checklist”.
Mantén el flujo conciso y enlaza su fuente de verdad.
- **Examples y comentarios de configuración:** explica límites de acceso, cableado u orden de carga no obvios, postura de seguridad, comportamiento de replay, excepciones y mal uso probable.
No narres entradas que la configuración ya muestra.
- **Prompts y cadenas visibles:** trata el texto como comportamiento.
Inspecciona la salida generada y ejecuta validación del comportamiento o explica por qué no aplica snapshot.
- **Diagnósticos:** nombra el sujeto o ruta que falla, la regla violada y la corrección cuando no sea obvia.
Elimina narración de ejecución interna.

Conserva nombres de mecanismos que puedan buscarse y el énfasis significativo de modalidad, tiempo o negación.
Normaliza solo el énfasis decorativo.

## Flujo

1. Confirma el alcance, el modo, la rama actual o base del PR y los archivos `AGENTS.md` aplicables.
No inspecciones ramas no relacionadas.
2. Lee [el estándar de documentación](../../../docs/AGENTS.es.md) y el código o documento propietario antes de juzgar un pasaje.
Para calibración o casos desconocidos, lee [los ejemplos destilados](references/examples.es.md).
3. Inspecciona el alcance solicitado, no solo los archivos más grandes.
Usa búsquedas y recuentos de palabras para encontrar candidatos y luego juzga los pasajes semánticamente.
4. Clasifica cada candidato como keep, add, trim, restore, restructure o defer.
Aplica cambios claros solo cuando la tarea autorice ediciones; no fabriques cambios para satisfacer un objetivo de borrado.
5. Actualiza al propietario antes que a los artefactos derivados.
Vuelve a comprobar pasajes análogos después de aprender una regla nueva.
6. Ejecuta las comprobaciones estrechas relevantes, los documentation gates, `git diff --check` y tests de comportamiento para cadenas visibles.
Verifica que el diff final no contenga ninguna ruta `vendor/` e informa cualquier coincidencia accidental en vendor en vez de afirmar un historial limpio de exclusión.
7. Informa el alcance inspeccionado, los cambios claros, las conservaciones deliberadas, los casos diferidos y las comprobaciones realmente ejecutadas.

## Decisiones limítrofes

Un caso es limítrofe solo cuando al menos dos versiones satisfacen la regla de proposición completa pero intercambian principios aceptados, y este skill no resuelve ya ese intercambio.
Una reescritura con una sola respuesta que preserva las proposiciones no es limítrofe.

En modo automático, aplica ediciones claras cuando estén autorizadas e informa casos genuinamente limítrofes sin hacer preguntas.
No debilites una proposición para avanzar.

En modo interactivo, agrupa pasajes análogos bajo el principio rector.
Presenta dos o tres versiones viables, recomienda una y declara la diferencia factual o estructural.
No ofrezcas distractores inferiores.
Usa el canal pedido por el usuario; al calibrar un PR mediante comentarios inline, coloca la versión provisional recomendada en el diff y adjunta las alternativas a esa línea exacta.

Después de que el usuario decida, destila el principio y las versiones en [los examples](references/examples.es.md), sin historia de PR ni narración de reviewers, y aplica la regla aprendida a cada pasaje análogo dentro del alcance.
