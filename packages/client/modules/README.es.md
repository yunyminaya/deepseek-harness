# @deepseek-ai/dsh-client-modules

[English](README.md) | Español

Sistema de módulos de cliente: el par de navegador del loader ESM interno de Node, construido como una tabla CJS perezosa. El shell web monta el Loader de cordis vendido para el gobierno de entradas (ciclo de vida de fiber, espera de inject, update/refresh) e inyecta el `ClientModuleLoader` de este paquete a través de su contrato `internal` — el único punto de consumo del lado vendido es `EntryTree.import`, así que reemplazar `internal` reemplaza exactamente «cómo llega el código de plugin» y nada más.

Modelo CJS perezoso (web2): ejecutar un bundle de plugin solo REGISTRA su factory (`window.__ModuleLoader__.load({id, factory})`); cada efecto secundario del cuerpo del módulo —inyección de CSS incluida— vive en el cierre de la factory y se ejecuta en la materialización (`factory(require)` → exports, memoizado en `loadCache`), no en la ejecución del script. Una factory que requiere otro módulo registrado pero no materializado lo materializa recursivamente; la composición de grafos coloca las peticiones dinámicas declaradas antes que sus consumidores, y los ciclos de require lanzan porque el CJS en forma de factory no puede entregar exports parciales. `<id>/client` y el id desnudo resuelven a los mismos exports (un bundle de plugin ES la mitad de cliente de su paquete).

El Host instala `window.__ModuleLoader__` antes de que se ejecuten las precargas del parser. Su `load()` en modo cola retiene los registros tempranos; `create()` materializa la factory de este paquete con un require de arranque que rechaza externos y llama a su export `createClientModuleSystem`. La construcción almacena en caché esos mismos exports como la fila de módulos, conmuta la misma fachada al registro en vivo y drena la cola restante. El bundle retiene el sistema resultante en un cierre de módulo, así que su `apply()` de Cordis posterior proporciona la instancia idéntica como `ctx.modules` sin otro global de página.

Orden de ramas de resolución (`import(specifier)`): palabra semilla de plataforma → instancia de shell; registro memoizado → exports; fila de grafo (`window.__DSH_BOOT__`) → registrar su factory de script clásico; factory registrada → materializar; cualquier otra cosa lanza — el espejo en runtime de la puerta de pureza del bundle en build-time. El `require` síncrono entregado a las factories recorre el mismo orden menos la carga asíncrona de la fila de grafo y registra las aristas observadas en el registro de módulo. `prefetch` es el hook de llegada de la etapa uno (solo carga de script y registro de factory; las llamadas concurrentes comparten una única tarea en vuelo); `invalidate` descarta una factory no de arranque y el registro materializado para que el siguiente prefetch/import recargue el script (el hook de HMR).

La mitad de nodo escanea las entradas habilitadas del Loader en busca de paquetes web `dsh.client`, resuelve cada `exports["./client"]`, hace hash del bundle construido en el grafo de arranque, transporta las peticiones `dsh.client.external` específicas de paquete, ordena los providers dinámicos antes que los consumidores y sirve cada bundle con su source map bajo `/plugins`. El arranque desde fuente mapea los imports del host al código fuente de TypeScript pero sigue consumiendo este export de cliente construido; los archivos ausentes comparten una instrucción de construcción seguida de una lista de paquete/ruta, mientras que los errores de filesystem no relacionados siguen siendo fallos separados.

`dsh.client.external` es una lista opcional de peticiones de specifier exacto más allá de la línea base implícita: React sembrado por el shell, librerías de Cordis y UI estáticas más runtime precargado por el parser. Una petición se responde con la fila de paquete dinámico que nombra o con una clave exacta de tabla estática; solo un `/client` final aliasa una fila de paquete, y no hay declaración de alias de provider. Los imports de solo tipo se borran y no crean petición. La composición rechaza peticiones malformadas, suppliers ausentes, auto-peticiones y ciclos de petición síncronos; import y prefetch registran recursivamente los suppliers dinámicos antes de que sus consumidores se materialicen. Véase [módulos compartidos y el grafo de módulos](../AGENTS.es.md#shared-modules-and-the-module-graph).

## Model Experience

Ninguna, ya que el loader de módulos es maquinaria de kernel del lado del navegador; nada de esto llega a una petición de modelo.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo diferido

- **Grafo de módulos plano por diseño** — cada bundle es un nodo de módulo cuyas aristas apuntan solo a las hojas de la tabla; la interfaz (`loadCache`/`edges`/`invalidate`) ya soporta un grafo de módulos general, así que la granularidad de externalización puede cambiar sin un cambio de interfaz.
- **Sin contabilidad propia de descarga** — la eliminación de estilos y el orden de desmontaje de fiber viven con el driver de HMR (`@deepseek-ai/dsh-client-hmr`); el loader solo inventaría los ids de etiquetas de estilo que posee por registro.
