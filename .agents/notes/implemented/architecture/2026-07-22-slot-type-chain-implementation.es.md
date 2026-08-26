# Agent Note: El estándar del sistema de slots — un único register, cuatro cuotas de props y el asiento del store del framework

Status: implemented

[English](2026-07-22-slot-type-chain-implementation.md) | Español

> Alcance: el diseño definitivo del sistema de slots para el cliente web — cómo los plugins de UI componen la página, dónde vive la autoridad de render, cómo se tipan las props de los componentes y dónde van los datos vivos de negocio. El [RFC de arquitectura del cliente web](2026-07-19-gui-web-client-architecture.es.md) es dueño del contexto circundante (cadena de carga, capa de objetos, servicios) y difiere aquí sus secciones de slots.

## Problema

La página se compone en runtime a partir de plugins cargados de forma independiente, de modo que la UI necesita un mecanismo de composición que responda a cuatro preguntas con fuerza estática. ¿Quién puede renderizar en una región — y esa autoridad es exigible, o meramente convencional? ¿Cómo recibe un componente todo lo que necesita sin dejar de ser una función pura (sin ctx, sin imports del framework), sin que cada valor tenga que pasarse a mano a través del código de ensamblaje? ¿Dónde viven los datos vivos de negocio para que las actualizaciones en streaming re-rendericen precisamente a los suscriptores — sin que cada plugin construya su propia maquinaria de suscripción? ¿Y cuánto de esto puede comprobar el compilador, de modo que un componente desviado, una llamada de render que excede su alcance o un schema de store no coincidente sea un error de compilación en un único punto de llamada visible en lugar de una sorpresa en runtime?

## Decisión

En una frase: **el ui-renderer renderiza solo `'root'`; un plugin compone la UI mediante una única llamada `register` que simultáneamente ocupa un slot, declara+autoriza sus slots hijos, declara su store e inyecta su cara de negocio; los componentes son funciones puras cuyas props llegan en cuatro cuotas, cada una derivada automáticamente de su única fuente de verdad.**

### `'root'` es el único slot a priori

`SlotRegistry` (runtime del cliente) declara `'root'` en la construcción — single/root, `owner: {}` — y su merge de `SlotMap` vive en el paquete de runtime. El ensamblaje completo del ui-renderer es `ctx.slots.renderSlot('root', {})`: el único punto de entrada de render a nivel de ctx; cualquier otra clave, un renderizador ausente o una root no registrada falla estrepitosamente (sin fallback).

### register es la única API; children = declaración + autorización + especificación de runtime

```ts ignore-check
ctx.slots.register({
  name: 'root',
  children: {
    'sidebar':      { kind: 'single', scope: 'root' },
    'conversation': { kind: 'single', scope: 'session' },
  },
  store: createLayoutStore,      // StoreHandle or factory (below)
  inject: injectFrame,           // business face (below)
}, AppFrame)
```

No hay una API separada de definición de slots. El objeto `children` a la vez **declara la existencia de los slots hijos** y **autoriza a este componente a renderizarlos** — un slot es un hueco en el árbol de render que existe porque alguien lo va a renderizar, de modo que su ciclo de vida es el de la entrada declarante (entrada eliminada → slots desaparecidos, contribuciones limpiadas). Los valores son la especificación de runtime (`kind`/`scope` guían la iteración de outlet y la selección de vinculación; `SlotMap` es solo de tipos y se borra en runtime, que es por lo que un array de claves no podría funcionar), comprobados estáticamente contra la entrada de `SlotMap` para que tipo y valor se declaren en un único punto y se validen de forma cruzada.

Regla de paridad: **la entrada declarante tiene el derecho exclusivo de renderizar sus slots hijos**, decidido enteramente en tiempo de register (una configuración incorrecta falla estrepitosamente en la carga; la ruta caliente de render no lleva comprobaciones). Casos que fallan estrepitosamente en la carga: una segunda entrada que declara un slot ya declarado; registrarse en un slot no declarado; un mismo handle de store montado bajo dos ámbitos; un registro de cadena sin su `select`.

Un contribuidor cuyo orden de activación sea independiente de la entrada declarante usa `ctx.slots.inject(key, callback)` y conserva el fail-loud directo de `register()`. Las vidas de declaración, contribuidor, reemplazo y fallo las especifica la [decisión de inyección de declaración de slots](2026-08-05-slot-declaration-injection.es.md).

La fusión de declaraciones de `SlotMap` sigue siendo la autoridad de tipos, y una entrada declara solo sus propios ejes más la **cuota del propietario** — las props inyectadas de quien registra nunca entran en la tabla global («quien inyecta algo, es dueño de su tipo»).

### Props de componente: cuatro cuotas, cada una de su propia fuente de verdad

| Cuota | Tipo | Fuente de verdad | Contenido |
|---|---|---|---|
| runtime | `PropsRuntime<K>` | entrada de SlotMap para K | `OwnerOf<K>` (parámetros del punto de render) + `useSession`/`sessionId` estándar de ámbito de sesión + `useSessions`/`useWorkspaces` globales |
| render de hijos | `PropsRenderSlots<S>` | claves de `children` del register | `renderSlot(key, owner)`, con key restringida estáticamente a S; las claves de cadena añaden `renderSlotChain` |
| store | `PropsStore<H>` | tipo de retorno de la fábrica de store | hook selector `useStore` + `actions.*` (sin el parámetro draft) |
| negocio | `I` | tipo de retorno de inject | datos planos + callbacks; un compartimento reservado `hooks` de observables desnudos llega enlazado como hooks selector `use<Name>` (`InjectFace<I>`) |

`sessionId` lo suministra el framework siempre que se declara `scope: 'session'` — los parámetros del propietario no lo llevan. El punto de llamada de register es el punto de estrangulamiento de doble bloqueo: un componente cuyas claves de renderSlot excedan la declaración de `children`, o que omita una cara declarada, o cuyas formas de store/inject se desvíen, es un error de compilación en esa línea. La delegación es paso ordinario de props (se entrega la función `renderSlot` hacia abajo, opcionalmente tras una firma más estrecha) — no hay objeto de cara de lista blanca ni API de acuñación.

### El tipo cadena: las entradas se autonominan, la primera coincidencia renderiza

El cuarto `SlotKind`, `'chain'`, invierte la autoridad de enrutamiento respecto a `keyed`: un punto de despacho keyed elige a su ocupante por `entryKey`, mientras que una entrada de cadena se autonomina — el propietario despacha una única moneda común de props del propietario y nunca llega a saber quién toma el relevo, de modo que un paquete de relevo nuevo se registra con cero ediciones del propietario. Un registro de cadena lleva un selector puro `select` (`ChainSelect<O, M>`: `(owner) => matched | null`) y una `priority` opcional (ascendente; los empates conservan el orden de registro = orden de ensamblaje — la topología de inject controlable en el despliegue — bajo la misma ordenación estable que el `order` de la lista); registrarse sin `select` es uno de los casos que fallan estrepitosamente en la carga de arriba. En el render, el outlet ejecuta los selectores en orden de cadena: el primer retorno no nulo elige a su entrada y el valor devuelto se une a las props del componente como `matched` (el componente nunca re-deriva su propia coincidencia), `null` cede el turno a la entrada siguiente, y todos-nulos renderiza el cuerpo de fallback del propietario (`ChainRenderOpts`).

La decisión de declinar vive en `select`, nunca en un componente montado que inspecciona sus propias props: un componente que se monta solo para renderizar null ejecuta igualmente sus hooks y efectos para nada, y la fricción resultante de montaje/desmontaje rompe la memoización y la semántica de claves de React, mientras que un selector es una función pura — comprobable mediante tests unitarios, cero efectos secundarios de montaje — la misma disciplina que «los métodos de presentación son funciones puras de `args`». La pureza es el contrato del selector: no lee estado mutable externo ni produce efectos secundarios, de modo que la decisión de enrutamiento es enteramente función de las props del propietario y segura de ejecutar en cada despacho. Los selectores enrutan; nunca acuñan — la construcción de objetos por despacho haría variar la identidad en cada render, así que envolver un valor coincidente en una cara más rica ocurre dentro del componente elegido (`useMemo` indexado por `matched`).

En la cadena de tipos, la forma de SlotMap de una entrada de cadena es `{ kind: 'chain'; scope; owner }` con `owner` como la moneda de la cadena; `M` — el tipo de la prop `matched` — se infiere del retorno de `select` (un selector que estrecha un miembro de unión tipa `matched` automáticamente), y la posición del componente se mantiene fuera de la inferencia de `M`, la misma resolución NoInfer que fija la cuota de inject (resoluciones abajo). En el lado del propietario, `renderSlotChain(key, owner, { fallback })` se une a `renderSlot` en la cuota `PropsRenderSlots`, con su dominio de claves restringido estáticamente a las claves de tipo cadena de la declaración de children de la entrada (`ChainKeysOf`); el punto de despacho es una línea y no alberga derivación ni lógica de enrutamiento propia.

### El asiento del store: motor del framework, schema de quien registra

El framework es dueño de exactamente una máquina de suscripción: el motor de store de instantáneas (zustand vanilla + immer + persistencia opcional en localStorage) vive en el **paquete de runtime** (entrada principal `./client` — sin subruta), produciendo fuentes observables crudas; ui-renderer las vincula en hooks en el outlet (vinculación uSES en caché por fuente). Lo que un store *contiene* es la declaración de quien registra, escrita como fábrica para que no exista ningún handle a nivel de módulo (un handle con ámbito de módulo sería un singleton de facto que sobreviviría a las recargas de plugin):

```ts ignore-check
export function createChatStore() {
  return defineStore({
    init: () => ({ selection: null as SelectionTarget | null, draft: '' }),
    persist: 'dsh.conversation.chat',
    actions: {
      select:    (d, t: SelectionTarget) => { d.selection = t },
      clearDraft:(d) => { d.draft = '' },
    },
  })
}
```

Una fábrica, tres puntos de consumo: (a) `register` — pasa la fábrica para un store exclusivo, o la llama una vez en `apply` y pasa el mismo handle a varios registers para compartir la instancia (compartir entre plugins es constructivamente imposible: el handle nunca sale del paquete); (b) `PropsStore<ReturnType<typeof createChatStore>>` deriva la cuota de store del componente con cero miembros escritos a mano; (c) los tests llaman a la fábrica y `.create()` una instancia real del motor, alimentando `useSelector`/`actions` directamente como props — los outlets de producción ejecutan exactamente la misma ruta `create`, así que no hay una segunda maquinaria.

El ámbito del store se **deriva del ámbito de la entrada que lo monta** (slot de sesión → una instancia por sesión, que vive y muere con la sesión; slot root → una por entrada). Lectura = `props.useStore`; escritura = solo `props.actions.*` — la instancia cruda (con `update`/`set`) nunca llega a un componente, de modo que las acciones declaradas son la API de mutación completa y auditable. El código de producción nunca llama a la fábrica ni a `create` fuera de `apply`.

### inject: la cara de negocio de quien registra, en su propio ctx

Una fábrica de inject toma lo que sus declaraciones le ganan — `sessionId` para los slots de sesión, `actions` enlazadas cuando se declara un store, nada en otro caso — y lee los servicios a través del **ctx propio del cierre de apply**, de modo que su frontera de capacidad es la topología de inject declarada del plugin (el proxy de propiedades de cordis se aplica de forma nativa; no hay ningún handle de ensamblaje que porte un ctx más amplio). Su valor de retorno son datos planos y callbacks, más a lo sumo el compartimento reservado `hooks`: un mapa de fuentes observables desnudas (getSnapshot+subscribe) que el renderizador vincula en hooks selector `use<Name>` antes de que la cara llegue al componente — el gemelo privado de quien registra del compartimento de hooks del canal provide, para hechos reactivos demasiado específicos para el kit estándar global (aviso del composer/lexicón, las filas de navegación de ajustes). Los componentes nunca reciben las fuentes crudas, de modo que el código de negocio sigue sin contener maquinaria de suscripción. Todo lo demás permanece plano: la cara de lectura/escritura estrechada de los propios servicios del plugin, la orquestación entre servicios (p. ej. `send` = `actions.clearDraft()` + `ctx.conversation.send(...)`), y los efectos secundarios de ensamblaje por (entrada×sesión). Sin hooks hechos a mano, sin productores de ReactNode, sin objetos de servicio completos — el estrechamiento es el valor: lo que un componente puede hacer es exactamente la forma de retorno de la fábrica.

### Disciplina de fronteras de datos

Los hooks los hace solo el framework: `useSession`, `useSessions`, `useWorkspaces`, `useStore`, `renderSlot` más los hooks enlazados de las contribuciones de provide y los compartimentos `hooks` de inject — todos sintetizados por la única maquinaria de vinculación del renderizador; el código de negocio pasa datos planos y callbacks entre padre e hijo (los hooks de comportamiento propios de un componente que no se suscriben a nada externo siguen siendo correctos). Los datos vivos tienen exactamente tres canales: lo que el padre sabe viaja como props del propietario en el punto de renderSlot; lo que solo el componente sabe es estado local; lo que debe compartirse entre entradas o sobrevivir a los remontajes es un store declarado. La derivación es una función pura sobre datos de hooks del framework (`useMemo`), nunca una suscripción propia.

### Contexto de árbol y el contrato del renderizador

`SessionProvider` es un componente del framework **entregado como asiento del kit estándar**: una entrada cuyos `children` declaren un slot de ámbito de sesión lo recibe como prop (tipo en ui-slots, valor inyectado por el renderizador) — los componentes nunca importan su valor. Está autocableado (lee internamente el estado de sesión actual del runtime; el ensamblador no le pasa nada), con forma de render prop — `children(sessionId)` con una rama `empty`, remontado bajo `key={sessionId}`. `BindingContext` es interno a la maquinaria; los componentes de negocio no ven ningún contexto de React. Las fábricas de inject se ejecutan dentro del outlet a propósito (los límites de error por entrada los capturan; un registrante que falla apaga solo su propia entrada mientras que los errores de ensamblaje se relanzan); el outlet lee el contexto de árbol como parámetro implícito solo de la maquinaria — la división «la identidad, del cierre de register; la situación, de la posición en el árbol».

El render vive detrás de un contrato de instalación para que el runtime siga libre de React: `SlotRenderer` (interfaz en ui-slots, implementación `createSlotRenderer()` en ui-renderer) se instala una vez en el arranque del shell mediante `ctx.slots.install(...)`; la doble instalación y el render antes de instalar lanzan una excepción. La contabilidad de propiedad es un único `Map<key, entry>` en el servicio — el ledger, los slots, las contribuciones, las vinculaciones de render y las instancias de store viven y mueren en el único eje de la entrada, lo que cierra la ventana de autoridad obsoleta a través de las recargas de plugin por construcción (el `renderSlot` capturado de una entrada eliminada lanza un error de autorización obsoleta al entrar).

### Resoluciones de implementación de la cadena de tipos

Hay dos decisiones de endurecimiento en la firma de register porque la alternativa obvia falla de una forma específica y reproducible; un editor futuro no debería volver a discutirlas:

1. **`SlotComponent<P>` (firma de llamada simple) en lugar de `FC<P>` en la posición de registro.** `FC` de React lleva campos estáticos (`propTypes`, `defaultProps`) cuyos tipos referencian `P` en posiciones covariantes; la asignabilidad entre dos instanciaciones de `FC` comprueba también esos estáticos y rechaza componentes que el diseño quiere aceptar. La firma de llamada simple comprueba solo mediante contravarianza limpia de parámetros; los componentes siguen siendo funciones ordinarias.
2. **`NoInfer<I>` fija la inferencia de la cuota de negocio a la fábrica de inject.** Sin él, TS también recoge candidatos de inferencia de la posición del parámetro del componente, y un componente desviado (que consume una clave que la fábrica no suministra) ensancha `I` en silencio para que la llamada compile — absorbiendo exactamente la desviación que la cadena existe para detectar. La especificación de muestra negativa lo fija: si el `NoInfer` se «simplificara y eliminara» alguna vez, el punto de expect-error se pone en rojo primero.

## Consecuencias

La autoridad de render es exigible en lugar de convencional: quién renderiza qué es un hecho de tiempo de carga, y auditar la estructura de la UI = leer las llamadas de register; para los slots de cadena, QUIÉN renderiza es además un hecho de tiempo de render, pero los selectores que lo deciden son declaraciones en el punto de register, así que el alcance de la auditoría sigue siendo las llamadas de register. Cada API de props se deriva estáticamente de una sola fuente (entrada de SlotMap, claves de children, fábrica de store, retorno de inject), de modo que un cambio de schema se propaga por el compilador en lugar de por grep. Los plugins no llevan maquinaria de suscripción propia — el ciclo de vida del store (instancias por sesión, eliminación, persistencia) es semántica del framework anclada al eje de la entrada. Costes: las opciones de registro son densas (objetos de especificación de children); el framework carga con maquinaria de inferencia real (la inferencia en la misma ronda de init/actions de `defineStore` puede necesitar un fallback currificado); y los dobles bloqueos en tiempo de compilación hacen que la desviación en fase de prototipo sea un error duro, no una advertencia.

## Alternativas consideradas

| Rechazada | Motivo en una línea |
|---|---|
| API separada en dos pasos define/register | La división deja la autoridad de render sin exigir e invita a bugs de orden; children-en-register resuelve declaración, autorización y especificación en un único lugar visible |
| Objetos de cara de lista blanca (`ScopedSlots` + helpers de estrechamiento) | Con la lista blanca ya en el tipo de props del componente, la cara es derivable por maquinaria; un objeto de cara acuñable es una tercera API de autoridad con comprobaciones solo de runtime |
| Handles de ensamblaje que llevan el ctx raíz a inject | Elude la topología de inject declarada — cada fábrica podría alcanzar cualquier servicio, de modo que las declaraciones de dependencia de package.json dejarían de significar algo |
| `children` como array de claves | kind/scope son datos de despacho de runtime; SlotMap se borra, así que un array fuerza una segunda API de registro de especificaciones — una API de definición renacida |
| Hooks de negocio hechos a mano / observables crudos en las props de componente | Cada plugin se convierte en su propia máquina de suscripción; el compartimento `hooks` de inject lleva los mismos hechos a través de la única maquinaria de vinculación auditada |
| Handles de store a nivel de módulo | Un handle con ámbito de módulo es un singleton entre recargas de plugin y casos de test; la forma de fábrica limita la identidad a la invocación de apply/test |
| Componentes que reciben la instancia del store | `update`/`set` en código de render hace inauditable la API de mutación; las acciones declaradas mantienen «lo que puede cambiar» como un hecho del punto de register |
| `FC` en la posición de register / inferir `I` desde el componente | Los estáticos de FC generan ruido covariante que rechaza componentes válidos; la inferencia del lado del componente absorbe la deriva de props en silencio (ver resoluciones arriba) |
| Despacho keyed con enrutamiento del lado del propietario para slots de relevo | El propietario acumula contratos por entrada y una tabla de enrutamiento hardcodeada (`find` + `entryKey` por relevo); la moneda de cadena mantiene los nuevos registros de relevo en cero ediciones del propietario |
| Componentes que declinan renderizando null | Declinar exige montar primero — los hooks y efectos se ejecutan para nada, y la fricción de montaje/desmontaje rompe la memoización y la semántica de claves; un selector puro decide sin instancia de componente |
