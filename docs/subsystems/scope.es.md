# Registro con alcance

[English](scope.md) | Español

El [paquete scope](../../packages/core/scope) aporta el vocabulario de identidad, portador y capas con alcance que hace que un contexto de registro signifique a la vez visibilidad por agent y propiedad compartida del ciclo de vida. Es una primitiva de biblioteca, no un servicio de Cordis; la [Agent Note de diseño en tiempo de ejecución del agent-scope](../../.agents/notes/implemented/architecture/2026-07-12-agent-scope-runtime-design.es.md#scope-routing-one-opaque-key-selects-one-layer) es la dueña del razonamiento del ciclo de vida, la [Agent Note de almacenamiento compartido](../../.agents/notes/implemented/architecture/2026-07-12-scoped-layers-store.es.md) es la dueña de la decisión de capas de registro, y el [README](../../packages/core/scope/README.es.md) del paquete es el dueño de la API invocable y de la semántica de filtrado.

Sources: [`packages/core/scope/src/index.ts`](../../packages/core/scope/src/index.ts) and [`packages/core/scope/src/store.ts`](../../packages/core/scope/src/store.ts).

## Identidad y portador de despacho

`ScopeKey` es una identidad de objeto opaca. El loop incluido usa el objeto `Agent` en vivo como su propia clave, pero la primitiva nunca inspecciona el objeto.

```ts type-equiv
/** An opaque, identity-compared scope key. */
type ScopeKey = object
```

`Scoped<T>` es la marca en tiempo de compilación sobre el receptor de enrutamiento opaco que devuelve `scopeTarget(base, key)`. Las declaraciones de eventos filtrados por alcance exigen este portador como su tipo `this`, mientras que el sujeto real del evento sigue siendo un argumento explícito.

```ts type-equiv
/**
 * A routing-only event receiver built by {@link scopeTarget}. The type
 * parameter records the subject type for dispatch checking; the carrier does
 * not expose the subject's properties. Event payloads carry the real subject.
 */
type Scoped<T extends object> = object & { readonly [ScopedBrand]: T }
```

## Contexto de registro con dueño

`Scope` empareja el contexto de registro etiquetado con dos vías de teardown. `rawDispose` conserva la identidad exacta del disposer de Cordis que necesita un efecto compuesto ordenado; `dispose()` es el límite público compartido de quiescencia para quienes llaman directamente y en carrera.

```ts type-equiv
/** A minted registration scope and its quiescent disposal boundaries. */
interface Scope {
  /** Context through which scope-owned registrations are made. */
  ctx: Context
  /** Exact Cordis disposer, used when nesting this scope in an ordered composite effect. */
  rawDispose: () => Promise<void> | void
  /** Dispose every scope-owned registration; racing calls await the same completion. */
  dispose(): Promise<void>
}
```

## Capa de registro con alcance

`ScopeLayer` representa la contribución completa de un registro a nivel global o de alcance exacto. Una capa concreta puede agregar varias tablas con nombre y anónimas; la vaciedad de la capa completa permite a `ScopedLayers` reclamar el estado con alcance sin descartar una tabla hermana.

```ts type-equiv
/** One scope's aggregate contribution to a registry. */
interface ScopeLayer {
  /** Whether every table in this layer is empty. */
  isEmpty(): boolean
}
```

`ScopedLayers<L>` es el dueño de la capa global ansiosa y de las capas de alcance exacto creadas de forma perezosa. Las lecturas no crean capas: `peek(undefined)` significa que no hay overlay, mientras que `merge()` materializa las entradas globales con nombre en orden de inserción seguidas de las sombras con alcance. Los registros usan un único contexto tanto para la visibilidad como para la propiedad de efectos de Cordis, recogen un único undo síncrono antes de la notificación opcional, devuelven el disposer exacto de Cordis y reclaman una capa con alcance solo cuando su `ScopeLayer` completo está vacío.

`NamedEntries<V>` aporta búsqueda en orden de inserción e iteración en vivo con errores de duplicados propiedad del que llama. `AnonymousEntries<V>` da a cada añadido una identidad única para que los valores iguales sigan siendo independientes. La iteración permanece en vivo dentro de una generación de tabla no vacía; vaciar la tabla desvincula los iteradores existentes de las inserciones posteriores. Ambos devuelven undos idempotentes de entrada exacta; la interfaz de implementación compartida `EntryValues` no es pública.
