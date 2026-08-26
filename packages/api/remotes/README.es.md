# @deepseek-ai/dsh-api-remotes
[English](README.md) | Español

BFF bidireccional para las capacidades Remote de Host seleccionadas por esta aplicación. La entrada de Host es dueña de la política de identidad de agent (agente)/sesión; la entrada de Client importa los artefactos `/remote` generados como valores de runtime, monta cada contribución mediante `ctx.remote.$mount()` y reexporta sus fusiones de declaraciones. Los paquetes de negocio de Client dependen de esta fachada en lugar de la implementación del Gateway o de las entradas de runtime de Remote individuales.

`createApiRemoteAgentResolver()` reutiliza los agents en vivo, reanuda sesiones frías ordinarias, deduplica las reanudaciones concurrentes, preserva la valla de propiedad de subagentes (subagent) y configura el mismo resolver para las búsquedas Typert `agent` y `session`. El Web API Proxy estándar aporta los valores predeterminados de Agent y la configuración de ámbito, y luego usa el resolver devuelto para los métodos heredados, de modo que los métodos migrados y no migrados comparten una única implementación de política.

El ensamblaje de Client actual monta la contribución Goal Remote y la contribución de inventario de plugins de Host de solo lectura (`pluginInventory/list`). La propiedad de effects de Cordis retira cada contribución cuando este ensamblaje se descarga, mientras que `@deepseek-ai/dsh-api-gateway/client` es dueño de la validación de descriptores, los Services de namespace con seguimiento, los métodos directos y con ámbito, la invocación y la cancelación. La entrada de Client consume la interfaz compartida `TypertClientRemote` a través de Cordis y no importa el Gateway concreto. Reexporta las fusiones de declaraciones de la cara de Gateway de Client solo a nivel de tipos, de modo que un consumidor que llegue al vocabulario de eventos reenviados a través de esta fachada no obtiene ninguna ventaja de runtime sobre la implementación del Gateway.

Este paquete no contiene lógica de transporte ni de descubrimiento de servicios de Host. Su cara de Client puede reutilizarse desde Web o desde un futuro TUI que aporte el mismo contrato `ctx.remote` sin React.

## Eventos de Host reenviados

`src/remote-events.ts` contiene `API_REMOTE_FORWARDED_EVENTS`, la lista de permitidos de eventos de Host de Cordis que esta aplicación reenvía a los consumidores tal cual — sin proyección, sin redacción, sin renombrado — y por tanto el conjunto de claves válidas de `ctx.remote.$on`; el `src/types.ts` solo de tipos deriva su cara de selección. Reenviar un evento más es una entrada en ese array y nada más: la proyección de tipos, la cara de claves del consumidor y el bucle de reenvío del Host se derivan todos de ella.

La firma del listener no se reitera aquí. La declaración `Events` de Cordis de cada evento de la lista de permitidos vive en la exportación `./types` segura para Client de su paquete propietario (`dsh-agent-presets`, `dsh-commands`, `dsh-credentials`, `dsh-llm`, `dsh-settings`), y ambas caras de este paquete incorporan esas declaraciones, de modo que «reenviado tal cual» se cumple por construcción, no por demostración. La cara de Host además contrasta la lista contra `TypertForwardableEvent`, que rechaza un nombre que no sea un evento declarado, uno que enlace un AgentScope y uno cuya forma no sea unidireccional.

## Frontera de compilación

Un paquete ordinario del repositorio pertenece a una única cara de TypeScript: los paquetes de Host se registran en el `tsconfig.host.json` raíz y los de Client en el `tsconfig.client.json` raíz. `api-remotes` es la única excepción deliberada porque su entrada de Host debe participar en el grafo Typert de Host, mientras que `src/client/index.ts` no puede compilar hasta que el tsdown de Host haya generado las declaraciones `/remote` de los paquetes de negocio.

El `tsconfig.json` raíz de este paquete es solo una solución que referencia `tsconfig.host.json` y `tsconfig.client.json`. El agregado de Host y los consumidores directos de Host referencian el primero, mientras que el agregado de Client y los consumidores directos de Client referencian el segundo; la solución raíz del paquete no debe entrar en el grafo de dependencias de ninguno de los dos agregados. Los dos proyectos son dueños de conjuntos disjuntos de archivos fuente y de archivos `.tsbuildinfo`, pero comparten el directorio de salida `lib/types`, con una única excepción deliberada: `src/remote-events.ts` y `src/types.ts` figuran en los `files` de AMBAS caras, porque la lista de permitidos de eventos reenviados es el único punto de control sobre lo que un consumidor puede recibir, y el bucle de reenvío del Host y la cara de claves `ctx.remote.$on` del Client deben leer una sola declaración en lugar de dos que podrían divergir.

Esa excepción no es solo una entrada de `files`. El `tsconfig.base.json` raíz mapea `@deepseek-ai/dsh-api-remotes/types` a `src/types.ts` — el plano fuente, como cualquier otro subpath de workspace y a diferencia de los artefactos `/remote` generados, que no tienen entrada `paths` y se resuelven a través de `exports` hasta la salida compilada. Ambas caras admiten por tanto la misma lista de permitidos y la misma proyección de tipos en sus propios programas y emiten salidas `remote-events` y `types` byte a byte idénticas en `lib/types`; los `.tsbuildinfo` permanecen independientes. Ninguna verificación impone la disyunción de archivos fuente entre caras — `scripts/project-reference-faces.ts` solo comprueba que una referencia a un proyecto dividido nombre la cara correspondiente — así que este párrafo registra por qué el doble listado es intencional.

El `clientBundle(..., { hostPhase: true })` local al paquete hace que el tsdown de Host compile la entrada de Host y que el tsdown posterior de Client compile solo la entrada de navegador. Los plugins de Client ordinarios siguen siendo proyectos de Client únicos y producen tanto su entrada de loader de Node como el bundle de navegador durante el tsdown de Client; no copies la división de este paquete solo porque un paquete tenga `src/index.ts` y `src/client/index.ts` a la vez.

## Experiencia de modelo

Ninguna, ya que este BFF selecciona los métodos de aplicación de Remote y la política de identidad, pero no registra nada orientado al modelo.

#### Efecto de KV Cache

Sin efecto directo; las capacidades de Host montadas son dueñas de cualquier comportamiento visible para el modelo que disparen.

## Limitaciones conocidas y trabajo diferido

- El conjunto de capacidades queda fijado por imports de valor explícitos en tiempo de compilación; el Client no descubre en runtime los Services activos del Host ni las definiciones de Remote.
- Las capacidades adicionales requieren un import de valor `/remote` explícito y un montaje en este ensamblaje.
- El Web Host estándar aporta los valores predeterminados de reanudación y la configuración de ámbito de Agent desde el API Proxy heredado hasta que esa configuración BFF restante se traslade a `api-remotes`.
