# dsh-client-ui-settings-plugins

[English](README.md) | Español

La sección de ajustes **Plugins** y su pestaña **Configuración de plugin**. La sección es la dueña del encabezado y del chrome compacto de la pestaña; los plugins de funcionalidades aportan páginas a través de `settings.plugins.tab`. La pestaña propia de este paquete muestra una tarjeta expandible por plugin del Host cuya configuración es de un usuario. Una tarjeta muestra el nombre del plugin y qué gobierna; expandirla en su sitio revela controles escritos a mano vinculados al espacio de nombres de ajustes de ese plugin, y cada campo marca si el usuario lo anuló y ofrece un restablecimiento al valor que compuso el despliegue.

## Qué aparece aquí

La pestaña configurable lee qué espacios de nombres de ajustes sirve el Host y despacha una clave de slot por espacio de nombres, así que lo que se renderiza es la intersección de dos registros: los espacios de nombres que registró un plugin vivo del Host y las tarjetas registradas bajo esas claves. Un espacio de nombres servido que ninguna tarjeta reclama no renderiza nada — otra superficie es la dueña, o este despliegue no trae una mitad de navegador para él — y una tarjeta cuyo espacio de nombres este despliegue no sirve nunca se despacha, de modo que un plugin no compuesto no deja rastro y no retiene la pestaña en su línea vacía. La línea vacía espera la primera respuesta del Host, así que una lectura sin respuesta nunca se lee como «este despliegue no configura ningún plugin». Las tarjetas aparecen en el orden en que se registraron, que es estable para las tarjetas que un paquete instala juntas y no estable entre plugins: el orden de apply entre paquetes no está restringido.

Las tarjetas que este paquete trae cubren el ejecutor de shell (`bash`), el paralelismo de llamadas de herramienta del agent loop (`agent-loop`) y el provider de búsqueda de DeepSeek (`web-search-deepseek`).

## Punto de extensión

La sección declara `settings.plugins.tab`, un slot de lista raíz cuyas etiquetas se convierten en pestañas ordenadas. Mantiene montada una pestaña tras su primera selección, así que los borradores locales y las instantáneas de solo lectura sobreviven a los cambios de pestaña. El paquete registra su propia contribución `configurable`, que declara el slot anidado `settings.plugin.item` — con clave en el espacio de nombres de ajustes que edita una tarjeta. Un plugin que trae una mitad de navegador registra su propia tarjeta bajo su propio espacio de nombres y es dueño de cada parte: chrome, controles y texto. Usar la clave del espacio de nombres es lo que permite que un plugin distribuido fuera de este repositorio aparezca aquí — registra el espacio de nombres en el Host y la tarjeta en el navegador, y la pestaña empareja ambos sin aprender qué significa el espacio de nombres. Las pestañas siguen el `order` de la contribución; las tarjetas siguen el orden de registro.

## Escrituras

Una tarjeta prepara lo que el usuario escribe y solo lo escribe cuando este guarda. Cada control renderiza el texto preparado, así que lo que hay en pantalla es exactamente lo que almacenaría un guardado; **Descartar** elimina los borradores, y una tarjeta con ediciones sin guardar lo dice en su cabecera incluso plegada. Un restablecimiento prepara el valor por defecto compuesto en lugar de escribir de inmediato, y un borrador que el campo no acepta bloquea el guardado en lugar de descartarse.

Guardar escribe cada campo preparado a través del alcance de ajustes del cliente, que acota cada escritura con la revisión del espacio de nombres que leyó, así que un formulario que se ha desviado del documento se rechaza en lugar de sobrescribir un cambio concurrente. El Host es la única autoridad sobre si un valor se aceptó — sus validadores son los dueños de las restricciones que ningún schema puede expresar — así que la tarjeta relee la sección después y notifica un guardado que no aterrizó, conservando esos borradores para que el usuario los corrija.

Una clave también puede escribirse desde otra superficie — la página Models aborda la misma referencia — lo que no cambia ninguna sección de ajustes, así que la tarjeta relee en el evento reenviado `credentials/reference-updated` para la referencia que vigila.

La presencia de un campo en la capa de user en bruto — no su valor — es lo que lo marca como anulado; un restablecimiento limpia ese campo para que vuelva a heredar la capa de composición. Los campos con rol de secreto nunca viajan en una respuesta, así que un control de clave arranca en blanco, solo informa de si hay una configurada y escribe a través del dominio de credenciales en lugar de la sección de ajustes; un borrador en blanco no escribe nada y conserva la clave almacenada.

## Model Experience

Ninguno: la sección renderiza una UI de configuración de navegador; los valores que escribe solo llegan a un modelo a través de los plugins que los poseen, y cada uno documenta ese efecto por su cuenta.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo pendiente

- **Solo aparecen los plugins del plano del host** — un plugin que monta un preset de agente lleva su configuración en línea en el `agent.cordis.yml` de ese preset y no puede registrar ningún espacio de nombres de ajustes (una segunda sesión que montara el mismo preset fallaría por un registro duplicado), así que esta sección no lista nada para él. Editar esos valores sigue siendo trabajo del editor de presets.
- **Una tarjeta sigue necesitando un bundle de navegador** — la mitad de navegador debe ser un paquete `dsh.client` compilado en el formato de fábrica lazy-CJS del sistema de módulos del cliente, y el preset `clientBundle` que lo emite vive en `packages/client/tsdown.client.ts` en lugar de en un paquete publicado, así que un plugin fuera de este repositorio tiene que reproducir ese build por su cuenta. La compuerta de pureza del bundle también prohíbe importar el chrome de tarjeta o el modelo de formulario de este paquete como valores, de modo que esa tarjeta es dueña de su propia preparación y acotación de revisión.
- **Los espacios de nombres servidos solo releen con dos señales** — el wire anuncia confirmaciones de documento de ajustes y reinicios de conexión, no registros, así que un espacio de nombres cuyo dueño se registra después de la lectura de la pestaña se une a la lista en la siguiente confirmación de documento o reconexión.
- **La tarjeta de shell sigue al ejecutor compuesto** — las familias de ejecutores POSIX y PowerShell comparten el espacio de nombres `bash` porque un host compone exactamente uno de ellos, así que el schema servido difiere según la plataforma (PowerShell añade `pwshPath`) aunque la tarjeta edite los mismos dos campos en ambos, y un despliegue que no compone ninguno no muestra tarjeta.
