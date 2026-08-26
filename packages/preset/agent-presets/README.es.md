# dsh-agent-presets

[English](README.md) | Español

Composición de agent por preset. Un **preset** es un directorio que contiene un `agent.cordis.yml`; el roster lo monta UNA vez por proceso bajo un ámbito permanente, y cada sesión que lo nombra se une haciendo que su clave de ámbito de agent se convierta en hija del montaje (la cadena de padres de `dsh-scope`). Las herramientas, las secciones de prompt y las unidades de proyección del montaje existen exactamente una vez y cubren a todos los agents unidos — sus plugins fijan su estado por Session/Agent, así que las sesiones se mantienen separadas dentro de una única instancia compartida — y un lector host sin ningún agent (una lectura de transcript en frío) resuelve los mismos registros permanentes por id de preset.

El mecanismo son dos seams. Los contextos de entrada se encadenan al contexto en el que se enchufó un subárbol, y tanto [`dsh-tools`](../../core/tools/README.es.md) como [`dsh-system-prompt`](../../core/system-prompt/README.es.md) archivan los registros en la capa de ámbito del contexto llamador — así que las contribuciones del montaje permanente aterrizan en la capa del PRESET. Lo que las lleva a cada sesión es la cadena de padres de `dsh-scope`: las vistas de un agent resuelven `agent → preset → global` (lo más cercano sombrea lo más lejano), y los listeners del montaje se admiten para cada agent bajo él mientras los de un preset hermano permanecen sordos.

## Servicio: `AgentPresets` (clave de ctx: `agentPresets`)

El descubrimiento no está memoizado: `list()` y `resolve()` releen las raíces en cada llamada, así que un preset creado mientras el proceso se ejecuta es visible de inmediato y uno eliminado desaparece de la siguiente lectura. El descubrimiento también es dueño de la **salud** de los presets: un directorio cuya composición falta o no se puede cargar (YAML no analizable — comprobado con el dialecto propio del loader, `!!js` incluido — o que no es una lista de filas de plugin con nombre) se lista con una razón `broken` en lugar de omitirse, porque un directorio omitido seguiría ocupando su id en disco mientras cada superficie no muestra nada que eliminar. Un directorio cuyo nombre no es un id de preset utilizable (`[a-z0-9][a-z0-9-]*`) se omite directamente: ninguna copia podría reclamarlo nunca.

- `ctx.agentPresets.defaultId: string` — El id de preset montado cuando un llamador no nombra ninguno.
- `ctx.agentPresets.list(): Promise<AgentPreset[]>` — Todos los presets que suministran actualmente las raíces configuradas, ganando la raíz anterior ante un id duplicado; los presets rotos incluidos, cada uno con su razón.
- `ctx.agentPresets.resolve(id?): Promise<AgentPreset>` — Un preset por id, con `defaultId` como predeterminado. Lanza nombrando los ids disponibles cuando ninguna raíz lo suministra. Un preset roto se resuelve — eliminar, leer e informar uno necesitan todos la fila.
- `ctx.agentPresets.mount(agentCtx, id?): Promise<AgentPreset>` — Componer un agent desde un preset: garantizar su montaje permanente (single-flight) y convertir la clave de ámbito del agent en hija de él — devolviendo el preset para que el llamador lo registre. Rechaza un preset roto de antemano con su razón informada por el descubrimiento, de modo que toda forma no cargable falle igual antes de implicar al loader.
- `ctx.agentPresets.composeFrom(agentCtx, parentCtx): string | undefined` — Unir un agent a la composición permanente en la que otro ya se ejecuta, devolviendo el id de preset unido — `undefined` cuando el padre no se unió a ninguna, que es el despliegue sin roster y no un error. Es un bind más que un montaje, así que es síncrono y no tiene modo de fallo de composición; sigue rechazando un error del llamador (un contexto sin ámbito, o un agent que ya se unió).
- `ctx.agentPresets.composedPreset(agentCtx): string | undefined` — El preset en el que se ejecuta UN agent vivo, leído de su cadena de ámbito en lugar de su sesión — la única respuesta disponible para un agent cuya cabecera durable aún se está construyendo.
- `ctx.agentPresets.recompose(agentCtx, id): Promise<AgentPreset>` — Re-vincular un agent a la composición permanente de otro preset. Solo es válido mientras el agent no haya producido nada — **el llamador es dueño de esa comprobación**; el nuevo montaje se garantiza antes de que el vínculo se mueva, así que un fallo deja al agent como estaba. Rechaza un preset roto igual que `mount()`.
- `ctx.agentPresets.standingKeyFor(id?): Promise<ScopeKey>` — La clave de ámbito permanente en la que un lector host sin agent (una lectura de transcript en frío) resuelve los registros del preset; garantiza el montaje sin iniciar un agent, sesión ni turno. Rechaza un preset roto igual que `mount()`.
- `ctx.agentPresets.roots: readonly PresetRoot[]` — Las raíces que escanea este roster — cada raíz configurada en orden y después la raíz derivada del home del harness. No es `config.roots`: lee esto para responder si un roster está compuesto en absoluto, de modo que una sola derivación lo decide.
- `ctx.agentPresets.authorable: boolean` — Si alguna de esas raíces tiene confianza `user` y, por tanto, si se puede crear un preset en absoluto.
- `ctx.agentPresets.read(id): Promise<string>` — El texto de composición de un preset, exactamente como está almacenado.
- `ctx.agentPresets.copy(from, id, name?): Promise<void>` — Crear un preset de autoría local copiando el directorio completo de uno existente — la única escritura de autoría. Ningún texto de composición cruza este seam, así que una copia es exactamente tan cargable como su fuente; los metadatos copiados conservan la descripción de la fuente pero nunca su nombre ni su orden en el roster, y `name` (o el id como alternativa) es lo que distingue las filas.
- `ctx.agentPresets.remove(id): Promise<void>` — Eliminar un preset de autoría local; las sesiones unidas conservan su montaje permanente. Limpia el predeterminado de usuario cuando nombraba al preset recién eliminado: almacenar un predeterminado que aún no existe es deliberado, pero uno que esta llamada eliminó nunca volverá a suministrarse y fallaría cada sesión creada sin una elección explícita.

`AgentPreset` lleva `id` (el nombre del directorio), `trust` (`system` o `user`, de la raíz bajo la que se encontró), `path` (el archivo de composición absoluto) y — solo cuando el preset no puede componer una sesión — `broken` (una razón legible por humanos, mostrada verbatim en las superficies del roster).

### Dónde llamar a `mount()`

El hook `setup(agentCtx)` de la fábrica de agents es el único punto de llamada admitido. Solo allí se instala la unión mientras el agent aún no está publicado, de modo que una composición rechazada revierte toda la creación en lugar de dejar una sesión a medio componer. El subárbol permanente es propiedad de la propia fiber del servicio del roster — deliberadamente su contexto UNTRACED, porque un subárbol acuñado desde un `this.ctx` trazado resolvería cada servicio a través de la fiber sombra del llamador en lugar del propio almacén de inyección de cada entrada — así que sobrevive a cada agent y solo se desenrolla con todo el árbol. Cada generación registra la stamp de su archivo de composición (mtime y tamaño): una sesión que encuentra la stamp obsoleta inicia la siguiente generación, mientras que cada sesión ya unida conserva la que ejecuta — la composición a la que se unió una sesión en ejecución sobrevive a que su archivo cambie o desaparezca bajo ella, y los archivos son el único editor de composición, así que la stamp es lo que lleva una edición a las sesiones posteriores.

### Componer un agent hijo

El hijo de un subagente se une a la composición permanente de su padre mediante `composeFrom()`, nunca mediante `mount()`. Toda fila orientada al modelo vive en el plano del agent, así que la capa global del registro de herramientas está vacía y un hijo que no se une a nada llega al modelo sin herramientas en absoluto y sin ninguna de las secciones de prompt de su padre.

Re-montar el preset del padre por id diferiría del bind en dos aspectos que ambos importan. Un archivo de composición editado desde que el padre arrancó entregaría al hijo una generación DIFERENTE de aquella bajo la que se produjo el historial de su padre, y un preset eliminado desde entonces haría fallar al hijo de plano mientras su padre sigue ejecutándose. El bind además es síncrono, que es lo que permite a los drivers de subagente en proceso usarlo en absoluto: componen a sus hijos dentro de una ventana de creación síncrona.

El hijo registra el id unido en su propia cabecera durable ([`dsh-subagent`](../../subagent/subagent/README.es.md)), de modo que una lectura en frío del historial del hijo reconstruye la composición bajo la que realmente se ejecutó en lugar del predeterminado del despliegue.

### Qué preset ejecuta una sesión

La cabecera de creación nombra el preset con el que una sesión COMENZÓ; `resolveSessionPreset(session)` nombra el que EJECUTA. Difieren siempre que una sesión en blanco cambió, así que toda ruta de reconstrucción — el resumen que lee un selector, una reanudación, una bifurcación — resuelve en lugar de leer la cabecera.

La cabecera permanece congelada porque es un hecho de creación. Un cambio es un evento de sesión `agent-preset/selected` añadido después de que el intercambio se confirme, que es lo que exige la regla visible-para-el-modelo ⟺ registrado: el preset decide los schemas de herramienta y las secciones de prompt que ve el modelo, así que tiene que poder reconstruirse desde el log. El servicio re-emite ese hecho confirmado como el evento cordis sin ámbito `agent-preset/selected(sessionId, agentPreset)` declarado por el export `./types` seguro para clientes, permitiendo que los consumidores remotos invaliden el estado derivado de la sesión sin importar tipos de runtime del Host. Leer solo la cabecera reconstruiría una sesión cambiada bajo la composición con la que se creó, repitiendo un historial sobre el que el nuevo conjunto de herramientas no puede actuar: el peligro exacto que el bloqueo de solo sesiones en blanco existe para prevenir.

### Cambiar un agent en blanco

`recompose()` desmonta el subárbol instalado y monta el nuevo, porque dos composiciones no pueden coexistir: ambas registrarían los mismos nombres de herramienta en una sola capa. Un montaje fallido restaura la composición anterior en lugar de dejar al agent sin nada, y un id desconocido se rechaza antes de desmontar nada.

La restricción a un agent que no ha producido nada es una regla de producto, no mecánica: intercambiar herramientas a mitad de conversación dejaría llamadas de herramienta registradas que la nueva composición no puede hacer. El gateway la hace cumplir en el wire ([`dsh-apiproxy`](../../host/apiproxy/README.es.md) responde `agent-preset-locked`), que es donde el historial de la sesión está a mano.

## Autoría

La autoría es solo copia. Un preset nuevo es una copia de directorio completo de uno existente — composición, metadatos, directorios de skill, assets — aterrizada bajo la primera raíz `user`; las entradas son dos ids que el servicio resuelve contra sus propias raíces más un nombre visible opcional, así que ningún llamador suministra jamás texto de composición y una copia no concede nada que el roster no llevara ya. Todo lo posterior a la creación ocurre en los propios archivos del preset. `copy()` rechaza tres cosas antes de que aterrice nada:

- **Un id que no es `[a-z0-9][a-z0-9-]*`.** El id se convierte en un nombre de directorio, así que la contención es una propiedad del propio id más que de una comprobación de ruta posterior: `../escape`, `a/b` y una ruta absoluta se rechazan todos como ids.
- **Un id que ya está ocupado.** Una copia nunca sobrescribe: cualquier raíz que suministre el id lo rechaza (un directorio de usuario nombrado como un preset incluido quedaría sombreado por él), y un directorio que ocupe el nombre en disco también lo rechaza. El descubrimiento lista ese directorio como un preset roto, de modo que la salida del rechazo — eliminarlo — está en la misma página que lo informó.
- **Una fuente desconocida.** La fuente puede tener cualquier confianza — copiar un preset incluido es el caso principal — pero debe existir; una copia fallida revierte su directorio a medio hacer en lugar de dejar uno que el descubrimiento no puede ver.

El árbol copiado se re-aprieta a solo-propietario (los archivos `0o600` conservan su bit de ejecución del propietario, los directorios `0o700`), los symlinks se desreferencian para que la copia sea autocontenida, y la raíz se crea en la primera copia — un despliegue que configura una raíz de usuario que aún no existe es el estado normal de primer arranque. El `preset.yml` copiado se reescribe: la descripción de la fuente se conserva para que el autor la edite en su sitio, pero su nombre y el `order` del roster se descartan — una copia que se presentara idéntica a su fuente, u ordenada dentro del orden declarado del conjunto incluido, haría que el roster dejara de distinguirlas. `remove()` rechaza un preset que se entrega con el despliegue; el conjunto incluido son las composiciones comprobadas (known-good) de las que parten las copias.

### Cómo se resuelven las filas de un preset

El **nombre de paquete** de una fila se resuelve desde la composición del host, no desde el directorio del preset. El Loader resuelve normalmente una entrada contra el `baseUrl` de su propio árbol, que para un preset es dondequiera que esté el archivo de composición; un preset de autoría local vive bajo el home del usuario, donde el ascenso de `node_modules` de Node nunca alcanza el harness, así que toda fila `@deepseek-ai/dsh-*` fallaría al importar. El montaje registra la base del host antes de enchufar el subárbol y envía allí los specifiers desnudos.

Una ruta **relativa** sigue resolviéndose desde el propio directorio del preset, así que los archivos de plugin y los directorios de skill del preset viajan con él.

Una ruta de sistema de archivos **absoluta** conserva su propia ubicación. El montaje la convierte en una URL `file:` antes del import ESM para que las rutas POSIX y las rutas de Windows con letra de unidad o UNC usen un specifier que Node acepta.

### Metadatos de visualización

Un preset puede publicar texto visible en un `preset.yml` opcional junto a su composición:

```yaml
name: 极简模式
description: 仅提供持久 bash 与 str_replace_editor 的双工具编码 Agent。
```

Lleva SOLO texto visible. `id` es el nombre del directorio y `trust` viene de la raíz bajo la que se descubrió el preset, así que ninguno es escribible aquí — de lo contrario, un preset de autoría local podría nombrarse a sí mismo dentro del conjunto incluido. Es un archivo separado porque la composición es una lista de nivel superior de filas de plugin: YAML no puede llevar claves hermanas junto a ella, y una fila de metadatos falsa entregaría al Loader algo que cargar.

Todo fallo de lectura degrada a ausencia de metadatos: ausentes, malformados, con tipo incorrecto o en blanco significan todos lo mismo, y un selector recurre al id. La presentación no es capacidad: un preset con un nombre roto sigue montándose.

## Config

| Campo | Predeterminado | Significado |
|---|---|---|
| `default` | obligatorio | Id de preset montado cuando un llamador no nombra ninguno |
| `roots` | `[]` | Directorios escaneados en orden de precedencia; cada uno suministra `path` (un `~` inicial se expande) y `trust` (predeterminado `user`) |
| `includeUserRoot` | `true` | Añadir `<dshHome>/.agent-presets` como raíz `user`, después de cada raíz configurada |

Una raíz ausente no suministra presets en lugar de fallar: la raíz de usuario no existe hasta el primer preset de autoría local, y nombrar un predeterminado que ninguna raíz suministra ya falla de forma ruidosa en la resolución.

### La raíz escribible es de este paquete; la raíz incluida, de la app

`<dshHome>/.agent-presets` es donde viven los presets propios de una persona, igual que `<dshHome>/skills` es donde viven sus propias skills ([`dsh-skill-filesystem`](../../skill/skill-filesystem/README.es.md)), así que el roster la deriva en lugar de esperar a que un despliegue la recuerde — un lanzador que no configura nada encuentra y crea presets igualmente. Se añade DESPUÉS de cada raíz configurada, lo que mantiene que una raíz anterior gane ante un id duplicado: un `standard` incluido sigue sombreando a un directorio del home que reclamó el nombre, y `copy()` rechaza ese id en lugar de aterrizar un preset que nada resolvería.

Las raíces se resuelven una vez, cuando el servicio se construye. Un conjunto de raíces que cambiara entre un `list()` y el `copy()` que actúa sobre su respuesta crearía en un directorio que el llamador nunca vio.

`includeUserRoot: false` monta un roster solo sobre `roots`. Un despliegue que confina los presets a sus propios directorios lo necesita, y también cualquier prueba que fije un roster exacto — de lo contrario, el `<dshHome>` real de la máquina decide lo que contiene el roster.

La raíz INCLUIDA sigue siendo un hecho de ensamblaje: está junto a la configuración propia de la app instalada, una ruta que solo esa app puede resolver.

### El preset predeterminado es un ajuste de usuario

Cuando se compone un provider de ajustes, este plugin registra el namespace `agent-presets` con `config.default` como su base de composición, de modo que el documento del usuario se superpone al predeterminado de ingeniería del despliegue:

```yaml
agent-presets:
  default: minimal
```

El valor se lee por resolución en lugar de tomarse como instantánea, así que un documento recargado en caliente surte efecto en la siguiente sesión creada y toda sesión en ejecución permanece en el preset desde el que se compuso. Limpiar el campo de usuario re-hereda el predeterminado de la composición. Un predeterminado que nombra un preset que ninguna raíz suministra se almacena sin queja y falla en el siguiente `resolve()` — el roster es un directorio vivo, así que un nombre ausente ahora puede existir cuando una sesión lo pida.

## Qué rechaza un montaje

Un subárbol enchufado directamente está ausente de `ctx.loader.entries()`, así que ninguna auditoría de arranque lo cubre. `mount()` por tanto prueba él mismo que el resultado es utilizable y rechaza tres cosas.

**Un objetivo sin ámbito.** Montar en un contexto que no lleva ámbito de agent registraría las herramientas del preset globalmente, para cada agent del proceso.

**Una fila que nunca se volvió utilizable.** El loader ya rechaza una fila cuyo módulo falló al importar o cuyo plugin lanzó; lo que queda es una fila que aún espera un servicio que la composición nunca suministra, y la auditoría lo nombra.

**Una fila que publicó un servicio en el realm raíz.** Tal servicio es global del proceso, así que el segundo preset que publica el mismo nombre colisiona con el primero, y un lector host resolvería la instancia de un preset para cada sesión. Un preset que posee de verdad un servicio lo pone detrás de un realm `isolate` — los realms locales de entrada mantienen separados los servicios homónimos de dos presets exactamente como una vez mantuvieron separados los de dos sesiones — o el servicio pertenece en su lugar a la composición del host.

El invariante del paquete re-comprueba esa última regla en cada notificación de servicio, porque una fila que publica desde un timer o una continuación asíncrona escaparía de la auditoría de una sola vez.

## Un archivo de preset es una entrada, nunca un objetivo de persistencia

El Loader escribe un árbol de vuelta a su archivo fuente siempre que decide que la configuración cambió, y una fila que dispone su propia fiber basta para decidirlo: la entrada se marca `disabled` y el árbol se escribe. Heredado, eso quemaría el estado de runtime de una sesión en un archivo que todas las sesiones comparten — comentarios eliminados por el viaje de ida y vuelta de YAML, y un rechazo de `writeFile` dentro de un `setTimeout` para un preset incluido de solo lectura.

El subárbol montado por tanto anula `write()` como no-op. Nada en este paquete escribe una composición; crearla es una operación separada y explícita.

## Confianza

Los presets son composiciones, así que un preset es exactamente tan privilegiado como los plugins que nombra. Un preset `user` — creado por una persona o por un agent — lleva la misma confianza que el acceso a shell; el campo `trust` existe para que los consumidores puedan presentar esa diferencia, no para imponerla.

## Experiencia del modelo

Indirectamente, a través de los plugins que registra una composición permanente, que poseen cada schema de herramienta y sección de prompt que el preset hace visibles a los agents unidos a ella.

#### Efecto de KV Cache

Estable en el prefijo durante la vida de un agent: una composición se instala una vez, antes de que el agent se publique y por tanto antes de su primera solicitud, y nunca se vuelve a leer mientras el agent se ejecuta. Elegir un preset distinto para una sesión nueva establece un prefijo distinto solo para esa sesión y no puede invalidar la reutilización de ninguna sesión ya en ejecución.

## Limitaciones conocidas y trabajo diferido

- **Un preset fuera de la raíz escribible es descubrible pero no eliminable** — `remove()` rechaza todo lo que no viva bajo la PRIMERA raíz `user`, así que un despliegue que configura su propia raíz escribible dejando `includeUserRoot` activado lista los presets del home del harness, los monta y luego responde «no vive bajo la raíz escribible de presets» en cada eliminación. El roster lleva una sola raíz escribible por diseño; un despliegue que quiere solo la suya pone `includeUserRoot: false`.
- **Un preset no puede cambiarse una vez que una sesión ha producido algo** — `recompose` re-vincula el ámbito padre de una sesión EN BLANCO a otro montaje permanente, y solo una en blanco: cambiar una composición que ya se ejecutó dejaría varadas herramientas que el modelo ha llamado. Cambiar el predeterminado afecta solo a las sesiones creadas después.
- **Una generación se fija solo en el archivo de composición** — la comprobación de stamp nota el cambio de `agent.cordis.yml`, no una edición a un archivo de skill o asset a su lado; esos llegan a sesiones nuevas solo cuando el propio archivo de composición se mueve o el proceso se reinicia.
- **Una generación superada nunca se reclama** — las sesiones ya unidas conservan la generación en la que se ejecutan, y el roster no guarda ningún recuento de uniones que pudiera decir cuándo se fue la última, así que todo el subárbol permanece montado hasta que el proceso termina. El coste es por generación más que por sesión, pero no es gratis: `dsh-skill-filesystem` observa sus raíces por defecto, así que cada ciclo de editar-y-crear añade un conjunto de watchers vivos. Acotado por la frecuencia con que se editan las composiciones — que el flujo de autoría de la página de ajustes convierte en un evento por guardado más que por despliegue. Reclamar una necesita un recuento de agents unidos en el montaje permanente; véase el `TODO` en `ensureStanding`.
- **Una copia nunca se monta para validarla** — es byte-idéntica a su fuente, así que una fuente rota en disco produce una copia exactamente tan rota como la fuente; la comprobación de salud del descubrimiento marca ambas filas en la siguiente lectura del roster en lugar de diferir el fallo al arranque de una sesión.
- **La salud es una comprobación de forma, no un montaje** — el descubrimiento prueba que la composición analiza en el dialecto del loader y contiene filas con nombre, no que el módulo de cada fila se resuelva o active; una fila que nombra un paquete ausente sigue fallando en la primera sesión, que revierte la creación.
- **Una copia es una instantánea que deriva** — actualizar el despliegue no actualiza las copias de los presets incluidos, y no hay semántica de parche en esta capa para expresar «standard más un cambio» (eso es el `cordis.patch.yml` de la capa de bundle); el propio conjunto incluido acepta el mismo coste — `cordis` y `code` son copias completas de `standard` — así que todo el ensamblaje sigue siendo legible en un solo archivo.
- **Los escaneos de raíces no se observan** — cada lectura golpea en su lugar el sistema de archivos, lo que mantiene fresco el roster pero pone un `readdir` por raíz en cada `list()`.
