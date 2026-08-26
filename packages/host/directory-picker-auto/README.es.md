# @deepseek-ai/dsh-host-directory-picker-auto

[English](README.md) | Español

El **selector adaptativo** del [seam de directory-picker](../directory-picker/README.es.md): un plugin solo de mitad de nodo que resuelve la situación del host una vez en el arranque y monta el backend dual-face correspondiente — [`-native`](../directory-picker-native/README.es.md) o [`-browse`](../directory-picker-browse/README.es.md) — como entrada real del Loader en el árbol raíz en memoria (nunca persistida en un archivo de config; el `write()` del árbol raíz es un no-op). Como el backend llega como una entrada ordinaria, su mitad de navegador la descubre la tabla de módulos del cliente exactamente igual que la de una fila de config, así que el invariante del seam de una-fila-intercambia-las-dos-caras se mantiene para la elección resuelta. Descargar el selector vuelve a eliminar la entrada, descargando ambas caras con ella.

La resolución es una muestra pura de tiempo de arranque (`resolveDirectoryPickerBackend`), exportada para reutilización. `native` exige todas las señales de que el operador puede ver el display del host y de que el backend nativo puede servirlo: un bind solo de loopback (leído del `webServer` inyectado; un bind de todas las interfaces admite navegadores remotos a los que ningún selector del SO puede llegar), sin lanzamiento SSH (`SSH_CONNECTION`/`SSH_TTY` sin fijar o en blanco — con port-forwarding SSH el selector se abriría en el servidor desatendido), y una sesión de display servible — asumida en darwin/win32; en linux `DISPLAY`/`WAYLAND_DISPLAY` más un binario zenity o kdialog en `PATH` (la sonda es un hecho más de arranque); nunca en ninguna otra plataforma, porque el backend nativo maneja exactamente darwin/win32/linux. Cualquier ambigüedad resuelve a `browse`, que funciona en todas partes. La muestra ocurre exactamente una vez por arranque para que la capacidad montada permanezca estable durante toda la vida del servicio, como exige el seam. Fijar una interacción no es un campo de config aquí — compón la fila `-native` o `-browse` directamente en lugar de esta, el punto de intercambio documentado del seam; montar el selector **y** una fila de backend juntos falla ruidosamente (servicio `directoryPicker` duplicado, flujo de cliente duplicado en los holes `single`).

## Experiencia del modelo

Ninguna, ya que el selector solo compone la selección de directorios del host de la GUI; nada de esto llega a una solicitud de modelo.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una solicitud de provider.

## Limitaciones conocidas y trabajo diferido

- **La detección infiere la ubicación del operador a partir del contexto de lanzamiento, algo que ninguna señal del lado del lanzamiento puede probar** — una sesión de tmux desprendida de su lanzamiento SSH pierde los marcadores `SSH_*`; un proceso Darwin fuera de una sesión Aqua sigue contando como mostrado; y un lanzamiento local a la estación de trabajo alcanzado después vía `ssh -L` llega desde `127.0.0.1`, resuelve `native` y abre el selector en la estación de trabajo desatendida. Una elección `native` equivocada degrada al diálogo de fallo reintentable existente del backend, y componer `-browse` directamente selecciona la interacción segura para tales despliegues.
- **La sonda del selector en Linux lee solo `PATH`** — un zenity/kdialog alcanzable de otra forma (alias de shell, instalación fuera de PATH) sigue resolviendo `browse`; instalar cualquiera de los dos binarios en `PATH` restaura la elegibilidad de `native` en el próximo arranque.
- **Solo en el arranque** — una resolución sirve a todos los clientes del arranque; la adaptividad por conexión (native para un navegador local, browse para uno remoto, mismo servidor) necesitaría una capacidad por cliente y el anuncio de cable que el seam eliminó deliberadamente, y espera a un despliegue que sirva ambos a la vez.
