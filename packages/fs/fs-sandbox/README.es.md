# dsh-fs-sandbox — el backend de sistema de archivos que impone el sandbox

[English](README.md) | Español

`SandboxedFileSystem` extiende [`LocalFileSystem`](../fs-local/README.es.md) y se registra como `ctx.fs`. Hereda tal cual cada mecanismo de almacenamiento de texto (resolve, stat, lectura/transmisión, listado, la escritura atómica, la sección crítica de lectura-coincidencia-escritura de la edición) y añade solo una barrera de MODE por llamada sobre `writeText`/`editText`. Las lecturas siempre pasan — todo modo permite leer.

Su config de plugin es la del backend local sin cambios: `cwd` sigue siendo el valor por defecto de resolución de rutas relativas, y `diffBasisMaxBytes` acota la base opcional de diff contextual de la sobrescritura.

Cargarlo EN LUGAR DE `dsh-fs-local`, junto con un [`ctx.sandboxPolicy`](../../sandbox/sandbox-policy/README.es.md), es todo el intercambio; las herramientas orientadas al modelo (`dsh-tool-fs`) quedan intactas. La capa de herramientas resuelve el modo y el cwd de la sesión llamante en la MISMA política por llamada que recibe bash, de modo que las dos familias nunca confinan a raíces distintas.

## La barrera

La política por llamada transporta el modo efectivo (anulación de sesión o concesión de escalada) junto con la raíz de cwd inmutable de la sesión llamante, recurriendo a la política de despliegue solo para llamadas sin ella:

- `read-only` — deniega toda mutación con el `FS_SANDBOX_DENIED` estructurado.
- `workspace-write` — permite una mutación solo cuando el objetivo se canonicaliza bajo una raíz escribible: la raíz del workspace más las áreas temporales de la plataforma (`/tmp`, `os.tmpdir()`), el MISMO conjunto que concede el perfil de Seatbelt, derivado de la única función [`writableRoots`](../../sandbox/README.es.md) para que la barrera de fs y el runner de bash no puedan divergir. Las grafías canónicas usan una vía rápida léxica; un respaldo de ancestros por identidad reconoce raíces equivalentes por alias, como los nombres largos de Windows y los nombres 8.3, sin tratar prefijos no relacionados como contenidos. El objetivo se vuelve a canonicalizar justo antes de delegar, de modo que un enlace simbólico ancestro intercambiado desde que la herramienta lo resolvió queda detectado.
- `danger-full-access` — delega sin barrera.

## Modelo de amenazas: una barrera de política, no un límite de kernel

La barrera es una comprobación en código CONFIABLE sobre una ruta CONTROLADA POR EL MODELO — las operaciones son las del propio seam (open, rename); solo la ruta de destino no es confiable, de modo que canonicalizar-y-luego-confinar es la respuesta completa para esta superficie. Esto refleja la postura de `code-runtime`: contención, no un límite de seguridad. El aislamiento de nivel kernel del CÓDIGO no confiable sigue siendo trabajo de `ctx.shell` ([`dsh-bash-sandbox`](../../shell/bash-sandbox/README.es.md)). El TOCTOU residual (un enlace simbólico ancestro intercambiado entre la re-comprobación de contención y la llamada al sistema) se estrecha al volver a canonicalizar inmediatamente antes de la escritura y se acepta para este modelo de amenazas; un límite ajustado a kernel necesita primitivas de la clase de `openat2` que no valen su coste de portabilidad aquí.

Una denegación es un `FsError` estructurado (`FS_SANDBOX_DENIED`, que transporta el modo efectivo) — sin inferencia de texto de stderr (a diferencia de las denegaciones de kernel de bash), porque una barrera dentro del proceso sabe exactamente qué ha rechazado. El marcador orientado al modelo `[sandbox: file access denied under <mode> mode]` y el reintento de un modo más amplio viven en la capa de herramientas (`dsh-tool-fs`), exactamente igual que los de bash. Ver [el Agent Note del sandbox de fs entre familias](../../../.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.es.md).

## Experiencia del modelo

### Política de sistema de archivos y denegaciones

#### Lo que ve el modelo

El propietario de la política contribuye con contexto `sandbox:policy` neutro en capacidad. Indirectamente, `dsh-tool-fs` renderiza las denegaciones `FS_SANDBOX_DENIED` de este backend como el marcador `[sandbox: file access denied under <mode> mode]` más la pista de escalada en el mismo turno.

#### Efecto en tokens

La cláusula de política actual añade un pequeño mensaje de contexto de ejecución mientras este backend está montado; una denegación añade el marcador acotado y la pista de escalada al historial de conversación.

#### Efecto en la caché KV

Un cambio de política vigente añade una instantánea de contexto de ejecución que la sustituye, renderizada por el propietario, tras el historial retenido; los resultados de operación permanecen de solo anexión.

## Limitaciones conocidas y trabajo diferido

- **Una barrera de política, no un límite de kernel** — la comprobación es código confiable sobre una ruta controlada por el modelo, de modo que el TOCTOU residual de resolver-a-llamada-al-sistema se estrecha (con la re-canonicalización en el lugar) pero no se elimina; los procesos host adversarios están fuera de alcance. El aislamiento de nivel kernel del código no confiable sigue siendo de `ctx.shell`.
- **La paridad barrera-vs-runner se deriva de un único propietario** — el conjunto escribible proviene de `writableRoots`, compartido con el perfil de Seatbelt; un perfil de runner que defina su conjunto escribible en otro sitio divergiría.
- **Requiere `ctx.sandboxPolicy`** — las herramientas lo usan para resolver la política de cada sesión y el backend lo usa para los respaldos de llamadas sin agent; el backend no confina si no está compuesto.
