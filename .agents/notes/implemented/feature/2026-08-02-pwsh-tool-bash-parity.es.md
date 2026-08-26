# Agent Note: Paridad de la herramienta pwsh con bash

Status: implemented

[English](2026-08-02-pwsh-tool-bash-parity.md) | [中文](2026-08-02-pwsh-tool-bash-parity.zh.md) | Español

## Problema

La primera base nativa de Windows publicó `dsh-tool-pwsh` como un perfil deliberadamente mínimo — solo en primer plano (un proceso nuevo por llamada; sin sesión PTY persistente), sin paridad de entorno gestionado más allá de tres claves `DSH_*` fijadas en código, y una narrativa de marcadores («siempre `[exit code: N]`») que divergía del renderizado de la herramienta bash sin estar declarada. El contrato visible para el modelo se desvió de la implementación: la descripción prometía un reporte de rutas de spill que el renderizador nunca realizaba, el README afirmaba exportaciones que no existían y un renderizado que la herramienta no hacía, y las propias pruebas de la herramienta fijaban el comportamiento con pérdida. El perfil mínimo también dejaba el seam de contribución `DSH_*` duplicado por ausencia: los plugins que contribuyen hechos de entorno a `ctx.shellEnv` no tenían ningún efecto en las llamadas pwsh.

## Decisión

`dsh-tool-pwsh` ahora refleja a `dsh-tool-bash` llamada por llamada, y su texto visible para el modelo describe exactamente ese comportamiento:

- **El renderizado adopta la narrativa de bash al pie de la letra**: stdout, una sección marcada `[stderr]`, avisos de truncamiento con rutas de spill, `(no output)` para un cuerpo vacío y marcadores de salida solo para salidas distintas de cero — una salida limpia no produce ningún marcador. La descripción y la sección de prompt `tool:pwsh` lo declaran con precisión («las salidas distintas de cero se reportan como marcadores `[exit code: N]`»), sin copiar deliberadamente la frase «cada resultado» del prompt de bash, a la que su propio renderizador contradice.
- **`run_in_background` se conecta a través del runtime de trabajos genérico** exactamente como la herramienta bash: preflight, registro del propietario, control `job_output`/`job_kill` y el mismo mapeo de resultados. El handle `start()` ya reflejado de `pwsh-local` lo respalda.
- **El entorno `DSH_*` se comparte, no se duplica**: `ShellEnvRegistry` se movió de `dsh-tool-bash` a un nuevo paquete independiente de la herramienta, `@deepseek-ai/dsh-shell-env` (`ctx.shellEnv` + built-ins + el contribuyente de persistencia de sesión), y ambas herramientas de shell lo inyectan. Los contribuyentes se aplican a las llamadas pwsh exactamente como a las llamadas bash; la propiedad del entorno compartido reside por tanto fuera de cualquiera de las herramientas de shell visibles para el modelo.
- **La realidad de Windows queda fijada donde bash no tiene análogo**: cada comando se ejecuta bajo un preámbulo de salida UTF-8 para que el fallback de Windows PowerShell 5.1 no pueda corromper la salida no ASCII a través del colector de decodificación UTF-8, y los prompts enseñan que la terminación forzada en Windows se resuelve como salida 1 sin marcador de señal.
- **Fuera de alcance, sin cambios**: shells PTY persistentes (los backends son solo Linux/macOS; ConPTY es trabajo de hoja de ruta). La escalada de sandbox se publicó después con la [decisión de sandbox de ACL de Windows](2026-08-08-windows-acl-restricted-token-sandbox.es.md) — la herramienta pwsh ahora lleva el renderizado de denegación de sandbox y la misma superficie de escalada `sandbox_permissions` en el mismo turno, más el contrato de ConstrainedLanguage de Windows en su descripción. La tarjeta de terminal específica de pwsh con la píldora de salida se publicó por separado en la decisión de [la presentación UI de pwsh coincide con bash](2026-08-05-pwsh-ui-bash-parity.es.md).

## Alternativas consideradas

**Mantener el perfil mínimo y corregir solo las afirmaciones.** Rechazada: los contratos de texto copiados de bash se desvían sin la implementación correspondiente; una herramienta mínima con afirmaciones precisas sigue dejando las llamadas pwsh sin ejecución en segundo plano, sin paridad de contribuyentes y con una narrativa de marcadores divergente que habría que rejustificar para siempre.

**Rechazar en la carga un dialecto de ejecutor que no coincide.** Intentada y revertida antes del merge: un marcador `ShellDialect` (`bash` | `powershell`) en `ShellExecutor`, con ambas herramientas de shell lanzando una excepción cuando el ejecutor montado habla otra shell. Obligaba a toda implementación de ejecutor — incluidos cada test y cada fake de ejemplo — a declarar un dialecto, añadiendo ruido a cada test de herramienta de shell por una guarda sin despliegue plausible ni en el repo que atrapar (las composiciones publicadas emparejan siempre tool-pwsh con `dsh-pwsh-local` y tool-bash con `dsh-bash-local`). El contrato de emparejamiento queda documentado en el README de cada herramienta en su lugar.

**Extraer una base de implementación de herramienta totalmente compartida (dialecto de shell abstracto, dos hojas delgadas).** Considerada y diferida: la extracción de shell-env y el espejo estructural (los gemelos `render.ts`/`background.ts`) son la base sobre la que descansaría; una base completa espera hasta que un tercer dialecto o el gemelo PTY persistente hagan observable la forma de la abstracción.

## Consecuencias

- Las herramientas bash y pwsh son ahora intercambiables en comportamiento para el trabajo de shell en primer plano, en segundo plano y en sandbox (la superficie de sandbox llegó con la decisión de sandbox de ACL de Windows), y cada frase del prompt/descripción de pwsh está respaldada por el renderizador — la comprobación del revisor de grep contra el código pasa.
- La paridad se ejecutó en AMBAS direcciones una vez: el aborto estructurado en primer plano de la herramienta pwsh (`HarnessError('tool call aborted', TOOL_ABORTED)` con nombre `AbortError`) se backportó a la herramienta bash, reemplazando su `Error('command aborted')` sin codificar — un cambio visible para el modelo/registrado, fijado por pruebas de forma exacta en ambos lados y por el fixture cancel-tool-calls.
- `@deepseek-ai/dsh-shell-env` es un paquete nuevo publicado; la configuración `dshHome` de `dsh-tool-bash` se movió allí, así que las composiciones que montan las herramientas de shell deben montar también `shell-env` (los bundles del spine lo hacen).
- Las semánticas exclusivas de Windows (normalización CRLF, salida-1/signal-null de terminación forzada, self-signal solo POSIX) siguen fijadas por pruebas como antes.
- La compuerta de cobertura por archivo de la herramienta pwsh viaja en la suite de fake-executor scriptable (`tests/tools.spec.ts`); las suites de integración pwsh real y de composición Loader se auto-omiten donde `pwsh` está ausente, reflejando la división del trabajo de las suites de bash.
- La etapa de paridad de la propuesta de hoja de ruta está entregada; la etapa de presentación de la tarjeta de terminal se publicó en la decisión de [la presentación UI de pwsh coincide con bash](2026-08-05-pwsh-ui-bash-parity.es.md) (la TUI en sí se eliminó), dejando la composición por defecto de Windows como etapa restante.
