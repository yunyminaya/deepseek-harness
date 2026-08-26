# Agent Note: Selector nativo de directorio de workspace

Status: implemented

[English](2026-07-27-native-workspace-directory-picker.md) | Español

## Problema

La GUI de escritorio pide a los usuarios escribir una ruta absoluta cuando añaden un workspace existente. Es más lento y más propenso a errores que elegir un directorio con el selector nativo del sistema operativo. La GUI se entrega a través del carrier Web local, así que abrir un diálogo nativo también crea una frontera privilegiada que las solicitudes remotas ordinarias no deben cruzar.

## Decisión

Añadir un RPC `host.pickDirectory` de carpeta única y exponerlo a través de `WorkspaceRuntime`. El menú de workspace presenta la acción plana **Add workspace...** (dos acciones cuando esto se decidió — **Open local folder...** junto a una entrada de crear-por-nombre que la [Nota one-route](../simplification/2026-07-31-one-route-to-add-a-workspace.es.md) eliminó después). Seleccionar una carpeta reutiliza el flujo existente `workspace.create({ path })`, selecciona el workspace devuelto e inicia una sesión en blanco.

El gestor de workspaces debe hacer upsert del workspace devuelto antes de que se ejecute el callback de selección. Un directorio recién adoptado renderiza por tanto su basename inmediatamente. Reabrir una ruta ya registrada conserva su título de workspace existente.

## Contrato de interacción

- El selector acepta un directorio en macOS, Windows y Linux.
- Cancelar el diálogo del sistema es silencioso y devuelve `null`.
- Una ruta duplicada selecciona el workspace existente.
- Una ruta canónica diferente adopta un Workspace separado incluso cuando su título derivado coincide con otro Workspace ([decisión de identidad](../bug-fix/2026-07-31-same-basename-workspace-adoption.es.md)).
- Otros fallos del selector muestran un error compacto reintentable.
- El flujo de crear-por-nombre que esta decisión dejó intacto ha desaparecido; elegir un directorio es ahora la totalidad de añadir un workspace ([Nota one-route](../simplification/2026-07-31-one-route-to-add-a-workspace.es.md)).

## Frontera del host

El RPC del diálogo nativo se acepta solo desde un socket de loopback con metadatos de navegador del mismo origen. El RPC no usa el timeout de solicitud por defecto de 30 segundos porque un diálogo del sistema puede permanecer abierto indefinidamente; los aborts del llamador y de la conexión siguen propagándose al proceso de la plataforma.

Los adaptadores de plataforma abren el diálogo sin shell — herramientas nativas spawn en POSIX, una conversación COM en proceso en Windows:

- macOS: `osascript` y el selector de carpetas del sistema.
- Windows: el proceso hijo koffi `IFileOpenDialog` con la mejor conciencia DPI de hilo que el host acepta (per-monitor-v2 cuando está disponible; los hosts sin PMv2 cascadan a per-monitor o system-aware) ([nota del diálogo en proceso](2026-08-02-win32-in-process-folder-dialog.es.md)); el nivel no tiene fallback — los fallos se muestran tal cual ([eliminación de la cadena PowerShell](../simplification/2026-08-04-drop-windows-powershell-picker-fallback.es.md)).
- Linux: `zenity`, con `kdialog` como fallback cuando Zenity no está disponible.

## Alternativas consideradas

- Un navegador de directorios personalizado duplica el comportamiento y los permisos del sistema operativo, y pertenece a la implementación Web en lugar de a este cambio solo de escritorio.
- Reutilizar el campo de ruta manual mantiene la interacción actual propensa a errores.
- Añadir infraestructura de autenticación para un diálogo nativo local único expandiría el cambio más allá de su modelo de amenazas; las comprobaciones de loopback y mismo origen son suficientes para este carrier.

## Consecuencias

La GUI actual abre una carpeta local a través de un selector nativo en macOS, Windows y Linux. Cancelar no cambia ningún estado, los fallos siguen siendo reintentables, las rutas duplicadas son idempotentes y las rutas distintas de mismo basename coexisten como Workspaces separados. El workspace seleccionado y su nombre mostrado se refrescan antes de que comience una nueva sesión en blanco. Este selector es ahora la única ruta a un workspace ([Nota one-route](../simplification/2026-07-31-one-route-to-add-a-workspace.es.md)): el operador elige un directorio existente, o crea uno dentro del selector.

Los tests añadidos de host, runtime, componente y GUI cubren la frontera nativa, las comprobaciones de confianza de solicitud, el manejo de cancelación y fallos, la reutilización de rutas existentes, la adopción de mismo basename y la actualización inmediata del nombre visible. El RPC privilegiado sigue siendo específico del carrier de escritorio local; un navegador de directorios Web remoto queda fuera de esta decisión.

## Riesgos

- Los entornos de escritorio Linux pueden no proporcionar ninguno de los selectores soportados. La GUI informa de esa limitación en lugar de caer a una ruta escrita.
- Los metadatos de navegador varían fuera del carrier local soportado. El endpoint rechaza intencionadamente las solicitudes que no pueden demostrar el contexto local de mismo origen requerido.
