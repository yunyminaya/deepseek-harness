---
name: record-browser-gif
description: Graba demos de interacción del browser o de la Web UI como GIFs optimizados usando el browser integrado disponible, captura de frames basada en estado y codificación determinista, y luego publícalos en una rama dedicada de assets cuando la tarea incluya adjuntar el GIF a un pull request. Úsalo cuando te pidan crear, grabar o generar un GIF que demuestre un flujo del browser, y para todo pull request que cambie comportamiento de GUI visible para usuarios del producto, que MUST incluir un GIF grabado desde el servidor real del pull request y el flujo real del modelo.
---

# Grabar un GIF del browser

[English](SKILL.md) | Español

Produce una demostración breve y veraz de la UI como un GIF local y — solo cuando la tarea incluya adjuntarlo a un pull request — publícalo mediante el flujo de rama de assets al final de este skill.
Usa el skill de browser-control para la interacción y el encoder incluido para obtener timing, dimensiones y tamaño repetibles.

La [decisión de cadena de evidencia](../../notes/implemented/process/2026-08-08-browser-gif-evidence-chain.es.md) posee por qué un storyboard sale de una sola ejecución aislada y por qué la publicación revalida tanto el artefacto como la cabeza del pull request demostrada.

## Todo pull request de GUI incluye un GIF

Un pull request que cambie comportamiento de GUI visible para usuarios del producto MUST incluir un GIF de demostración grabado con este skill e incrustado en el cuerpo del pull request mediante [el flujo de rama de assets](#publish-to-an-assets-branch).

La propia grabación forma parte de la evidencia: usa un servidor real arrancado desde el árbol de esa rama del pull request, una API key real y rondas reales del modelo.
Nunca sustituyas con consultas de fixtures, transportes mock, inyección sintética de eventos ni hooks solo de test salvo que el usuario haya pedido explícitamente una grabación con fixtures.
Junto al embed, indica el commit SHA exacto demostrado, el árbol y el origin que lo sirvieron, cualquier mode flag o excepción de estado del browser y si se ejecutó una ronda real de modelo, para que los reviewers sepan exactamente qué demuestra la grabación.

## Mantén separadas la grabación y la publicación

- La grabación produce imágenes de frames y un único artefacto local `.gif`; nunca muta estado remoto.
- La publicación — hacer push del GIF a una rama de assets e incrustarlo en el cuerpo de un pull request — es el paso final separado, realizado solo cuando la tarea incluya adjuntar el GIF a un pull request.
Nunca toca la rama del propio pull request.
- Conserva las condiciones de grabación solicitadas.
Una demo con servidor real o API real no debe usar consultas de fixtures, transportes mock, inyección sintética de eventos ni hooks solo de test.
Si no hay credenciales o el servidor no está disponible, informa esa limitación en lugar de sustituir por un fixture.
- Nunca leas ni expongas valores de credenciales.
Usa la ruta normal de configuración de la aplicación y un prompt de demostración inocuo.

## Prepara la aplicación

Un GIF para un pull request concreto demuestra el árbol de ese pull request, así que la preparación es por pull request:

1. Exige un worktree limpio, registra su commit exacto con `git rev-parse HEAD` y luego construye ese árbol registrado — aquí, `pnpm run build && pnpm run build:web`.
Un GIF grabado contra el build de otro commit atribuye mal la evidencia.
2. Arranca un servidor por puerto desde ese árbol con `DSH_HOME`, `DSH_AGENTS_HOME`, workspace y estado de sesión scratch y frescos.
Da al browser también un contexto o perfil aislado y fresco; si el flujo de browser no puede crearlo, borra las cookies y el site storage de ese origin antes de navegar para que el estado persistido del cliente no afecte la evidencia.
Carga la `.env` raíz para la API key a través de la ruta normal de la aplicación; nunca imprimas la key.
3. Trata un storyboard como una ejecución de evidencia: todo frame publicado proviene de ese servidor y de esos state roots, workspace, sesión y escenario respaldado por modelo.
Si la automatización de captura falla, descarta sus frames y vuelve a ejecutar desde roots frescos; nunca mezcles frames de ejecuciones separadas.
4. Al cambiar entre pull requests, detén el servidor viejo por PID o con una coincidencia exacta de su línea de comandos.
Un patrón amplio con `pkill -f` puede coincidir y matar la shell que lo lanzó — incluida la tuya.

## Graba el flujo

1. Invoca el skill de browser-control disponible y sigue sus instrucciones de preparación, interacción y limpieza.
Usa el estado de Chrome existente del usuario solo cuando se solicite o sea necesario; declara esa excepción en la procedencia y no afirmes que hubo estado fresco del cliente.
Si browser control no está disponible, usa la dependencia Playwright declarada por el repositorio en un browser headless aislado; no instales otro driver ni lances el browser del usuario.
Declara ese fallback en la procedencia.
2. Antes de grabar, identifica el origin exacto, si la app está construida o en desarrollo, el transporte y cualquier modo fixture o mock.
Registra solo afirmaciones que la preparación observada respalde.
3. Cuando un default de producción abra una superficie nativa del sistema operativo que la automatización headless no pueda conducir, selecciona un backend oficial operable desde el browser a través de la configuración normal de la aplicación.
Declara el override en la procedencia; un fixture, mock transport o hook solo de test no es un sustituto aceptable.
4. Elige de tres a seis estados que cuenten una sola historia, por ejemplo typed, running, settled y detail.
Prefiere cambios de estado semánticos frente a captura continua; omite churn de carga que no ayude al espectador.
5. Mantén un único viewport y crop para todos los frames, y nómbralos léxicamente: `00-initial.png`, `01-typed.png`, etc.
6. Guarda los frames bajo el directorio gitignored `.playwright-mcp/` del repositorio — las capturas del browser tool solo pueden escribirse bajo sus roots permitidos, y los nombres relativos resuelven contra la raíz del repositorio.
Crea primero el subdirectorio de frames (`mkdir -p .playwright-mcp/gif-frames-<label>`); escribir en un directorio ausente falla con ENOENT al capturar.
7. Antes de cada screenshot, espera una condición concreta de UI como una etiqueta única, un control habilitado, un título de documento cambiado o una respuesta completada.
Exige que el locator resuelva exactamente un elemento; para locators por accessible-name de Playwright, usa `exact: true` cuando se pretenda igualdad, porque el texto descendiente o un eco del prompt pueden crear una coincidencia falsa.
No uses un delay fijo como prueba de que la aplicación llegó al estado.
8. Haz que los predicados de finalización coincidan con un elemento de texto exacto — por ejemplo, un elemento cuyo texto recortado sea exactamente la respuesta esperada — nunca con una comprobación por subcadena como `body.textContent.includes(...)`, que también satisface el eco del propio prompt del usuario.
9. Cuando la afirmación involucre una llamada de herramienta, rechazo o recuperación, incluye un frame de detalle o trayectoria que muestre la identidad de la herramienta, el estado o código de error estable y el resultado posterior.
Un resultado solo en el chat no demuestra por qué se comportó así la ruta de la herramienta.
10. Captura un estado transitorio (spinner, fila running) conduciendo una operación lenta en foreground — por ejemplo, un comando bash `sleep 15` — y haciendo polling de un marcador DOM concreto (un atributo `data-*`) dentro de una sola llamada del browser script que también tome el screenshot.
El estado consultado a través de llamadas de herramienta separadas se pierde, porque el turno se liquida entre llamadas.
11. Diseña el prompt para que el estado que necesitas ocurra de verdad: instruye al modelo a esperar en foreground cuando de otro modo enviaría a background un comando lento, y dale un sentinel de settle como “reply with the single word done” para anclar el predicado de finalización.
12. No captures secretos, datos personales, pestañas no relacionadas ni notificaciones transitorias.
Detén cualquier ejecución real de API innecesariamente larga cuando el estado demostrado ya sea visible.

Usa la API de screenshot del propio browser.
Cuando devuelva bytes de imagen, guarda esos bytes directamente; el encoder detecta contenido de imagen independientemente de la extensión del nombre de archivo.

## Codifica el GIF

Exige `python3`, `ffmpeg` y `ffprobe`.
Si falta alguno de los binarios multimedia, informa la dependencia en vez de instalar software sin autorización.

Exporta `GIF_SKILL_DIR` como la ruta absoluta de este skill en su propia línea antes del comando python — una asignación inline `GIF_SKILL_DIR=... python3 "$GIF_SKILL_DIR/..."` falla, porque el argumento se expande antes de que la asignación tenga efecto:

```sh
export GIF_SKILL_DIR=/absolute/path/to/this/skill
python3 "$GIF_SKILL_DIR/scripts/encode_gif.py" \
  /absolute/path/to/frames \
  /absolute/path/to/demo.gif \
  --durations 1.5,1.5,1.5,3.5 \
  --fps 10 \
  --max-width 1200 \
  --colors 128
```

Una única duración se aplica a todos los frames; en caso contrario, proporciona una duración positiva por frame separada por comas, manteniendo el último estado settled durante más tiempo.
El encoder rechaza menos de dos frames, dimensiones o duraciones que no coincidan, límites no válidos, sobrescrituras accidentales, duraciones inesperadas y salidas por encima de `--max-bytes`.

Para un artefacto grande, reduce primero `--max-width`, luego `--colors` o `--fps`; conserva texto legible y el estado final el tiempo suficiente para inspeccionarlo.
Usa `--force` solo después de resolver la ruta exacta de salida.

## Verifica el artefacto

1. Lee el resumen JSON del encoder y confirma la ruta de salida, los recuentos de frames fuente y codificados, las dimensiones, la duración y el tamaño en bytes.
2. Lee visualmente el GIF codificado en sí, no solo los frames fuente.
Confirma que la transición sea legible, que el último estado se mantenga el tiempo suficiente y que no aparezca contenido sensible.
Si el visor renderiza solo el primer frame, decodifica frames representativos del GIF codificado con `ffmpeg` e inspecciónalos; las capturas previas a la codificación no prueban el orden, la paleta ni la duración final del GIF codificado.
3. Ejecuta `git status --short` y confirma que los frames y el artefacto hayan caído solo bajo rutas ignoradas.
4. Devuelve la ruta absoluta del GIF, muéstralo cuando el cliente soporte media local e indica si la grabación usó API real, fixture u otro transporte.
Cuando la tarea no incluya adjuntar el GIF a un pull request, detente aquí.

## Publica en una rama de assets

Realiza este paso solo cuando la tarea incluya adjuntar el GIF a un pull request.

Nunca hagas commit de un GIF en la rama del propio pull request ni en ninguna rama que se fusione en una rama de larga vida: el media binario allí infla el historial del repositorio para todo clon futuro.
Los GIF viven en una rama huérfana dedicada de assets — una rama sin commit padre y sin nada salvo media — y una rama de assets sirve a toda una serie de pull requests (llamada `<series>-assets`; lista las existentes con `git ls-remote --heads origin '*assets*'`).

Antes de que cualquiera de los flujos siguientes haga push, verifica que la rama de assets contenga solo media y que el checksum del GIF staged coincida con el del artefacto local verificado.

Para una rama de assets existente, trabaja en un scratch clone shallow de una sola rama para que la publicación no pueda tocar tu working tree:

```sh
git clone --branch <assets-branch> --single-branch --depth 1 <repo-url> /tmp/assets-checkout
cp /absolute/path/to/demo.gif /tmp/assets-checkout/<name>.gif
cd /tmp/assets-checkout
git add <name>.gif
git commit -m "assets: <what it shows> gif (#<pr>)"
git push origin <assets-branch>
```

Para una serie nueva, crea un scratch clone shallow fresco (`git clone --depth 1 <repo-url> /tmp/assets-checkout`), crea la rama huérfana con `git switch --orphan <assets-branch>` y luego añade el GIF, haz commit y push del mismo modo.

Después del push, usa la API autenticada de GitHub o peticiones raw para confirmar la ruta remota, el tamaño en bytes, el checksum, la respuesta `200` y el content type `image/gif`.
Un `404` anónimo no refuta un asset de repositorio privado; autentica también la verificación.
Esto demuestra la ruta de review para miembros del repositorio, no la disponibilidad pública.

Inmediatamente antes de editar el cuerpo del pull request, relee su head vivo y compáralo con el commit registrado junto al GIF.
Detente y vuelve a grabar si se movió.
Después de editar, vuelve a leer el head vivo y exige que siga en ese commit registrado.
Por separado, renderiza el cuerpo a través de la API Markdown de GitHub y confirma que el `<img>` esperado esté presente.

Incrusta el GIF en el cuerpo del pull request con la raw blob URL; el sufijo `?raw=true` es obligatorio, porque la blob URL simple renderiza la página de archivo de GitHub en vez de la imagen:

```markdown
![<alt text>](https://github.com/<owner>/<repo>/blob/<assets-branch>/<name>.gif?raw=true)
```

Nunca borres ni reescribas una rama de assets, y nunca le hagas force-push: los cuerpos de pull requests fusionados referencian sus URLs para siempre.
Añade solo commits nuevos.
