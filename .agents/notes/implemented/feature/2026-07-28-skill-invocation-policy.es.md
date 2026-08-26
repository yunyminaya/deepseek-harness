# Agent Note: Política de invocación de skills independiente para modelo y usuario

Status: implemented

[English](2026-07-28-skill-invocation-policy.md) | Español

## Problema

El registro de skills trataba originalmente el descubrimiento como un catálogo del modelo: `ctx.skills.list()` eliminaba los skills deshabilitados para el modelo, mientras que `ctx.skills.get()` seguía siendo un loader de confianza sin filtrar. Eso bastaba para la carga iniciada por el modelo, pero no podía representar skills compatibles con Claude anunciados solo a una persona, solo a un modelo, a ambos o a ninguno. La TUI agravaba el desajuste derivando el autocompletado del usuario de la lista filtrada por modelo y permitiendo que cualquier nombre exacto pasara por `get()`.

El parser local también exponía una ortografía interna camel-case como frontmatter. Soportar los campos negativos consolidados `disable-model-invocation` y positivos `user-invocable` exige una representación de dominio duradera y simétrica sin convertir cada posible clave YAML en un contrato entre paquetes sin tipar.

## Decisión

`SkillSummary` lleva un objeto tipado obligatorio `invocation: SkillInvocationPolicy` cuyos campos `modelInvocable: boolean` y `userInvocable: boolean` son positivos y simétricos. La omisión existe solo en los límites de entrada explícitos: un `SkillRegistration` de runtime sin política y un frontmatter local sin ninguna de las dos claves de invocación resuelven a `{ modelInvocable: true, userInvocable: true }` antes de producir candidatos o definiciones. Las claves de frontmatter futuras permanecen fuera del modelo de dominio hasta que existan un consumidor y un contrato de aplicación; el provider local sigue parseando el frontmatter como un `Record<string, unknown>` abierto y luego proyecta solo los campos reconocidos y sus valores por defecto en la política tipada normalizada.

`ctx.skills.list()` devuelve todos los summaries ganadores y ya no elige una superficie de invocación. `isModelInvocable(skill)` e `isUserInvocable(skill)` leen el campo positivo correspondiente directamente. `ctx.skills.get()` sigue siendo neutral respecto a la política porque los llamadores internos de confianza pueden necesitar cualquier definición, mientras que un consumidor público debe aplicar su propio predicado antes de anunciar o cargar un skill. La herramienta del modelo y la TUI comprueban el summary neutral respecto a la invocación antes de llamar a `get()` y luego vuelven a comprobar la definición cargada, de modo que un nombre denegado nunca llega a la carga de definiciones y un cambio de política entre el descubrimiento y la carga no puede exponer su cuerpo.

El provider local acepta las claves de frontmatter kebab-case exactas `disable-model-invocation` y `user-invocable`. Acepta booleanos YAML más `true`/`false`, `yes`/`no`, `on`/`off` y `1`/`0` insensibles a mayúsculas, coincidiendo con las formas booleanas prácticas que aceptan los skills de Claude. Mapea `disable-model-invocation` al campo positivo inverso y rellena ambos campos positivos con sus valores por defecto incluso cuando ninguna de las dos claves está presente. Una ortografía externa camel-case o un valor de invocación no booleano descarta el skill completo del descubrimiento con una advertencia dirigida; este repositorio previo al lanzamiento no mantiene un alias de compatibilidad en disco. Los datos de invocación fallan cerrado porque ignorarlos daría por defecto permiso y podría exponer el skill en una superficie deshabilitada, mientras que los valores opcionales `whenToUse` y `metadata` mal tipados se omiten porque no deciden la invocación.

El catálogo y el loader `dsh-tool-skill` orientados al modelo aplican `isModelInvocable`. El autocompletado `/skill:` de la TUI y el loader exacto aplican localmente el campo de usuario, de modo que un skill solo de usuario es visible y cargable allí incluso cuando está ausente del descubrimiento del modelo, sin convertir el peer de skill opcional en un import de runtime. El skill inicial sembrado por el launcher que usan las sesiones guiadas de `dsh migrate` y `dsh upgrade` sigue esa misma ruta de TUI y debe seguir siendo invocable por el usuario. El RPC `skill.list` del navegador sirve una referencia seleccionada por el usuario que todavía pide al modelo que cargue el skill, por lo que expone la intersección de los skills invocables por modelo y por usuario; no se añade ningún RPC directo de carga de skills en el navegador.

Estas reglas permiten las cuatro combinaciones:

| Política | Invocación por modelo | Invocación por usuario |
|---|---|---|
| `{ modelInvocable: true, userInvocable: true }` | incluida | incluida |
| `{ modelInvocable: true, userInvocable: false }` | incluida | excluida |
| `{ modelInvocable: false, userInvocable: true }` | excluida | incluida |
| `{ modelInvocable: false, userInvocable: false }` | excluida | excluida |

Esta decisión extiende el [skill system](2026-07-05-skill-system.es.md) y sustituye la limitación de política de invocación registrada por el [comando slash de skills de la TUI archivado](../../archived/feature/2026-07-21-tui-skill-slash-command.md).

## Alternativas consideradas

**Almacenar todo el frontmatter en un `Map` genérico y leer las claves de cadena en `isModelInvocable` / `isUserInvocable`.** Rechazada porque las claves mal escritas, los valores no booleanos y la coerción específica del consumidor cruzarían los límites entre paquetes sin comprobación de tipos. El límite del parser sigue abierto; el modelo de dominio es deliberadamente tipado y estrecho.

**Mantener `ctx.skills.list()` filtrado por modelo y añadir una segunda lista de usuario.** Rechazada porque el descubrimiento, la resolución de duplicados, el caché y el orden son trabajo neutral respecto a la superficie. Un catálogo completo más predicados explícitos impide que esos mecanismos deriven, a la vez que hace visible la política de cada consumidor en su propio límite.

**Aplicar la política de invocación dentro de `ctx.skills.get()`.** Rechazada porque `get()` no puede saber si su llamador es una herramienta del modelo, un comando humano u orquestación de confianza. Filtrar allí haría además imposible inspeccionar o administrar el cuadrante ambos-deshabilitados.

**Tratar el frontmatter camel-case como un alias.** Rechazada porque el formato externo es el contrato kebab-case de los skills de Claude y el repositorio no tiene obligación de compatibilidad publicada. Fallar ruidosamente evita conservar silenciosamente una ortografía no estándar.

**Añadir un RPC de invocación directa de skills en el lado del navegador.** Rechazada para este cambio porque el flujo existente del navegador inserta una referencia del modelo en lugar de un cuerpo de instrucción cargado. Su política correcta es por tanto la intersección; una superficie de carga directa por el usuario necesita su propio diseño de cable y registro.

## Consecuencias

Los providers y los registros de runtime exponen un contrato de invocación tipado y pequeño, mientras que el YAML local sigue siendo extensible. Cada nuevo consumidor del descubrimiento debe elegir conscientemente el predicado de modelo, el predicado de usuario, su intersección o el acceso de confianza sin filtrar; olvidar esa elección ahora es visible en la revisión en lugar de quedar oculto en el comportamiento del registro.

El catálogo de modelos modificado queda fijado por la instantánea ACP sin clave, que incluye un skill solo de modelo y excluye un skill solo de usuario. La instantánea de TUI ensamblada sin clave descubre y carga un skill solo de usuario por nombre exacto y luego rechaza un skill solo de modelo antes de cargar su cuerpo; el smoke de Loader/PTY real prueba la misma ruta solo de usuario a través del proceso de terminal publicado. La instantánea de Chromium en host real fija la intersección del navegador en los cuatro cuadrantes de política. La cobertura unitaria de TUI ejercita esos cuadrantes más las carreras de disposición, mientras que las pruebas de registro, parser local, herramienta de modelo y proxy de API cubren los valores por defecto, las formas booleanas soportadas, los valores malformados, el rechazo de la clave heredada, la aplicación en la carga exacta y la intersección del navegador.
