# Agent Note: El seam de ajustes de usuario (`ctx.settings`) y el provider de archivo

Status: implemented

[English](2026-07-28-user-settings-seam.md) | Español

> Ámbito: la familia de capacidades de `packages/settings/` — la Service Definition, el provider respaldado por archivo y la frontera de composición entre los ajustes de usuario y `cordis.yml`. La [nota del config-tree web](2026-07-24-web-config-tree-boot-and-transport-layering.es.md) registró «la ruta de escritura del profile» como aplazamiento; este seam es el dueño de esa ruta de escritura. Las migraciones de Consumers (tema, locale, ruta de modelo por defecto) y la superficie RPC web `settings.*` son seguimientos, no parte del ámbito publicado de esta nota.

## Problema

La configuración editable por el usuario no tenía dueño: `dsh web` leía un json de profile anclado al cwd a través de una whitelist estática sin ruta de escritura, la TUI leía parches de loader crudos de `$DSH_HOME/config.yaml`, y ambos se congelaban al arrancar. Una página de ajustes personales (GUI web) necesita una capa de usuario transversal con validación de schema, ruta de escritura y propagación en caliente — y los productos pares (Codex, Claude Code, Kimi, OpenCode, Pi) convergieron todos en separar las preferencias de usuario de la composición de extensiones. Las actualizaciones de config reactivas al loader no pueden transportar esto: `fiber.update` intercambia la config de entrada en su sitio, de modo que un plugin que leyó la config en la construcción no observa nada y ningún callback le dice lo contrario.

## Decisión

**Dos planos con una prueba de fuego.** `cordis.yml` (+ parches Include) sigue siendo el plano de composición: qué plugins existen, el cableado, la config de despliegue, propiedad del orquestador y actualizado con el producto. Un namespace de ajustes transporta solo el subconjunto editable por el usuario; la prueba es «¿debería editarlo la página de config personal?». Los valores viven en ambos planos sin ambigüedad porque el layering es el contrato: defaults de schema, luego el `base` de composición del registrante (su subconjunto de entry-config), y luego la sección del documento de usuario.

**Frontera de tres paquetes que refleja `session-persistence/`.** `dsh-settings` es dueño del servicio abstracto `SettingsProvider`: registro de namespaces, resolución en capas, validación de schema, detección de cambios por deep-equal por namespace y el evento de confirmación `settings/updated`. Los providers implementan solo `writable`/`load()`/`persist(ns, section)` y empujan los documentos observados externamente a través del `publish(doc)` protegido — de modo que la semántica de actualización en caliente es idéntica en todos los providers, y un backend de centro de configuración de red (estilo nacos, posiblemente de solo lectura) está a un paquete hermano de distancia. `dsh-settings-file` es el provider de archivo: YAML/JSON bajo `resolveSpec` (con default explícito a `<DSH_HOME>/settings.yaml`), watch de chokidar, persists de read-modify-write bajo un candado de escritor entre procesos con commits atómicos `0600` de tmp+rename, parcheado de diff a nivel de hoja del namespace escrito (los comentarios sobreviven a los nodos no tocados) y supresión de autoescritura por igualdad de contenido ([nota de integridad de la ruta de escritura](2026-07-30-settings-write-path-integrity.es.md)).

**Los registros son efectos de la fiber llamadora.** `register()` se ejecuta a través del proxy de servicio, así que `this.ctx` es el contexto del registrante y el registro viaja en `ctx.effect`: disponer del registrante elimina el namespace y sus watchers (demostrado por el test de disposal de HMR), mientras que la sección del usuario sigue viviendo en el almacenamiento para el siguiente dueño.

**Fallar alto en reposo, último-bueno en movimiento.** La validación en el arranque y en el registro lanza (una sección almacenada inválida hace fallar al plugin registrante; un documento existente pero no analizable hace fallar la carga del provider). Una vez en vivo, una edición externa mala avisa y conserva el último estado bueno por namespace — una recarga en caliente nunca debe tumbar el proceso. Esta asimetría refleja `Include.refresh()` y la recarga segura en runtime de Kimi.

**Los Consumers siguen siendo opcionales por construcción.** Un Consumer se registra dentro de `ctx.inject(['settings'], …)`; sin un provider montado sigue resolviendo solo la entry config, de modo que toda composición, demo e instantánea existente funciona sin cambios y la migración es por plugin.

## Alternativas consideradas

- **Write-back de Include como capa de usuario** (páginas de config por plugin que escriben archivos de entrada del loader, estilo cordis-webui): el write-back apuntaría a archivos por composición, ligando las preferencias de usuario a un `cordis.yml`; una capa por usuario debe sobrevivir a las actualizaciones de plantillas y servir a TUI y web desde un único documento.
- **`fiber.update` reactivo al loader como canal de propagación**: las lecturas en tiempo de construcción no observan nada; el `watch()` explícito del seam convierte la actualización en caliente en un contrato de Consumer en lugar de magia del framework.
- **Un servicio de ajustes consciente del dominio** (getters por área de producto): rechazado como acoplamiento; el servicio almacena, valida y publica — el significado de dominio permanece en el registrante que es dueño del schema.
- **Precedencia multicapa ya** (niveles system/gestionado/proyecto al estilo Codex/Claude Code): aplazado hasta que exista una segunda capa real; el paso de resolución es el único lugar donde el layering se extendería.
- **Un lockfile entre procesos ya** (proper-lockfile de Pi): aplazado inicialmente como «reemplazo atómico más convergencia de watcher hasta que aparezca contención real» — pero la convergencia pierde namespaces hermanos no observados, así que el aplazamiento queda superado por el candado de escritor hecho a mano de la [nota de integridad de la ruta de escritura](2026-07-30-settings-write-path-integrity.es.md).

## Consecuencias

Aplazado, en orden de dependencia: la superficie RPC web `settings.raw`/`settings.describe`/`settings.update` (que debe redactar los campos `role('secret')` antes de exponerlos); las primeras migraciones de Consumers (`ui-theme`, locale, ruta por defecto de api-gateway) que retiran `PROFILE_MAPPINGS` y el json de profile; la indirección de valores `${env:VAR}` para secretos; el layering en el lado del provider. La obligación de instantánea sin clave aterriza con el primer Consumer visible por modelo o por usuario del producto, no con este paso de infraestructura.
