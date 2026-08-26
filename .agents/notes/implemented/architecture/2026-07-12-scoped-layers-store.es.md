# Agent Note: Almacenamiento compartido de capas con ámbito

Status: implemented

[English](2026-07-12-scoped-layers-store.md) | [中文](2026-07-12-scoped-layers-store.zh.md) | Español

## Problema

El scoping de agent ([decisión](2026-07-08-agent-scope-contexts.es.md), [diseño de runtime](2026-07-12-agent-scope-runtime-design.es.md)) da a los registros con ámbito la misma forma recurrente: una capa de registro global más una capa exacta por agent. Siete fachadas de registro usan esa forma: `tools.register`, `tools.restrict` y `tools.guard` en `dsh-tools`; `SystemPrompt.section`, `SystemPrompt.tools` y `SystemPrompt.variable` en `dsh-system-prompt`; y `CommandRuntime.register` en `dsh-commands`.

Sin una primitiva compartida, cada fachada repite la coreografía de ciclo de vida en torno a su estado de dominio: derivar la visibilidad del contexto llamante, crear un contenedor con ámbito bajo demanda, adjuntar la propiedad al mismo fiber de Cordis, instalar el undo antes de notificar a los observers, devolver el disposer exacto de Cordis y reclamar el estado con ámbito vacío. Los mapas y tipos de colección separados dejan además a un servicio sin un único objeto que represente la contribución completa de un scope.

El código duplicado arrastra tres requisitos no obvios:

- La visibilidad y la propiedad deben venir del mismo contexto; aceptarlas por separado permite un registro visible en un scope pero dispuesto con otro.
- El undo debe recogerse antes de que se ejecute la callback de cambio, de modo que una callback que lanza revierta la mutación.
- El disposer público debe ser la función exacta devuelta por `ctx.effect()`; envolverlo rompe el teardown ordenado por identidad de Cordis.

La parte compartida es el ciclo de vida y el almacenamiento ordenado por inserción, no la política del registro. Las restricciones de herramientas, el manejo del transporte reservado, el momento de evaluación del prompt, la normalización de comandos, los diagnósticos exactos y el confinamiento de callbacks siguen siendo contratos de dominio distintos.

## Decisión

`@deepseek-ai/dsh-scope` ofrece un módulo de implementación `store.ts` independiente de claves. El paquete sigue siendo peer de Cordis y de `@deepseek-ai/dsh-invariants`, y su acompañante de invariantes permanece sin cambios. La raíz del paquete exporta cuatro símbolos de almacenamiento: `ScopeLayer`, `ScopedLayers`, `NamedEntries` y `AnonymousEntries`. `EntryValues` permanece interno y `store.ts` no es un subpath de paquete.

`ScopeLayer` mantiene explícito el concepto de agregado exigiendo solo el vaciado de la capa completa. Un servicio define una capa concreta cuyas tablas y helpers de dominio encajan con ese servicio; `ScopedLayers` es dueño de la construcción, la selección, el adjunto al ciclo de vida, la notificación y la reclamación del agregado.

## Interfaz pública

```ts ignore-check
export interface ScopeLayer {
  isEmpty(): boolean
}

export class ScopedLayers<L extends ScopeLayer> {
  constructor(
    createLayer: (scope: ScopeKey | undefined) => L,
    onChange: () => void,
  )

  readonly global: L
  peek(scope: ScopeKey | undefined): L | undefined

  merge<V>(
    scope: ScopeKey | undefined,
    pick: (layer: L) => NamedEntries<V>,
  ): Map<string, V>

  effect(
    ctx: Context,
    action: (layer: L) => () => void,
    options: { label: string; notify?: boolean },
  ): () => void
}

export class NamedEntries<V> {
  constructor(duplicateError: (name: string) => Error)
  insert(name: string, value: V): () => void
  get(name: string): V | undefined
  has(name: string): boolean
  keys(): IterableIterator<string>
  entries(): IterableIterator<[string, V]>
  values(): IterableIterator<V>
  isEmpty(): boolean
}

export class AnonymousEntries<V> {
  append(value: V): () => void
  values(): IterableIterator<V>
  isEmpty(): boolean
}
```

## Contrato de almacenamiento

- El constructor crea `global` una vez con `createLayer(undefined)`. Una capa con ámbito solo la crea `effect()`; `peek()` y `merge()` nunca crean una, y `peek(undefined)` devuelve `undefined` porque la capa global ya es explícita.
- `merge()` es la única lectura genérica materializada. Copia las entradas globales con nombre en orden de inserción y aplica después las entradas con ámbito coincidentes en su orden de inserción, de modo que las entradas del mismo nombre se sombrean sin mover nombres no relacionados.
- `NamedEntries.insert()` comprueba e inserta atómicamente, devuelve un undo idempotente de entrada exacta y obtiene el diagnóstico de duplicado exacto del registro de la factory suministrada por el llamador. La búsqueda y los iteradores conservan el orden nativo de `Map` y siguen vivos dentro de una generación de tabla no vacía; vaciar la tabla inicia una generación nueva para que un iterador en vuelo no pueda observar una auto-sustitución.
- `AnonymousEntries.append()` asigna una clave interna única por registro, de modo que callbacks o valores iguales permanecen independientes. Su iterador sigue el orden de inserción y usa el mismo límite de generación viva.
- `effect()` deriva la clave con `scopeOf(ctx)` y adjunta la acción a ese mismo `ctx.effect()`. Acepta una acción síncrona que devuelve un undo síncrono; las acciones deben devolver su undo o lanzar antes de retener una contribución. La helper no normaliza la unión más amplia de `Effect` de Cordis.
- `effect()` recoge el undo de la acción antes de llamar a `onChange` y devuelve el disposer exacto de `ctx.effect()`. La disposición ejecuta el undo de la acción antes de notificar, es idempotente a través de Cordis y elimina una capa con ámbito solo después de que su `ScopeLayer.isEmpty()` completo sea verdadero.
- `options.notify` tiene por defecto `true`. La política propia de la callback sigue siendo autoritativa: las callbacks de cambio de herramienta y prompt pueden lanzar y disparar la reversión del registro; `CommandRuntime.notifyChange()` contiene los fallos de observers; los guards de herramienta pasan `notify: false`.

## Migraciones de registros

`dsh-tools` define una `ToolLayer` que contiene herramientas con nombre más restricciones compiladas anónimas y registros de guard. `ToolRuntime` conserva su resolver de dominio privado para definiciones visibles, nombres conocidos previos a la restricción, nombres globales restringibles, shadowing con ámbito, restricciones e inserción reservada de `run_code`. La evaluación de guards itera en vivo primero los registros globales y luego los con ámbito: las adiciones a una generación no vacía pueden ejecutarse en el despacho actual, mientras que una auto-sustitución tras vaciar la tabla de guards empieza con el siguiente despacho.

`dsh-system-prompt` define una `PromptLayer` que contiene secciones con nombre y variables más providers de herramientas anónimos. El ensamblado fusiona las secciones antes de evaluarlas, de modo que un provider sombreado nunca se llama. La pertenencia al provider de herramientas se materializa una vez por ensamblado. Los providers de variables iteran en vivo primero las tablas globales y luego las con ámbito: las adiciones a una generación no vacía pueden ejecutarse en el ensamblado actual, mientras que una auto-sustitución tras vaciar la tabla de variables empieza con el siguiente ensamblado.

`dsh-commands` define una capa de una sola tabla que contiene `NamedEntries<RegisteredCommand>`. Las vistas efectivas usan `merge()`, mientras que `CommandRuntime` conserva la normalización y congelación de definiciones, los diagnósticos de duplicado exactos, los descriptores inmutables ordenados, la ejecución directa, la limpieza de HMR y los observers de `commands/change` contenidos independientemente.

Las siete fachadas mantienen la validación y los diagnósticos en su registro propietario y siguen devolviendo el disposer exacto de Cordis. La migración no cambia ni el comportamiento público de los registros ni la salida visible para modelos, humanos, wire, persistencia o configuración.

## Alternativas consideradas

**Conservar las implementaciones independientes.** Esto evita una interfaz de biblioteca nueva, pero deja el orden del ciclo de vida, la identidad del disposer y la reclamación del scope duplicados en siete fachadas.

**Una helper por tabla.** Esto elimina algo de código local, pero conserva múltiples mapas por scope y no puede reclamar correctamente la contribución agregada de un scope.

**Instancias de registro por scope.** Los registros hijos necesitarían delegación para las vistas global-más-con-ámbito, sustracción especial para las restricciones y descubrimiento de observers entre instancias. Trasladarían la complejidad en lugar de eliminarla.

**Parámetros de scope explícitos en los métodos de registro.** Entradas separadas de visibilidad y propiedad hacen representables los ciclos de vida no coincidentes, mientras que un scope omitido se convierte silenciosamente en global.

**Aceptar la unión completa de `Effect` de Cordis.** Ninguno de los siete registros tiene setup asíncrono, múltiples undos ni un límite de settlement independiente. La normalización general duplicaría la maquinaria de ciclo de vida de Cordis sin un consumidor actual.

**Exponer `ScopedLayers.values()`, `ScopedLayers.keys()` o un predicado de admisión global.** Esas operaciones codifican políticas de viva/materializada y de filtrado específicas del consumidor. La iteración directa de tablas preserva la semántica viva explícita, `merge()` cubre la operación compartida de shadowing con nombre y `ToolRuntime` conserva su resolver privado más rico.

**Poner `values()` en `ScopeLayer` o exportar `EntryValues`.** Una capa agrega tablas heterogéneas y no tiene un tipo de valor ni una política de iteración coherentes. `EntryValues` solo sirve para compartir detalles de implementación entre las dos clases de tabla; hacerlo público agrandaría la interfaz sin dar a los llamadores una lectura significativa a nivel de capa.

**Generar capas a partir de una descripción de tabla de tipo mapeado.** Las capas concretas de tres tablas y de una tabla son cortas, inspeccionables y libres de contener helpers de dominio. Un generador de clases añadiría un segundo modelo de construcción y una forma de runtime generada para poca ventaja.

## Consecuencias

- Los registros con ámbito expresan una capa agregada y reutilizan la misma coreografía de construcción, propiedad, reversión, notificación y reclamación. La validación, los diagnósticos, el filtrado, la evaluación y la política de observers específicos del dominio permanecen en cada registro.
- La API de lectura pública sigue siendo estrecha: la iteración directa de tablas preserva el comportamiento explícitamente vivo, mientras que `merge()` es la única operación compartida de shadowing materializado. Una `ScopeLayer` heterogénea no tiene contrato de `values()` a nivel de capa.
- La helper es deliberadamente síncrona. Un registro futuro que necesite setup asíncrono o varios undos de propiedad independiente debe identificar sus límites de propiedad y settlement antes de ampliar este contrato.
- Una acción debe lanzar antes de retener una contribución o devolver un undo por todo lo que retuvo; la helper no puede reparar mutaciones fuera de ese contrato. Las operaciones de entrada provistas son atómicas y los registros migrados realizan la validación falible antes de la inserción.
- Una capa con ámbito permanece asignada hasta que cada tabla de su agregado esté vacía. Disponer una fachada no puede por tanto descartar contribuciones hermanas propiedad del mismo scope.
- Los cuatro símbolos públicos se convierten en un contrato de paquete reutilizable. Mantener `EntryValues` interno y la política de consumidores fuera de la helper limita la API de compatibilidad.
- La migración no cambia ningún comportamiento público de los registros ni ninguna salida de modelo, humano, wire, persistencia, configuración o grafo de dependencias.

## Verificación

- Las pruebas unitarias de `dsh-scope` cubren la construcción global, la construcción con ámbito perezosa, las lecturas que no crean, el orden y el shadowing de la fusión con nombre, la reclamación del agregado, la limpieza de fallos de factory y acción, el orden de notificación y la reversión, `notify: false`, las etiquetas de effect, la identidad exacta del disposer, el teardown idempotente, los errores de duplicado propiedad del llamador, los duplicados anónimos independientes, los iteradores vivos y el desacoplamiento de generación drenada.
- Las suites enfocadas de herramienta, system-prompt y comando cubren las restricciones, el manejo del transporte reservado, el acuerdo de nombres conocidos/restringibles, la reentrancia y la auto-sustitución de guards, el orden de validación, los diagnósticos exactos, el sombreado de secciones antes de la evaluación, la pertenencia de instantáneas de providers, la reentrancia y la auto-sustitución de variables, los observers de comando contenidos, las vistas congeladas y ordenadas, la ejecución directa y la disposición del ciclo de vida.
- La comprobación de equivalencia de tipos de core-data con ámbito une la documentación de `ScopeLayer` con su declaración fuente. La documentación del repositorio, el grafo de módulos, el build, la higiene, la cobertura y las compuertas de artefactos construidos ejercitan la exportación raíz y el límite del paquete.
- Las instantáneas sin clave existentes de ACP, headless y TUI siguen siendo el límite de regresión para los schemas de herramienta y el ensamblado de prompt; la cobertura de TUI es dueña de los comandos humanos. La implementación no actualiza ningún transcript esperado.
