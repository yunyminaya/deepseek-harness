# @deepseek-ai/dsh-settings

[English](README.md) | Español

Service Definition de ajustes de usuario (`ctx.settings`). Un único provider mantiene un documento crudo con secciones por namespace; los plugins registran un schema de namespace y leen un valor resuelto en capas: primero los valores por defecto del schema, luego la `base` de composición de quien lo registra (su subconjunto de entry-config de cordis.yml) y por último la sección del documento de usuario. Sin un provider montado nada cambia para los consumers: siguen resolviendo solo la entry config, así que toda composición funciona con o sin ajustes.

## API del servicio

- `documentPath` — ruta absoluta del archivo editable por el usuario del provider cuando lo tiene; los providers sin archivo la dejan en `undefined`. Los adaptadores de configuración del Host derivan de ella la disponibilidad, mientras que los protocolos de navegador exponen solo una capacidad booleana y nunca un destino del sistema de archivos.
- `prepareDocument()` — devuelve esa ruta después de dejar el documento listo para un editor nativo. La implementación base devuelve `documentPath`; un provider de archivo puede materializar primero un documento ausente.
- `register(ns, schema, { base?, applies? })` — devuelve el `SettingsScope` del propietario (`get`/`watch`/`update`). El registro es un efecto en el fiber del plugin llamador: disponer de ese fiber retira el namespace y sus observadores. Una sección almacenada que el schema rechaza hace fallar el propio registro; un namespace duplicado falla en voz alta.
- `describe(options?)` — un descriptor por namespace (envoltorio `schema.toJSON()`, valor resuelto, capas `base`/`user` desacopladas, `applies`) para las superficies de configuración; la presencia de un campo en `user` es lo que lo marca como sobrescrito por el usuario. `describe({ redactSecrets: true })` retira los campos `role('secret')` de todas las capas y añade la lista de slots `secrets` (`{ path, set }`); toda superficie de wire DEBE pasarlo, y el walker puro `redactSecrets(schema, value)` se exporta para otros wires.
- `get(ns)` — valor resuelto, `undefined` mientras no esté registrado.
- `update(ns, patch)` — fusiona en profundidad el parche de objeto plano solo sobre la sección de usuario (nunca sobre `base`), valida el candidato resuelto, persiste a través del provider y luego confirma. Los parches solo pueden contener datos compatibles con JSON: un Date, Map, BigInt, número no finito o referencia circular se rechaza con su ruta enraizada en `$` antes de que nada persista (el almacenamiento YAML/JSON cambiaría en silencio esos valores al recargar). Un fallo de validación rechaza antes de que se persista nada; un provider de solo lectura (`writable: false`) rechaza toda escritura. Las escrituras a un namespace se serializan en el orden de llamada.
- `replace(ns, section)` — fija la sección de usuario por completo: el restablecimiento deliberado (`replace({})` vuelve a heredar `base` y los valores por defecto del schema).
- `mutate(ns, ops)` — aplica ediciones ordenadas `{ op: 'set' | 'unset', path }` a la sección tal como está cuando la escritura llega al frente de la cola. Esta es la vía de eliminación para cualquier llamador que tenga una vista INCOMPLETA: una UI de configuración lee el descriptor censurado, así que reconstruir una sección a partir de él y reemplazarla por completo elimina todos los secretos que el wire nunca devolvió, mientras que un op nombra el único campo que quiere tocar.
- Toda escritura acepta un `expectedRevision` opcional. Cada descriptor lleva la `revision` del namespace, un contador monótono sobre su sección CRUDA; una escritura cuya expectativa ya no coincide rechaza con `SettingsConflictError` (`code: 'SETTINGS_CONFLICT'`, con ambas revisiones adjuntas) en lugar de sobrescribir al escritor que llegó primero. La cola de escrituras ordena las escrituras, pero por sí sola no puede distinguir a un escritor fresco de uno que mantiene una instantánea obsoleta.
- Los valores resueltos son instantáneas congeladas en profundidad. Los watchers reciben `(next, prev)` después de cada confirmación: las invocaciones de un mismo callback se ejecutan de forma asíncrona, una a la vez, en orden de confirmación (una invocación lenta y obsoleta nunca puede aplicarse después de una más nueva), y los fallos — tanto los throws síncronos como los rechazos asíncronos — quedan contenidos. Después de que el disposer de watch devuelve, no arranca ninguna invocación más (una ya encolada se omite); una invocación ya iniciada aún se resuelve. El evento `settings/updated` hace fan-out de un listener a la vez, de modo que un listener que lanza no puede dejar hambrientos a los demás; el rechazo de un listener asíncrono se contiene y se registra, que es por lo que los fallos con código `INVARIANT` solo se vuelven a lanzar desde listeners síncronos.
- El teardown del servicio rechaza nuevas escrituras y nuevos arranques de watcher, y luego drena cada escritura en cola y cada invocación de watcher iniciada antes de que la disposición se complete; una escritura cuyo fiber registrante se dispuso a mitad de vuelo sigue llegando al almacenamiento, pero no confirma ni notifica a nadie.

## Contrato del provider

Las subclases implementan `writable`, `load()` y `persist(ns, section)`, opcionalmente sobrescriben `documentPath` y `prepareDocument()` para un único archivo local editable por el usuario, y empujan los documentos observados externamente a través del `publish(doc)` protegido. El init del servicio base carga y publica el documento una vez antes de que el servicio sea inyectable; un provider con init propio (watcher, conexión) delega primero mediante `yield* super[Service.init]()`. Al publicar, cada namespace registrado se vuelve a resolver de forma independiente: una sección inválida conserva el último valor bueno de ese namespace y avisa — una recarga en vivo nunca tumba el proceso — mientras que la validación en el arranque y en el registro falla en voz alta.

## Eventos

`settings/updated (ns, next, prev, source)` se dispara después de cada confirmación; `source` es `update` (escritura en el proceso) o `provider` (cambio externo). Nunca se dispara por un valor resuelto profundamente igual (deep-equal) — es el evento orientado al consumer, y a un consumer solo le importa que su valor se haya movido.

`settings/document-updated (ns, revision)` se dispara siempre que cambia la sección CRUDA de usuario, cambie o no el valor resuelto. Las superficies de configuración necesitan este: almacenar un override igual a la base de composición deja intacto el valor resuelto, pero cambia lo que dice el documento (el campo ahora está sobrescrito, no heredado) y mueve la revisión que mantiene cada editor abierto. La contención de listeners coincide con `settings/updated`.

Ambas declaraciones viven en el export de subruta `./types` seguro para el cliente, junto con los tipos `SettingsNamespace` y `SettingsUpdateSource` que nombran sus firmas; la raíz del paquete reexporta esos tipos. Así, un consumer fuera de la cara de compilación del Host lee la firma exacta que emite el Host en lugar de reafirmarla.

## Model Experience

De forma indirecta, a través de los plugins consumer que resuelven valores que afectan al modelo (por ejemplo una ruta de modelo predeterminada) desde sus namespaces; la surface propia de cada consumer documenta el efecto.

#### Efecto de KV Cache

Sin invalidación directa; un consumer que pliega un valor de ajustes en el prefijo de petición es dueño de ese cambio.

## Limitaciones conocidas y trabajo diferido

- **Capa de usuario única** — la resolución conoce los valores por defecto del schema, una `base` de composición y un documento de usuario; todavía no registra qué capa aportó cada valor resuelto.
- **`redactSecrets` no es una frontera de wire probada** — el walker sigue `object`/`dict`/`array`, así que un `role('secret')` alcanzable solo a través de una unión, intersección o transformación se devuelve VERBATIM con una lista `secrets` vacía, y `schema.toJSON()` lleva el `.default(...)` de un campo secreto a todo cliente. Ninguno de los dos casos se rechaza; un schema cuyos secretos no sean alcanzables a través de los contenedores recorridos no debe registrarse en un namespace expuesto por wire. Un `describeForWire()` fail-closed — uno que rechace un schema que no puede probar como seguro y sane el envoltorio serializado y el texto de error — es la respuesta real y está diferido.
- **La concurrencia entre procesos la define el provider** — el seam serializa las escrituras por namespace solo dentro del proceso; los procesos concurrentes convergen según el comportamiento del provider (el provider de archivo local hace read-modify-write bajo un candado de escritor, así que los namespaces sobreviven a escritores concurrentes y los conflictos del mismo namespace se resuelven last-write-wins).
