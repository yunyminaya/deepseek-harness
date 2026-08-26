# Agent Note: Capas del shell del cliente y límites de los paquetes dinámicos

Status: implemented

[English](2026-08-15-client-shells-and-dynamic-packages.md) | Español

> El [modelo de carga de plugins del cliente](2026-07-23-client-plugin-loading-model.es.md) es dueño de la llegada de módulos, el ciclo de vida de Cordis y el HMR. Esta nota es dueña de la colocación de paquetes, las caras de compilación, las peticiones de módulos compartidos y las declaraciones de dependencias npm; esas decisiones superan a la taxonomía de paquetes más antigua y a las reglas de aristas de importación de la nota de carga.

## Problema

Las secciones de dependencias npm del cliente describen relaciones de instalación y desarrollo, pero no describen de forma fiable el contenido del bundle. Tratar `dependencies`, `peerDependencies` o `devDependencies` como instrucciones implícitas para el bundler puede incrustar una identidad React o de workspace compartida, o dejar una biblioteca compilada que arrastra imports hijos sin resolver sin el host que debe ensamblarlos.

La aplicación de navegador también contiene roles diferenciados: la entrada de compilación HTML/Vite, el kernel de arranque de Cordis sin framework, las bibliotecas de ensamblaje estático y los plugins gobernados por el Loader. La ejecución temprana desde el HTML es una política de llegada, no un tipo de paquete. El runtime y los módulos deben llegar antes del módulo principal de Vite conservando los artefactos ordinarios `lib/client.js` y las filas del grafo dinámico.

Las bibliotecas de UI compartidas todavía exponen valores síncronos de TypeScript y React a muchos consumidores. Hasta que esos valores se muevan detrás de servicios o slots, convertir las bibliotecas en entradas dinámicas formales conservaría el acoplamiento de valores mientras oculta qué identidad de módulo debe compartir el shell.

## Decisión

### Capas y formas de compilación

| Capa | Miembros | Responsabilidad | Forma de compilación y carga |
| --- | --- | --- | --- |
| Shell de compilación web | `apps/web` | Es dueño de `index.html`, la configuración de Vite, los chunks de dist y los activos estáticos | Ensambla la salida final del navegador a partir de las exportaciones compiladas de los paquetes |
| Kernel de arranque | `packages/client/web` | Es dueño de la página de arranque DOM plano, el cableado del sistema de módulos, la liquidación de Cordis y la entrega al renderizador | `staticLinked` `lib/index.js`; sin fila `dsh.client` |
| Bibliotecas de ensamblaje estático | Cordis, `ui-primitives`, `ui-slots` | Suministran identidades de módulo compartidas y APIs de valor directas | ESM `lib/index.js`, fusionado y troceado por Vite; sin entradas Loader |
| Arranque de módulos | `packages/client/modules` | Suministra la tabla de módulos del cliente y su envoltorio de Cordis | Paquete dinámico con un `lib/client.js` ordinario; el host entrega su fábrica de forma temprana |
| Paquetes de cliente dinámicos | runtime, `ui-renderer`, theme y los plugins de funcionalidad | Participan a través de servicios, slots y efectos de Cordis | Declaran `dsh.client`, emiten un `lib/client.js` auto-registrado y siguen siendo entradas del grafo del host |

`packages/client/web` conserva Cordis como dependencias peer y de desarrollo coincidentes y usa los módulos y los paquetes de UI estáticos como entradas de compilación de desarrollo. `apps/web` consume las exportaciones compiladas de los paquetes en lugar de alias dentro del código fuente del workspace.

El preset `staticLinked` deja cada especificador desnudo como import externo en `lib/index.js` y emite los CSS relativos a su lado. El host Vite resuelve y deduplica esos imports y decide los límites finales de los chunks. Una biblioteca estática no copia por tanto la política de empaquetado del host en su propio artefacto.

### Peticiones de módulos compartidos

Los bundles de navegador dinámicos externalizan implícitamente la base común: `PLATFORM_MODULES` nombra las identidades React, Cordis y de UI estáticas sembradas por el shell, mientras que `PRELOADED_CLIENT_EXTERNALS` nombra la identidad dinámica precargada por el parser de runtime. Un paquete usa `dsh.client.external` solo para una petición de valor exacta que no pertenezca a la base. Los imports solo de tipos se borran y no crean petición; las bibliotecas de implementación de terceros permitidas siguen siendo contenido privado del bundle.

Una petición tiene exactamente dos suministradores:

1. La fila de paquete dinámico que nombra; un `/client` final aliasa esa fila de paquete.
2. Una clave exacta en la tabla de módulos estáticos del shell.

No existe un mecanismo general de alias `dsh.client.provide`. Las filas dinámicas y las claves estáticas agotan los suministradores reales, mientras que la provisión de servicios de Cordis permanece independiente. La composición del grafo rechaza las peticiones malformadas o ausentes, las auto-peticiones y los ciclos síncronos de peticiones, y ordena los suministradores dinámicos antes que sus consumidores. `ClientModuleSystem.import()` y `prefetch()` registran de forma recursiva esas fábricas de suministradores dinámicos antes de que el consumidor pueda materializarse, de modo que el tiempo de red no puede violar el grafo síncrono de peticiones.

### Precarga del parser y entrega a React

La mitad Node de los módulos inyecta el protocolo de arranque en el HTML servido en este orden:

1. Instala `window.__ModuleLoader__` en modo cola con `pendingQueue`, `load()` y `create()`.
2. Ejecuta el `lib/client.js` ordinario de la fila del grafo de módulos como script clásico bloqueante.
3. Ejecuta el `lib/client.js` ordinario de runtime del mismo modo.
4. Asigna `window.__DSH_BOOT__`.
5. Ejecuta el módulo principal de Vite.

Ambos scripts tempranos solo registran fábricas. El kernel de arranque pasa el grafo crudo y las semillas del shell a `__ModuleLoader__.create()`. La fachada elimina el registro de módulos, lo materializa con una función `require` que rechaza todo externo, e invoca su exportación `createClientModuleSystem`. El bundle de módulos parsea el grafo, construye `ClientModuleSystem`, guarda sus propias exportaciones como la fila de módulos y retiene el sistema en un cierre de módulo. La construcción conmuta la misma fachada a modo vivo antes de drenar la fábrica pendiente de runtime. La cara de cliente de módulos tiene por consiguiente un requisito de arranque con cero externos en runtime.

Tras registrar sus fábricas el nivel `immediately`, el kernel crea todas las entradas Loader, espera la quietud de Cordis y exige que toda fibra esté ACTIVE. Después llama a `ctx.uiRenderer.mount(container)`. El paquete dinámico `ui-renderer` es dueño de React, el renderizado de slots, la hidratación del DOM de arranque existente y el ciclo de vida de la raíz React; el kernel de arranque y la página de fallo permanecen libres de React.

### Declaraciones de dependencias

Todo paquete de cliente conserva Cordis en `peerDependencies` y `devDependencies` coincidentes. Un paquete dinámico que importa, re-exporta, aumenta o nombra un paquete dinámico interno en `dsh.client.inject` conserva ese paquete como dependencias peer y de desarrollo coincidentes. Las entradas estáticas de cliente y los módulos React son entradas solo de desarrollo para un paquete dinámico porque el shell suministra sus identidades de runtime.

Las bibliotecas instaladas ordinarias siguen siendo `dependencies`: una compilación dinámica puede empaquetar una implementación privada, mientras que una biblioteca `staticLinked` retiene su import desnudo para el host final. Cada cara de compilación decide la externalidad independientemente de las secciones npm. Las listas de archivos publicados cubren toda entrada de runtime, activo relativo y archivo de declaraciones alcanzado por el artefacto.

`verify-client-packages` aplica estas clasificaciones, secciones de dependencias, formas de compilación, alineación de precarga del parser, peticiones de módulos compartidos y aciclicidad del grafo de módulos. El pase publint del repositorio aplica el cierre de publicación. El modo `--fix` del verificador solo repara desvíos de manifest inequívocos.

## Alternativas consideradas

**Convertir cada paquete de cliente en un plugin dinámico de inmediato.** `ui-primitives` y `ui-slots` todavía suministran valores síncronos sin ciclos de vida de servicio o slot independientes; una declaración de manifest por sí sola no eliminaría esos imports.

**Generar un `client-static.js` separado para módulos o runtime.** Ambos paquetes siguen siendo filas del grafo dinámico y plugins de Cordis; solo su llegada de fábrica es temprana. Un segundo artefacto codificaría la política del host en un nombre de archivo y crearía dos productos de runtime a partir de una sola fuente.

**Compilar todos los módulos compartidos en la entrada de Vite.** Esto eliminaría la composición de despliegue y el reemplazo a nivel de plugin de los plugins de negocio, incluidos el renderizador y el tema.

**Conservar una declaración general de proveedor de módulos.** Las filas de paquete y las claves estáticas exactas ya nombran todos los suministradores; los alias añadirían otro protocolo de titularidad sin una tercera fuente de suministro.

**Fijar las URLs de precarga en `apps/web/index.html`.** Las URLs y los valores `rev` pertenecen al grafo actual del host. Reescribir el HTML servido mantiene la cola, las URLs del bundle y el manifest en una sola revisión del grafo.

## Consecuencias

El contenido del bundle permanece estable cuando una dependencia npm se mueve entre las secciones peer y de desarrollo, porque cada cara de compilación declara la externalidad directamente. Las bibliotecas estáticas siguen siendo ensambladas por el host, mientras que los paquetes dinámicos conservan artefactos uniformes y gobernanza de ciclo de vida.

El protocolo de arranque depende de los ids de paquete de módulos y runtime, y los módulos deben permanecer autocontenidos en runtime. Un registro de arranque ausente falla antes de que Cordis arranque; los fallos posteriores de import, apply y espera de servicios de plugins siguen siendo visibles a través del escaneo ACTIVE de la página de arranque.

El shell consume los productos compilados `lib/`, de modo que el código fuente y los artefactos de navegador pueden desviarse hasta que se ejecute la compilación o el watcher correspondiente. Solo comprobar tipos del código fuente no demuestra que la aplicación servida use el mismo código.

Las dos bibliotecas de UI estáticas siguen siendo excepciones deliberadas. Convertir cualquiera de ellas en paquete dinámico exige mover todos los consumidores de valores a servicios o slots y retirar su identidad de la semilla estática en el mismo cambio.
