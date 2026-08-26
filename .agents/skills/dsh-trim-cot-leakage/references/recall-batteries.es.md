Baterías de recall

Sondas para [la taxonomía](../SKILL.md#taxonomy), afinadas durante la purga de 2026-08. Cada acierto exige juicio semántico — las baterías pecan por exceso deliberadamente, y pecan por defecto por naturaleza: cada ronda de revisión de la purga encontró casos que ninguna batería detectó, así que combínalas con una lectura sin patrones de la prosa más densa del ámbito.

## Reglas de invocación

- Añade `--hidden --glob '!.git/**'` para que se busque `.agents/`; ripgrep se salta por defecto los directorios con punto y el mayor riesgo de omisión de la purga eran los Agent Notes.
- Las exclusiones van al final para que un include posterior no pueda readmitirlas: `--glob '!vendor/**' --glob '!node_modules/**' --glob '!.agents/notes/archived/**' --glob '!.agents/skills/dsh-trim-cot-leakage/**'` (los propios archivos de la skill citan redacciones filtradas como calibración), más los directorios de fixtures y snapshots grabados que estén en el ámbito. La [nota propietaria](../../../notes/implemented/process/2026-08-09-committed-artifact-citations.md) también se auto-detecta por su evidencia citada; júzgala como evidencia, no como uso.
- Las líneas de lenguaje natural llevan `-i` para que acierten las mayúsculas de inicio de oración ("This PR adds…", "Probably fine…"); la primera línea, que corresponde a patrones de código, se mantiene sensible a mayúsculas — con `-i`, `\\bT\\d\\b` y `\\bP-I\\b` se volverían ruido.
- Un patrón sin aciertos no prueba nada hasta que lo hayas visto acertar: pruébalo contra una cadena positiva conocida antes de confiar en el negativo.

## Batería de inglés

```sh
rg -n --hidden '\\(decision \\d|\\(audit [A-Z]\\d|design §|plan §|design ledger|\\(B ruling|\\bP-I\\b|\\bW\\d\\b|\\bT\\d\\b' ...
rg -n --hidden -i 'this PR|this branch|this stack|later PR|previous commit|this commit' ...
rg -n --hidden -i 'used to |no longer|previously|the old |was renamed|was moved' ...
rg -n --hidden -i '\\bv1\\b|this cut|\\bcut \\d|\\btoday\\b|\\bfor now\\b|roadmap' ...
rg -n --hidden -i 'rejected in review|review round|reviewer|as of v\\d' ...
rg -n --hidden -i 'probably |should be enough|should suffice|it simply|is safe —|is safe --' ...
rg -n --hidden '§\\d' ...
```

## Batería de chino

```sh
rg -n --hidden '设计稿|评审|上一?轮|旧版|老的|不再|以前|本版|遗留|私有' ...
rg -n --hidden '(^|[^a-zA-Z])端([^a-zA-Z]|$)' --glob '*.md' ...
```

## Familias de falsos positivos conocidos

Juzgadas y conservadas durante la purga; espera volver a encontrarlas:

- **«used to» instrumental** — "the key used to sign requests" es instrumental, no temporal. La forma temporal lleva antes un estado-sujeto ("colors used to come from…").
- **Viejo/nuevo en runtime** — "the old connection drains before the new one accepts" nombra objetos vivos durante un relevo, no estados del repositorio.
- **"This PR" en documentos de proceso** — la documentación *sobre* el flujo de PR ("the PR body should…", plantillas, las notas de proceso de este repo) dice legítimamente "PR"; lo vetado es que un documento adopte el punto de vista de un PR concreto sobre el código.
- **`v1` como segmento de protocolo o de ruta** — los endpoints `/v1/chat` y los nombres de formato del canal son identificadores, no sellos de versión.
- **`§N` con propietario comprometido** — las normas externas (RFC 9110 §10.1.5) y los documentos comprometidos que son dueños de su numeración con § siguen siendo citables por sección.
- **"actually" contrastivo y "wait" como sustantivo** — inglés corriente, no regulación de apuestas; ninguna línea comprometida los sondea, así que solo aparecen cuando amplías la batería con patrones de regulación más amplios.
- **"Today" en marcas de tiempo generadas y muestras de salida de CLI** — la salida grabada conserva su voz.
- **本版本 en prosa zh** — una renderización legítima de "this release" en contextos de artefacto versionado; lo vetado es 本版 como sello puro que refleja "this cut".
- **Secciones de alternativas consideradas** — un "rejected" dentro de la ranura de género de un Agent Note es su casa legítima, no coreografía de revisión.
