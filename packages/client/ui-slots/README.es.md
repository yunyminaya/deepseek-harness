# @deepseek-ai/dsh-client-ui-slots

[English](README.md) | Español

Núcleo puro del registro de slots, diseño terminal de slots: la fusión de declaraciones de SlotMap, la única API de composición `register` en SlotCore, la familia de tipos de props de componente de cuatro partes, la familia de tipos de asiento de store y el contrato de instalación del renderizador. Solo tipos de React en runtime — el paquete está libre de React y libre de cordis.

Una llamada `register({ name, children?, store?, inject?, ...kind }, Component)` aporta un componente a un slot declarado y, en el mismo acto, declara slots hijos (declaración = autorización de renderizado = spec de runtime, una sola tabla), un asiento de store y la cara de negocio del registrante. El componente se comprueba en el punto de llamada contra `ComposedProps` — la intersección de cuatro partes, cada una derivada de su única fuente de verdad:

| parte | tipo | fuente |
|---|---|---|
| runtime | `PropsRuntime<K>` | entrada de SlotMap: `owner` (punto de llamada renderSlot del padre) + kit estándar de sesión + asiento global |
| render de hijo | `PropsRenderSlots<S>` | el conjunto de claves `children` de la llamada register (`renderSlot` restringido estáticamente) |
| store | `PropsStore<H>` | el handle declarado: hook selector `useStore` + `actions` sin el parámetro de borrador |
| negocio | `I` | inferida del retorno de la fábrica `inject` |

Los slots de tipo cadena invierten el enrutado con clave — las entradas se autonominan en lugar de que el sitio de despacho elija una `entryKey`: cada registro lleva un selector `ChainSelect` puro (más una `priority` ascendente opcional, empates en orden de registro), el primer retorno no nulo elige su entrada y se convierte en la prop `matched` del componente, y todo-nulo cae al fallback `renderSlotChain` del propietario (`ChainRenderOpts`).

Las interfaces del kit estándar (`SessionStandardProps`, `GlobalStandardProps`) se declaran vacías aquí y las fusiona el paquete runtime (el mismo patrón declare-merge que las claves de SlotMap). El renderizador enlaza las fuentes observables de sesión y workspace del runtime en hooks selectores. Los parámetros de la fábrica inject derivan de la declaración (`InjectParams`): los slots de sesión reciben `sessionId`, un store declarado añade `actions` horneadas, nada más — el acceso a datos vive en el ctx del cierre de apply.

La familia de store (spec de `defineStore` de entrada / `StoreHandle<T, A>` de salida) tipa el asiento de store: `init` infiere el schema de estado, `actions` es el conjunto completo de escrituras de transformación de borrador, `BakedActions` elimina el parámetro de borrador en los callbacks que reciben los componentes y las fábricas inject. La implementación de valor de `defineStore` vive en el paquete runtime (el hogar del motor) y satisface el contrato `DefineStore` exportado aquí. Los productos del motor y el contrato de host del renderizador llevan fuentes de instantánea desnudas (`getSnapshot`/`subscribe`), nunca hooks de React — el enlace de hooks pertenece a la maquinaria de renderizado; aquí solo vive el tipo de hook del contrato de props (`SnapshotSelectorHook`).

`SlotCore` siembra el slot a priori `'root'` en la construcción y aplica la validación en tiempo de carga (registro de slot no declarado, declaración duplicada de hijo, un mismo handle compartido bajo dos alcances, un registro de cadena sin `select` — todo lanza en register). El liberador de una entrada colapsa sus slots hijos declarados recursivamente: las filas del ledger, las contribuciones y los montajes de store mueren en un mismo eje de ciclo de vida. Cada clave lleva también una época de declaración que solo avanza en la declaración y en el colapso; el runtime la usa para [`ctx.slots.inject`](../runtime/README.es.md#slot-declaration-injection), independientemente de las versiones ordinarias de entrada. `renderer.ts` lleva el contrato de instalación (`SlotRenderer`, `SlotRendererHost`) más `StaleAuthorizationError`/`SlotOwnershipError`; ui-renderer es el dueño tanto de la implementación como de su instalación de ciclo de vida de plugin.

## Model Experience

Ninguno: el registro de slots es fontanería de UI del lado del navegador; nada de esto llega a una petición de modelo.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo pendiente

- **`isLive` explora todos los registros linealmente** — suficiente con recuentos de registro de plugins de UI (decenas); revisar con una referencia inversa entrada→registro si los ledgers llegaran a calentarse.
- **El ancla fantasma `__renders` es visible en `PropsRenderSlots`** — el mismo ruido aceptado que el `__accepts` del diseño de cadena de tipos: las firmas de métodos genéricos se comparan con holgura entre uniones de claves, así que el marcador contravariante es lo que hace cumplir «conjunto de claves de componente ⊆ declaración de children».
