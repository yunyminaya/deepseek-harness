# Introducción a Cordis

[English](cordis-primer.md) | Español

Cordis es el framework de plugins vendored que sustenta DeepSeek Harness. Esta introducción enseña las ideas de Cordis que un autor de plugins del harness necesita antes de leer la referencia generada de servicios y eventos en las [páginas de subsistemas](subsystems/core.es.md); el [tutorial de Cordis](cordis-tutorial/index.es.md) recorre las mismas ideas con las manos en la masa. El código fuente vendored y el procedimiento de sincronización viven en [vendor/README.md](../vendor/README.md).

## Cordis en cinco ideas

- **Un plugin es un objeto que implementa Service.** Puede ser una función con campos opcionales `inject` y `apply(ctx)`, o una subclase de `Service` cuyo ciclo de vida Cordis monta en el contexto actual.
- **Un contexto es un repositorio de servicios.** Un servicio reclama una `ctx.<key>` estable, como `ctx.tools`, `ctx.llm` o `ctx.sessions`, de un contexto; los demás plugins encuentran los servicios por clave en lugar de importar una implementación concreta.
- **Declara la dependencia de servicios mediante `inject`.** Un plugin que nombra los servicios requeridos espera hasta que esos servicios existan; así, el orden de carga se expresa mediante los requisitos de servicio en lugar de una secuencia de arranque manual.
- **Eventos tipados para la comunicación.** Los servicios declaran los nombres de evento mediante la fusión de declaraciones de TypeScript y luego los despachan como `emit`, `waterfall`, `parallel` o `serial` según los listeners observen, envuelvan, se ramifiquen o se ejecuten en orden.
- **Los registros son efectos reversibles.** Las secciones de prompt, los schemas de herramientas, los adaptadores, los providers y los listeners se instalan mediante `ctx.effect()` o `ctx.on()` para que la recarga y el desmontaje los deshagan de forma predecible.

## Modos de despacho

Cada evento puede tener uno de los siguientes modos de despacho y solo puede despacharse mediante el método correspondiente.

| Modo | ¿Con espera? | Orden de despacho | ¿Devuelve valor? |
|---|---|---|---|
| `emit` | No | los listeners observan en orden de registro | No |
| `waterfall` | No | los listeners observan en orden de registro | Sí |
| `parallel` | Sí | todos los listeners observan el evento en paralelo | No |
| `serial` | Sí | los listeners observan en orden de registro | Sí |

El modo de despacho forma parte del contrato público del evento. Los eventos nuevos del harness lo documentan con una etiqueta `@mode` para que el catálogo generado pueda comprobar las declaraciones contra los lugares de despacho.

## Semántica del waterfall en Cordis

`ctx.waterfall` es middleware envolvente (around-middleware). Un listener recibe `(...args, next)`. Llama a `next()` para delegar el resultado, posiblemente envuelto, al siguiente servicio; devuelve sin `next()` para cortocircuitar. Los valores se propagan a través del valor de retorno de `next()`.

Los listeners cooperativos normalmente mutan un objeto de petición o decisión compartido y luego delegan. Un listener también puede optar por sustituir el resultado por completo, y los listeners posteriores solo verán el resultado tras la sustitución. Usa `prepend: true` únicamente cuando el listener deba ejecutarse antes que los registros ordinarios.

Para los eventos de decisión única, el cortocircuito es el diseño. Un listener de política puede devolver sin `next()` cuando la decisión es suya, mientras que un listener que solo anota u observa debe delegar.

## Configuración del loader

`@deepseek-ai/cordis-plugin-include` analiza `!!js` en nodos de expresión. El Loader interpola el `config` de una entrada (una vez activadas las inyecciones declaradas, contra el contexto de ese plugin — `ctx.serviceName`) y su campo `disabled` (en cada decisión de montaje, contra el contexto del loader); Include conserva las expresiones de fila anidadas hasta la activación del objetivo. El resto de metadatos de la entrada permanece literal. Usa overlays cuando el entorno selecciona plugins.

## Reglas prácticas

Encapsula el comportamiento en plugins: un evento de la pipeline de herramientas pertenece a `ctx.tools`, el streaming de modelos a `ctx.llm` y la coordinación en vivo de agentes a `ctx.agents`. Prefiere los eventos para la intercepción y la política; prefiere los métodos de servicio para las llamadas directas a capacidades.

Todo registro debería tener un disposer, ya sea devolviendo uno desde `ctx.effect()` o usando un helper de Cordis que lo haga por ti. Si el orden de desmontaje importa, mantén el trabajo relacionado en un único efecto para que la liberación se deshaga en la secuencia prevista.
