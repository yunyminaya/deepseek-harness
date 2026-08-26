# Agent Note: Los providers de subagentes de producto viven en el host de perfil compartido

Status: implemented

[English](2026-08-10-product-subagent-providers-in-shared-host.md) | [中文](2026-08-10-product-subagent-providers-in-shared-host.zh.md) | Español

## Problema

Los [contratos de provider de Codex y Claude Code](../feature/2026-08-04-claude-code-and-codex-subagent-backends.es.md) se lanzaron por primera vez como paquetes instalables de forma independiente que un despliegue cargaba junto a la herramienta de subagentes común. Los Agent Presets se convirtieron después en el dueño ordinario de las herramientas visibles al modelo de un agent, pero un preset no puede ser dueño de estos providers de producto con seguridad: `ctx.subagents` es un registro de procesos, los nombres de provider son únicos dentro del Host y los consumidores del host resuelven el mismo registro entre sesiones. La composición repetida de presets contendería por tanto por los mismos nombres configurados. Exigir a una persona que edite a la vez un Profile y un Preset haría además que una fila de preset genérica fuera incompleta por sí misma.

La decisión de ubicación debe preservar dos hechos independientes. Cargar un provider no debe arrancar ni autenticar un producto, mientras que conceder una herramienta debe seguir siendo por preset para que dos sesiones puedan exponer productos distintos. Un conmutador global de producto, una instancia de provider por agent o presets de combinación pre-enumerados crearían cada uno un segundo dueño para uno de esos hechos.

## Decisión

Los providers de producto siguen siendo registros del plano del host con ámbito de proceso. La [decisión de exclusión de la instalación de producción](../simplification/2026-08-12-production-dsh-excludes-product-subagent-providers.es.md) supera solo la elección anterior de este note de instalar en el bundle base: el `dsh-base` de producción ni depende de ellos ni los monta. Un Profile que se adhiere instala el Bundle del provider seleccionado; su parche monta la instancia por defecto, y el Profile puede montar instancias con nombre adicionales en el plano del host. La [decisión de instancias con nombre](../feature/2026-08-18-product-subagent-named-instances.es.md) es dueña de la identidad de registro de cada fila: ambos productos aceptan múltiples valores únicos de `providerName` y conservan `codex` y `claude-code` como sus valores por defecto. Cargar cualquiera de los dos plugins solo registra un backend inactivo; el proceso de Codex o Claude correspondiente se arranca en la primera llamada de delegación real. Los Agent Presets contribuyen de forma independiente filas ordinarias de `dsh-tool-subagent` cuyos valores de `provider` y `toolName` exponen exactamente las instancias configuradas que necesita un agent sin cambiar el registro del Host.

Cada paquete de provider es dueño de su parche de Bundle directamente instalable y de su runtime de producto privado. Este note sigue siendo dueño de la ubicación en el Host a nivel de proceso cuando cualquiera de los dos providers está instalado. El note del contrato de provider sigue siendo dueño de cada protocolo de producto, del mapeo de resultados, de la cancelación, del ciclo de vida del árbol de procesos y de los niveles de evidencia. La [arquitectura de Agent Presets](2026-08-03-per-session-agent-presets.es.md) sigue siendo dueña de la división Host/Agent, de la autoría de presets y de la regla de que las ediciones afectan solo a las sesiones recién compuestas.

Cada Bundle delega la selección del ejecutable en su runtime de producto propiedad del paquete: el paquete de Codex ejecuta su wrapper declarado, mientras que el paquete de Claude Code deja que su Agent SDK fijado seleccione el ejecutable nativo privado. Ninguno de los dos providers consulta ni recurre a un comando de producto del host. La carga del Profile no crea estado de producto, no sondea versión ni autenticación y puede suministrar la configuración de despliegue de cada instancia de Provider montada, incluidos los valores de `permissionMode` específicos del producto que son dueños de la [decisión de permisos no interactivos](../feature/2026-08-15-product-subagent-noninteractive-permissions.es.md), sin mover esas elecciones a un Agent Preset ni a una herramienta visible al modelo. Las cargas de plataforma ausentes y los fallos de producto quedan localizados en la delegación intentada.

## Verificación

La prueba del bundle base demuestra que el `dsh-base` de producción no contiene ni la dependencia de ningún provider de producto ni su fila de provider. La composición de Web instala ambos Bundles opcionales y cubre ninguno, solo Codex, solo Claude y ambos conjuntos de herramientas, incluido el aislamiento de generación después de que un preset redactado cambie. Las composiciones de Loader propiedad del paquete demuestran que cada Bundle por defecto y cada instancia con nombre adicional se registran sin arrancar un proceso de producto. Las instantáneas ACP sin clave fijan el listado de dos herramientas de Codex y la combinación final de cuatro herramientas, mientras que las pruebas de provider demuestran por separado la selección privada de cargas de plataforma sin recurso al host, el aislamiento de configuración, el fallo, la cancelación y la quietud del árbol de procesos.

## Alternativas consideradas

**Mantener los providers de producto opt-in en la capa de Profile.** Esto preserva un cierre de dependencias por defecto más pequeño, pero exige que el usuario edite a la vez un Profile y un Preset. La decisión de exclusión de la instalación de producción acepta esa compensación de instalación; este note conserva el requisito de que las instancias de provider seleccionadas se montan en el plano del host y no dentro del preset.

**Almacenar conmutadores de habilitación de producto globales o por Profile.** Un conmutador de proceso compite con el Preset como dueño de las herramientas visibles al modelo y no puede expresar que dos sesiones usen combinaciones distintas. La disponibilidad y la autenticación son hechos de despliegue, no otro estado de producto persistido.

**Montar los providers dentro de cada Agent Preset.** Los nombres de provider pertenecen a un registro de procesos, de modo que la composición repetida de sesiones colisionaría con los mismos nombres configurados. Los consumidores del host también necesitan el registro con independencia del ciclo de vida de cualquier agent concreto.

**Publicar cuatro presets de combinación de producto.** Cuatro identidades duplicarían composiciones completas para representar dos filas de herramienta independientes. Las filas ordinarias ya expresan la matriz completa sin añadir estado de listado ni de mantenimiento.

## Consecuencias

Un usuario instala cada provider de producto seleccionado en un Profile, monta las instancias con nombre requeridas y expone sus herramientas por el mismo camino de autoría de Agent Presets que el resto de plugins. Cada sesión nueva recibe exactamente las herramientas que contribuye su preset elegido. Los Profiles que no seleccionan un provider de producto no llevan el paquete correspondiente ni huella de carga de módulos; cargar instancias seleccionadas sigue sin arrancar proceso de producto, inicio de sesión, llamada de modelo ni home de producto.

El registro del Host sigue siendo la única autoridad de provider, cada Bundle sigue siendo la autoridad de disponibilidad de despliegue y cada Preset sigue siendo la autoridad de herramientas de modelo. Este ciclo de vida explícito de dos compuertas evita un conmutador de habilitación global y mantiene la eliminación de paquetes independiente de la autoría por sesión.
