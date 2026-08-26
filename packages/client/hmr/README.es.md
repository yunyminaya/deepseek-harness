# @deepseek-ai/dsh-client-hmr

[English](README.md) | Español

Recarga en caliente para los plugins de cliente cargados por script. El bundle web monta la fila de forma incondicional; sin un watcher de reconstrucción (`pnpm run dev:web`) que reescriba los bundles de cliente, el sondeo no observa cambios y la cadena permanece inactiva.

La mitad de navegador se suscribe al canal SSE del sistema (`GET /plugins/events`) y recarga un plugin por cada frame `rebuilt` mediante una cola serializada. La secuencia por frame — `invalidate`, `prefetch` (carga y registra el bundle nuevo mientras el fiber viejo sigue sirviendo), `registry.delete` (antes del fiber: una disposición de fiber desnuda dispara la rama de auto-disposición del Loader vendido, que marcaría la entrada como deshabilitada), drenar el fiber viejo, borrar `entry.fiber`, eliminar las etiquetas `<style data-plugin>` propiedad del plugin, `entry.refresh()` vuelve a importar y remonta, `fiber.await()` relanza los fallos de arranque en voz alta. Los dependientes se recargan a través del propio cordis: la época de activación de un fiber encadena los uids de sus providers de servicio, así que reemplazar el fiber de un provider en cascada a cada dependiente sin análisis de grafo del lado del cliente. La mitad de nodo detecta las reconstrucciones con un intervalo que hace stat-poll a cada bundle del grafo desde una línea base síncrona, vuelve a hacer hash inmediatamente después de añadir una fila, retiene las filas ausentes como sucias y difunde solo los cambios de rev reales; cualquier proceso de watch de tsdown que produzca el bundle dispara por tanto HMR sin canal constructor→host.

## Model Experience

Ninguna, ya que el driver de recarga es maquinaria del lado del navegador; nada de esto llega a una petición de modelo.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo diferido

- **La recarga es gruesa por diseño** — un fiber nuevo y componentes nuevos; el estado de React dentro del plugin recargado se pierde mientras la capa de datos (fibers de connection/runtime, objetos de Session) queda intacta. La preservación de estado a la altura de react-refresh entra en conflicto con «re-ejecutar el bundle vuelve a ejecutar la factory» y queda excluida deliberadamente.
- **Sin reversión de fallos** — una recarga que falla deja la entrada en FAILED y visible en la proyección de estado del loader; el bundle anterior no se restaura automáticamente.
- **La rev de grafo no se refresca con los frames reconstruidos** — la rev obsoleta es inofensiva porque el endpoint del bundle sirve sin caché; solo la reconexión la refresca.
