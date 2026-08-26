# Agent Note: Monturas en pie por preset sobre una cadena principal de ámbito

Status: implemented

[English](2026-08-08-per-preset-standing-mounts.md) | Español

## Problema

Las monturas de preset por sesión hacían que la superficie del registro frente al modelo fuera por agente mientras tres lectores independientes del host todavía asumían que era estática: el `session.history` en frío no encontraba presentadores (cada tarjeta se degradaba silenciosamente al renderizador genérico — indistinguible de "la herramienta no tiene presentador"), el bloque de proyecciones eliminaba claves registradas por preset (los clientes tratan una clave omitida como ausencia de capacidad y CLEAR la fila), y la puerta de enlace Typert resolvía `goals` en la raíz del host (`service-unavailable`). Parchear cada lector individualmente intercambiaba una degradación silenciosa por otra: reanudar para alcanzar presentadores volteaba la extracción de proyecciones de desconectada a en vivo y borraba los recuentos de tokens en su lugar.

## Decisión

Un preset es una composición por PROCESO, no una por sesión. La lista lo monta una vez bajo un ámbito en pie sintético; cada agente se une vinculando su clave de ámbito al de la montura (`bindScopeParent(agentKey, standingKey)`). Dos mecanismos de `dsh-scope` llevan todo: las vistas de registro recorren la cadena principal (`agent → preset → global`, el más cercano ensombrece al más lejano), y el despacho con ámbito admite escuchas etiquetadas con un ancestro de la clave portadora — solo hacia arriba, así que los escuchas de un preset hermano permanecen sordos.

## Consecuencias

Las monturas en pie arreglan la clase, no las instancias: los registros que un lector necesita existen durante la vida del proceso, con clave por id de preset, sin requerir agente. Lo que lo hizo barato

- Los plugins de preset con estado (`plan-mode`, `token-meter`, `compaction-basic`) ya clavan el estado por `Session`/`Agent` — preceden a los presets. Compartir una instancia es un retorno a su diseño, no una reescritura. `jobs-local` compartió esa propiedad y desde entonces ha dejado el plano de preset por completo: los productores fuera de su reino (`tool-bash`, `tool-terminal`, un `tool-subagent` no continuable) resuelven el registro con `ctx.get`, lo cual un reino local de entrada les oculta, así que se compone en el plano del host y solo la fila `tool-jobs` frente al modelo se mantiene por preset.
- Los ymls de preset están sin cambios: una montura por preset = una Entry por preset, cuyos reinos locales de entrada (`isolate: <name>: true`) mantienen dos servicios del mismo nombre de presets aparte exactamente como mantenían dos sesiones aparte.
- Una etiqueta de reino compartido NO era una opción: `provide()` lanza en un segundo registro bajo el mismo símbolo de reino, así que las etiquetas agrupan el REALM, nunca la instancia — un mundo por sesión compartiendo una etiqueta hace fallar la segunda montura.

## Detalles que soportan carga

- **Las monturas en pie cuelgan del `selfCtx` sin seguimiento del servicio.** Un método invocado a través del proxy rastreable ve `this.ctx` rebotado hacia la persona que llama con una sombra; la resolución de reflexión para cada fibra en un subárbol acuñado a partir de este comienza en la fibra de la sombra, así que las entradas fallan en servicios que su propio `inject` declara (`cannot get property "tools" without inject` mientras el almacén de la entrada lo tiene). El precedente selfCtx de `jobs-local`, ahora con un segundo consumidor.
- **Una montura asentada sirve hasta que cambie la estampilla del archivo de composición.** La composición a la que se unió una sesión en ejecución debe sobrevivir a que el archivo cambie o desaparezca; cada generación registra la estampilla del archivo (mtime + tamaño) y una sesión que la encuentra obsoleta inicia la siguiente generación, así que las ediciones de archivo — el único editor de composición una vez que la autoría se volvió de solo copia — alcanzan sesiones posteriores sin ninguna llamada de autorización que suelte el puntero. Las sesiones unidas mantienen su generación, y las generaciones superadas se reclaman solo por desmontaje de árbol completo — deliberado, limitado por la frecuencia de edición, registrado en las Limitaciones Conocidas del paquete.
- **`peek()` permanece ciego a la cadena.** Las restricciones y guardias abordan las propias contribuciones de un ámbito; solo las VISTAS de registro heredan. Las restricciones a lo largo de la cadena se intersecan (cualquier ámbito puede enmascarar un nombre registrado globalmente para todo lo anidado dentro).

## Alternativas consideradas

Reanudación al leer (borra proyecciones desconectadas), una tabla de presentadores del plano del host más una bandera de completitud de bloque (arregla dos lectores, deja la clase), monturas de plantilla por sesión (duplica cada instancia para servir funciones puras). Guardado para el registro: el dominio `goals` frente a la puerta de enlace sigue siendo del plano del host de todos modos — un método Remoto cuyo receptor proviene de un descriptor generado se resuelve en el host, que es el criterio del plano del host `shell-env` leído desde el lado consumidor.