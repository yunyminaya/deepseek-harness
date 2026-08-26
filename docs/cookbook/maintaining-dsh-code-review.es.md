# Mantenimiento del skill dsh-code-review

[English](maintaining-dsh-code-review.md) | Español

El skill (destreza) [`dsh-code-review`](../../.agents/skills/dsh-code-review/SKILL.es.md) se mantiene al día gracias a un único operador designado que ejecuta una herramienta de mantenimiento periódica y privada. Este recetario es el punto de entrada para ese operador — y para cualquiera que asuma el rol — y para quienes contribuyen al repositorio y quieren entender por qué las actualizaciones del skill llegan como PRs (pull request) pequeños y periódicos en lugar de auditorías puntuales. El propio flujo de trabajo está especificado en el [Agent Note de mantenimiento del skill de revisión humana](../../.agents/notes/proposed/process/2026-07-13-human-review-skill-maintenance.es.md).

## Lo que recibe el mantenedor

El operador invoca el wrapper manualmente, a diario con un solapamiento de dos días UTC; una ejecución semanal manual de recuperación usa una ventana de siete días. El flujo de trabajo:

1. Selecciona los PRs fusionados en la ventana elegida (por defecto dos días UTC para la cadencia diaria y siete para la semanal) cuyo commit de fusión sea alcanzable desde `origin/master`. Los PRs cuyo commit de fusión no sea alcanzable (ramas apiladas cuyo padre fue aplastado (squash)) o que superen un tope de adquisición de 250 commits se registran en `skipped-pulls.json` y se omiten en lugar de abortar la ejecución.
2. Recopila los comentarios de revisión humana previos a la fusión con anclas de commit (comentarios en línea y envíos de revisión) y luego compara los parches de los PRs en el momento de la retroalimentación y en su forma final fusionada. No adquiere los comentarios de conversación de los PRs porque el estado actual de GitHub no puede darles una línea base de tiempo de retroalimentación segura frente a force-push, y excluye de la evidencia de adopción los cambios exclusivos de la rama destino.
3. Dos adaptadores revisores configurados de forma independiente clasifican quién escribió cada elemento y si el cambio lo adoptó, y luego clasifican los elementos adoptados de común acuerdo contra el skill actual.
4. El adaptador principal redacta un `SKILL.md` revisado completo; ambos adaptadores revisan el mismo diff; los hallazgos bloqueantes se repiten en bucle hasta que ambos aprueban.
5. `pnpm run doc-sync` y `pnpm run lint` se ejecutan contra el candidato antes de que la herramienta declare el éxito.

Cada ejecución guarda sus artefactos en la máquina del operador. El diff guardado, el `SKILL.md` candidato y el manifest de promoción van a parar a `~/dsh-code-review-outputs/` con nombre por marca de tiempo. El manifest registra el commit maestro de origen y el blob del skill, los IDs y URLs de los comentarios de origen, los rangos de evidencia aterrizados, los veredictos de los adaptadores y los resultados de los gates; la E/S en bruto de cada adaptador permanece en un directorio temporal privado cuya ruta se escribe en la notificación y en el registro diario bajo `~/Library/Logs/dsh-code-review-maintainer/`. El propio worktree de mantenimiento se restaura limpio después de cada ejecución para que el operador nunca se sienta tentado a editar la copia de mantenimiento en el lugar.

## Qué hace el operador con un diff candidato

Cuando una ejecución produce un candidato, llega una notificación de macOS con una pista `dsh-code-review-promote <timestamp>`.

1. **Lee el diff por sus propios méritos.** No te remitas a «los revisores lo aprobaron»; el contrato del mantenedor es que el operador toma la decisión final. Busca listas de verificación infladas, prosa histórica, extrapolaciones sin respaldo a partir de un único incidente y cobertura duplicada con el skill existente o el contenido de los documentos autoritativos.

   ```sh
   ls ~/dsh-code-review-outputs/                         # every candidate ever produced
   less ~/dsh-code-review-outputs/2026-07-16T02-00-00Z.diff
   less ~/dsh-code-review-outputs/2026-07-16T02-00-00Z.SKILL.md
   less ~/dsh-code-review-outputs/2026-07-16T02-00-00Z.manifest.json
   ```

2. **Contrasta con los artefactos de la ejecución.** El manifest de promoción asigna cada regla propuesta a la retroalimentación de origen y a la evidencia aterrizada; la E/S detallada de cada adaptador, el consenso y la evidencia adoptada viven bajo el directorio temporal privado de la ejecución (la ruta aparece en el registro). Haz una comprobación puntual de al menos un candidato: ¿el comentario humano enlazado respalda de verdad la regla añadida? ¿El PR enlazado la adopta de verdad?

3. **Decide una de tres:**
   - **Descartar.** Borra el candidato guardado. La herramienta vuelve a considerar la misma retroalimentación en la siguiente ejecución bajo lo que diga entonces el skill actual.

     ```sh
     rm ~/dsh-code-review-outputs/2026-07-16T02-00-00Z.{diff,SKILL.md,manifest.json}
     ```
   - **Agrupar (batch).** Aparta el candidato si la actualización es pequeña y podría combinarse con una futura. La comprobación del skill de origen sigue aplicando; vuelve a ejecutar el análisis o haz rebase manual y vuelve a revisar el diff si `master` cambia antes.
   - **Promover (promote).** Desde un checkout limpio de `master` del repositorio, ejecuta el helper de promoción. Actualiza `master`, verifica que el skill actual coincide con el blob de origen registrado, aplica el diff guardado y abre un PR en borrador cuyo cuerpo lista las URLs o IDs de la retroalimentación de origen, el rango de commits aterrizados, la ejecución de origen, las comprobaciones y las ediciones del operador. Se detiene ante la deriva del skill (drift) en lugar de sobrescribir orientación más nueva; el operador sigue revisando el PR en GitHub y lo fusiona o lo cierra.

     ```sh
     cd ~/path/to/deepseek-harness   # clean master
     dsh-code-review-promote 2026-07-16T02-00-00Z
     ```

4. **No hagas commit del resultado del adaptador tal cual.** Las ediciones pequeñas durante la promoción — ajustar la redacción, eliminar un ejemplo que solo tiene sentido con el contexto del PR de origen, plegar una regla dentro de otra existente — son esperables y preservan el «criterio del revisor» del que depende el flujo de trabajo. Haz amend de la rama antes de fusionar.

## Cuando una ejecución no produce ningún candidato

Ese es el caso habitual después de que cada etapa de clasificación no vacía haya producido al menos un resultado de adaptador válido. La herramienta registra «sin candidato» en su registro diario, no envía ninguna notificación (para evitar la fatiga de alertas) y sigue adelante. Los días sin actualización del skill son el flujo de trabajo comportándose correctamente, no un atasco.

## Interrupciones y traspaso

El mecanismo vive en una sola máquina. El operador gestiona las interrupciones según surgen:

- **Ejecución diaria omitida.** La ventana de solapamiento de dos días cubre automáticamente un día saltado; las ausencias más largas se recuperan ejecutando el wrapper manualmente con `DSH_CODE_REVIEW_SINCE=<Nd>`. Las ventanas solapadas son idempotentes: la orientación ya presente en el skill actual se clasifica como `covered` y no vuelve a entrar como candidata.
- **Caída del provider del adaptador.** La herramienta se niega a ejecutarse cuando los dos comandos de revisión se resuelven a ejecutables idénticos byte a byte. Un único lote cuya respuesta de adaptador falle la validación de schema o de id se cierra con fallo a nivel de lote (cada elemento del lote marcado como unclear) y la ejecución continúa; la salida en bruto se conserva para depurar. Si cualquiera de los adaptadores no produce ningún resultado válido para ningún lote no vacío en una operación, la ejecución falla, escribe un registro de fallo y notifica al operador; nunca colapsa una caída total del provider en «sin candidato».
- **Traspaso a otro mantenedor.** Abre un Agent Note de seguimiento que supere al actual: o mueves el mecanismo al repositorio o registras la configuración privada del nuevo operador. No transfieras la herramienta en silencio: el «factor bus de un único mantenedor» en la sección de Riesgos del Agent Note es la razón por la que el traspaso necesita una decisión documentada.

## Dónde vive la configuración privada del operador

El código fuente de la herramienta, los adaptadores revisores, las credenciales del provider y el programador son infraestructura privada del operador y quedan fuera de este repositorio por diseño (consulta la sección «Where the mechanism lives» del Agent Note). Este recetario y el Agent Note describen **qué garantiza el flujo de trabajo**; **cómo** se implementan esas garantías es una cuestión de infraestructura privada. Si eres el nuevo operador, las secciones `## Proposal` del Agent Note son la especificación contra la que construyes.
