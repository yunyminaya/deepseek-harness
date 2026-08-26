# @deepseek-ai/dsh-client-ui-skill

[English](README.md) | Español

Fuente de invocación de skill, mitad de navegador: registra la fuente `skill` disparada por `/` en `ctx.inputTriggers`. Los candidatos de sesión ordinaria provienen del RPC `skill.list` abordado por el `{sessionId}` de la proyección `ClientSessionContext` por llamada, y el host resuelve `cwd` desde la cabecera de sesión. El host sirve toda skill invocable por el usuario; una entrada `modelInvocable: false` (una skill `disable-model-invocation`, cuyo único punto de entrada es esta vía) lleva el marcador solo-usuario como prefijo de descripción en el idioma activo. Los hijos continuables abordados por catálogo no resuelven candidatos de skill localmente porque el RPC de skill existente exige una sesión adjunta; ver su historial persistido no debe activarlos. Los catálogos se cachean por sesión ordinaria con una búsqueda single-flight; el hook `warm` del nacimiento del alcance precalienta la entrada de la sesión, el evento del propietario reenviado `agent-preset/selected` descarta la entrada de esa única sesión (el catálogo pertenece al preset, y una sesión en blanco puede cambiar después del calentamiento), y `connection/reset` lo limpia todo. Los resultados se filtran por `startsWith(query)`.

Una selección aterriza el texto literal `/name ` y el prompt envía ese mismo literal ([Agent Note del pipeline de slash](../../../.agents/notes/implemented/architecture/2026-07-25-web-input-machine-and-slash-pipeline.es.md)) — esta fuente no implementa hooks de adjudicación ni codec de referencia. El determinismo vive en el lado del host: el límite de gesto previo al paso (`dsh-tool-skill`) reconoce tokens `/name` delimitados por espacios en blanco que nombran skills invocables por el usuario en cualquier parte de un mensaje de usuario e inyecta el `<skill_content>` renderizado para cada punto de entrada, de modo que una selección de menú, un token tecleado a mano y un prompt de TUI/ACP cargan la skill de la misma manera. Un nombre compartido con un comando del host sigue resolviendo al comando: la adjudicación reclama la línea en el lado del cliente antes de que jamás se convierta en un prompt — precedencia deliberada, en línea con los productos homólogos. El RPC de lista viaja en la conexión de contexto raíz del plugin capturada en el registro — la fuente nunca lee servicios de un argumento por llamada; los visuales de chip de borrador derivan de la exploración del `lexicon`.

Un `skill.list` fallido lanza desde `candidates`, que el shell de slash registra y pliega en un descarte silencioso del grupo de menú — el menú solo muestra estados pendiente/listo.

Los exports de `/client` son solo el cuerpo del plugin (`apply`/`inject`); el objeto de la fuente es interno al efecto de registro.

## Fila de la herramienta de skill

El plugin de navegador también registra el nombre de wire `skill` en el slot con clave `tool.call.toolview` de `ui-tool`. Una fila colapsada renderiza el glifo de documento con destello de skill de 14 píxeles, el título `Skill`, el separador y el nombre de skill solicitado con la misma jerarquía neutra que la fila de Bash; las llamadas en curso llevan el shimmer del transcript, los fallos sustituyen el nombre por la primera línea de error y las llamadas interrumpidas usan el estado de advertencia. Una fila asentada se expande como un desplegable de fila completa en una tarjeta `Instructions` acotada que contiene la salida de herramienta duradera exacta, con la affordance estándar de trayectoria `Inspect` cuando está disponible. La fila deriva su nombre, su ciclo de vida y su cuerpo solo de la rebanada congelada de llamada/resultado que aporta `ui-tool`, nunca del catálogo actual, de modo que la reproducción permanece estable cuando cambian las skills instaladas o sus descripciones.

## Model Experience

### Invocación de skill explícita del usuario

#### Qué ve el modelo

El mensaje del usuario llega al modelo verbatim, con el literal `/name` incluido. El límite previo al paso del host (`dsh-tool-skill`) añade entonces el bloque canónico `<skill_content>` — la misma salida `renderSkillContent` que devuelve la herramienta `skill` — como contexto de instrucciones inyectado al final de las inyecciones de ese paso, lo más cerca de la respuesta del modelo. La carga es determinista: el modelo recibe el cuerpo completo sin que se le pida llamar a la herramienta `skill`, y el catálogo le dice que no vuelva a cargar una skill inyectada en línea.

#### Efecto de tokens

Una invocación añade el cuerpo de skill renderizado a ese turno como contexto inyectado — el mismo coste que si el modelo cargara la skill a través de la herramienta, pagado incondicionalmente en lugar de a discreción del modelo. Navegar por el menú y la búsqueda de candidatos añaden cero tokens de modelo.

#### Efecto de KV Cache

Solo anexado: el mensaje inyectado aterriza después del prefijo de historial reutilizable. Este paquete nunca edita tokens de petición anteriores.

## Limitaciones conocidas y trabajo pendiente

- **Las páginas de historial solo de resultados usan la fila genérica** — el despacho con clave necesita la llamada emparejada en la ventana del runtime; la paginación que deja la llamada fuera no tiene identidad de herramienta. Esta funcionalidad de presentación del cliente no extiende el contrato de wire del historial para recuperarla.
- **El texto es la verdad** — la referencia es texto de borrador plano; un token idéntico tecleado a mano es la misma referencia, y el límite de gesto del host juzga el texto enviado, no la interacción con el menú. Los visuales de chip derivan de la exploración del léxico; no hay identidad de aparición, seguimiento de posición ni payload de referencia estructurado en el wire del prompt (ambos son elementos del ledger).
- **Un menú abierto antes de que el precalentamiento se asiente** no muestra candidatos de skill para esa pulsación; la siguiente pulsación vuelve a consultar la caché asentada.
