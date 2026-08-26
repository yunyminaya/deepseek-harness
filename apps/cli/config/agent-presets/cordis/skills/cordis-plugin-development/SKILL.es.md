---
name: cordis-plugin-development
description: Crea, modifica, depura o amplía Plugins dinámicos de Cordis, incluidos los Services y Events del Host, los Slots del Client y la UI de temas, las llamadas privadas de paquete del Client al Host, las Tools dinámicas, las actualizaciones de versión, los fallos de aprobación y los diagnósticos de runtime. Usa esta skill para enrutar una petición del usuario a la plataforma y al Inspect Provider correctos y, a continuación, definir, ejecutar, reparar o revertir el Plugin.
---

# Desarrolla Plugins dinámicos de Cordis

[English](SKILL.md) | Español

Primero determina si una capacidad pertenece al Host o al Client y, después, consulta la interfaz real antes de escribir código. Nunca infieras una API completa a partir de un nombre de Service, un payload de Event, unas props de Slot, un token de tema o un ejemplo.

## Flujo de trabajo estándar

1. Llama a `cordis_inspect_list` para obtener los Providers, métodos y schemas registrados actualmente en Host y Client.
2. Selecciona el conjunto más pequeño de llamadas a `cordis_inspect_query` necesario para leer los Services, Events, Builtins, Slots, tokens de tema o Tools exactos que usará la implementación.
3. Para un Plugin nuevo, diseña su primer Package. Para modificar un Plugin existente, usa primero `cordis_inspect_self(pluginId, packageId)` para leer el código fuente base y los diagnósticos.
4. Escribe JavaScript plano en `code.host`, `code.client` o en ambos, y luego llama a `cordis_define`.
5. Llama a `cordis_run` con el `pluginId` y el `packageId` finales que devuelve define.
6. Gestiona la aprobación, la espera, la carga del Client y los fallos de renderizado desde la tarjeta Run, los mensajes de steering o `cordis_inspect_self`.
7. Usa `cordis_stop` para deshabilitar el Plugin temporalmente. Usa `cordis_undefine` solo cuando ya no sea necesario.

No esperes en el mismo turno la aprobación del usuario ni los resultados asíncronos del navegador. Después de que `cordis_run` devuelva `awaiting-approval` o `starting`, termina el flujo de Tool actual y espera a que el sistema informe del resultado final mediante las actualizaciones de estado y el steering.

## Guía de uso de las Tools

| Tool | Úsala cuando | No hacer |
| --- | --- | --- |
| `cordis_inspect_list` | Descubrir los Providers actuales de Host/Client y los schemas de métodos en una sola llamada; actualízalos cuando cambie el directorio de capacidades del runtime | Fijar los nombres de los Providers en el código y saltarte la lista; tratar un manifest como datos de negocio |
| `cordis_inspect_query` | Confirmar los métodos exactos de los Services, los modos de los Events, los Builtins, los Slots, los tokens o los schemas de las Tools antes de escribir código | Usarla en lugar de llamar a un Service real desde el Plugin; suponer que una consulta del Client terminará sin una página que responda |
| `cordis_inspect_self` | Listar los Plugins actuales, inspeccionar los punteros de versión o leer el código fuente exacto de un Package y los diagnósticos de runtime | Traer todo el código fuente solo para construir una lista; usarla para modificar o arrancar un Plugin |
| `cordis_define` | Crear la primera versión de un Plugin o añadir un Package inmutable a un Plugin existente; deja que el usuario previsualice el código primero | Esperar que define ejecute `apply`, solicite aprobación o actualice current |
| `cordis_run` | Activar un Package exacto; usa `run` para la primera activación, el reinicio o el rollback, y `update` para cambiar de versión | Usar `run` para cambiar de versión implícitamente; tratar pending o starting como éxito |
| `cordis_stop` | Pausar los efectos actuales conservando los Packages, los grants y los punteros de versión para un uso posterior | Usar stop para significar una eliminación permanente |
| `cordis_undefine` | Eliminar permanentemente un Plugin y todos sus Packages y borrar las vistas de negocio históricas | Llamarla cuando todavía se necesite rollback, inspección o reinicio |

## Elige una plataforma

| Requisito | Plataforma preferida | Inspecciona primero |
| --- | --- | --- |
| Archivos, comandos, procesos o red | Host | `fs`, `bash`, `subprocess`, `pty` y `web` en `Service.listService` |
| Agents, datos de Session durables o ciclo de vida del Host | Host | El Service relevante y `Event.listEvents` |
| Registrar una Tool dinámica invocable en el siguiente paso del modelo | Host | `harness` en `Builtin.listBuiltins`, más `Tool.listTools` |
| Tema de la página, diseño o estado actual de la página | Client | `Theme.listTokens` y `Service.listService` del Client |
| Conversation Snapshot o listas de sesión/workspace | Client | Las props estándar y las props de propietario del Slot objetivo |
| Páginas de ajustes, barras laterales, áreas de entrada, overlays o tarjetas de Tool | Client | `Slots.listSubTree` |
| Traer datos en el Host y mostrarlos en el Client | Ambas | Service del Host + `harness.handle`; Slot del Client + `host.call` |

Prefiere la capacidad más cercana al propietario de los datos. Si las props del Slot ya proporcionan la Conversation Snapshot, no la vuelvas a traer a través del Host. Si solo necesitan cambiar los estilos propios del Package, no sobrescribas el tema global. Si solo se necesita un punto de entrada pequeño, no reemplaces una región completa de la UI del producto.

## Navegación por Providers

Selecciona los métodos del resultado real de `cordis_inspect_list`. Los métodos iniciales habituales incluyen:

- `Service.listService`: sin `service`, devuelve todos los Services invocables con su propósito y las firmas exactas de sus métodos. Vuelve a consultar el `service` seleccionado para obtener las reglas de acceso, las descripciones/parámetros/retornos estructurados de los métodos y solo sus tipos referenciados.
- `Event.listEvents`: sin `event`, devuelve todos los Events con su propósito, modo de despacho y firma exacta del listener. Vuelve a consultar el `event` seleccionado para obtener su contrato de listener estructurado y solo sus tipos referenciados; un listener Waterfall debe llamar a `next()`.
- `Builtin.listBuiltins`: devuelve los símbolos y las firmas proporcionados por el evaluador que no se pueden obtener mediante `ctx.get()`.
- `Slots.listSubTree`: sin `root`, devuelve árboles en vivo compactos con el propósito, el tipo, el ámbito, las claves de registro, el riesgo de reemplazo y los hijos de cada Slot. Con un `root` exacto, devuelve también el contrato completo, las props y los ocupantes actuales del Slot seleccionado, manteniendo compactos los descendientes.
- `Theme.listTokens`: devuelve los tokens de tema que actualmente se pueden consultar y sobrescribir; no modifica el tema.
- `Tool.listTools`: devuelve los schemas de las Tools realmente visibles para el Agent actual, incluidas las Tools registradas dinámicamente.

Los nombres de los Providers, los métodos y las entradas deben proceder del resultado de la lista actual. El catálogo Service/Event describe qué interfaces permite esta versión; no garantiza que un Service esté montado actualmente. En runtime, usa Services y Events reales en lugar de cachear o mostrar los resultados de consulta del catálogo.

## Entorno de ejecución

Tanto `code.host` como `code.client` son cuerpos de función de JavaScript plano que devuelven un Plugin de Cordis. No los compila TypeScript, JSX ni un bundler.

No uses:

- `import`, `require`, tipos de TypeScript, `as`, decoradores o JSX;
- globales no confirmados por `Builtin.listBuiltins`;
- acceso adivinado a `window`, `document`, `process`, `Buffer`, `fetch` o temporizadores nativos.

El código React del Client debe usar `React.createElement(...)`.

Correcto:

```js
return {
  apply(ctx) {
    const slots = ctx.get('slots')
    if (slots === undefined) return
    slots.inject('tool.view.cordis', () => slots.register(
      { name: 'tool.view.cordis', key: 'self' },
      () => React.createElement('div', null, 'Hello'),
    ))
  },
}
```

Incorrecto:

```jsx
return {
  apply(ctx) {
    return <div>Hello</div>
  },
}
```

JSX no es el único problema de este ejemplo. `apply()` registra contribuciones del ciclo de vida y no puede devolver un React Element como resultado del Plugin. La UI debe registrarse en un Slot consultado.

## Acceso a Services

Lee las capacidades opcionales con `ctx.get(name)` por defecto y gestiona su ausencia:

```js
return {
  apply(ctx) {
    const service = ctx.get('serviceName')
    if (service === undefined) return
    service.someMethod()
  },
}
```

Declara `inject` solo cuando un Service sea una dependencia dura y el Plugin deba entrar en espera hasta que Cordis lo reactive después de que aparezca el Service:

```js
return {
  inject: ['requiredService'],
  apply(ctx) {
    ctx.requiredService.someMethod()
  },
}
```

No abuses de `inject` solo para evitar una comprobación de `undefined`. No accedas a `ctx.requiredService` sin declarar la inyección; el Guard rechaza las dependencias no declaradas.

## Gestiona los efectos secundarios

Toda contribución debe eliminarse después de detener, actualizar o eliminar el Plugin. Prefiere las APIs de ciclo de vida de Cordis:

- Usa `ctx.on()` para registrar listeners de Events.
- Usa `ctx.effect()` para ser propietario de una suscripción externa que devuelva un disposer.
- Conserva los disposers que devuelven las APIs de Service, Tool, Slot, temporizador y tema de Cordis.
- No crees efectos secundarios a nivel de proceso o de página en el ámbito del módulo ni fuera de `apply()`.

Recomendado:

```js
return {
  apply(ctx) {
    const service = ctx.get('serviceName')
    if (service === undefined) return
    ctx.effect(() => service.subscribe((value) => {
      console.log(value)
    }))
  },
}
```

Si `subscribe()` no devuelve un disposer, consulta primero si el Service proporciona un mecanismo de limpieza compatible. No asumas que unload elimina automáticamente callbacks arbitrarios de terceros.

## Temporizadores de Host y Client

En ambas plataformas, el temporizador es un Service llamado `timer` con la misma interfaz; no es un Builtin. Consulta `{ "service": "timer" }` mediante `Service.listService` de la plataforma correspondiente antes de usarlo. Declara `inject: ['timer']` antes de usar el mixin de temporizador.

Retardo de un solo disparo:

```js
return {
  inject: ['timer'],
  apply(ctx) {
    const onClick = () => {
      ctx.timeout(() => console.log('done'), 300)
    }
    // Pass onClick to a queried Slot UI.
  },
}
```

Trabajo periódico en un componente React:

```js
return {
  inject: ['timer'],
  apply(ctx) {
    function Clock() {
      React.useEffect(() => ctx.interval(() => console.log('tick'), 1000), [])
      return React.createElement('div', null, 'Running')
    }
    // Register Clock in a queried Slot.
  },
}
```

Incorrecto:

```js
return {
  apply(ctx) {
    ctx.timeout(() => console.log('invalid'), 300)
  },
}
```

```js
setTimeout(() => console.log('invalid'), 300)
```

El primer ejemplo no declara la dependencia dura del temporizador. El segundo usa un temporizador global que no existe.

## Escucha Events

Consulta primero el Event Provider para confirmar la plataforma, el orden de los parámetros, el valor de retorno y el `mode`.

Event de emisión ordinaria:

```js
return {
  apply(ctx) {
    ctx.on('some/event', (payload) => {
      console.log(payload)
    })
  },
}
```

El último parámetro de un Event Waterfall es `next`. Salvo que el listener detenga intencionadamente el procesamiento posterior, debe llamarlo y devolverlo:

```js
return {
  apply(ctx) {
    ctx.on('some/waterfall', (payload, next) => {
      console.log(payload)
      return next()
    })
  },
}
```

## Registra la UI del Client

Consulta `Slots.listSubTree` sin `root` para elegir un objetivo en el árbol compacto de propósito y topología y, después, consulta el Slot exacto con `root` antes de escribir su registro. El resultado exacto determina:

- el propósito del Slot en el diseño;
- si su protocolo de registro es `single`, `list`, `keyed` o `chain`;
- las opciones de registro;
- las props estándar de ámbito y las props de propietario de negocio;
- los ocupantes actuales, los riesgos de reemplazo y los Slots descendientes.

Usa `ctx.get('slots')` y gestiona su ausencia. Después usa `slots.inject` para esperar la declaración del Slot y llama a `slots.register` dentro del callback:

```js
return {
  apply(ctx) {
    const slots = ctx.get('slots')
    if (slots === undefined) return
    slots.inject('target.slot', () => slots.register(
      { name: 'target.slot', id: 'my-view' },
      (props) => React.createElement('div', null, String(props.someValue)),
    ))
  },
}
```

`ctx.get('slots')` no requiere inyección. No lo reescribas como `ctx.slots` salvo que declares `inject: ['slots']`:

```js
return {
  apply(ctx) {
    ctx.slots.register({ name: 'target.slot' }, () => null)
  },
}
```

No adivines un `id`, `key`, selector o props antes de consultar el protocolo del Slot. No recurras por defecto a los Slots de nivel raíz `root`, `sidebar`, `conversation` o `details`; reemplazar un ocupante completo también elimina los Slots descendientes que declara.

### Páginas de ajustes

Una UI de ajustes completa debería registrar normalmente su propia sección mediante `settings.section` para obtener un área de contenido completa. `settings.general.item` solo es adecuado para una preferencia compacta de uso general. Consulta el subárbol, las opciones y las props reales de ambos y, después, selecciona el punto de entrada más reducido que siga siendo suficiente.

Los Plugins dinámicos son temporales y locales al proceso, por lo que su UI de ajustes no necesita almacenamiento persistente. No añadas ajustes durables ni otro mecanismo de persistencia para ella. Registra la UI en el Slot de ajustes adecuado y mantén en memoria cualquier estado de interacción transitorio durante la vida del Plugin.

### Datos de sesión y de página

Un Slot con ámbito de sesión puede proporcionar `useSession`, `useSessions`, `useWorkspaces`, `useProjection`, estado de entrada o acciones mediante props estándar. Sigue el resultado de la consulta y prefiere directamente las props de propietario o estándar; no añadas un RPC del Host para datos que ya están presentes.

Selecciona solo los campos que la UI necesita realmente. No copies ni renderices una Conversation Snapshot, una Session, una Tool call o un objeto de props de Slot completos.

### Panel específico de Cordis Run

Para colocar UI interactiva en la tarjeta más reciente de `cordis_run`, registra `tool.view.cordis` con `key: 'self'`:

Cuando la funcionalidad necesita interacción del usuario ligada al resultado de este Package, esta región suele encajar bien porque mantiene los controles en el flujo de conversación junto a la tarjeta Run. No es el objetivo por defecto para toda UI del Client: los ajustes, las barras laterales, las acciones de mensaje y los overlays deberían usar sus propios Slots consultados cuando esas ubicaciones encajen mejor con la funcionalidad.

```js
return {
  apply(ctx) {
    const slots = ctx.get('slots')
    if (slots === undefined) return
    slots.inject('tool.view.cordis', () => slots.register(
      { name: 'tool.view.cordis', key: 'self' },
      (props) => React.createElement('div', null, `Package ${props.packageId}`),
    ))
  },
}
```

En runtime, `self` se vincula a `pluginId + packageId`. No incluyas `pluginRunId` en la key. Cuando el mismo Package se ejecuta varias veces, la tarjeta Run más reciente aloja la UI y las tarjetas antiguas se degradan automáticamente.

### Tarjetas de Tool ordinarias

Para personalizar la tarjeta de llamada de una Tool de modelo ordinaria, consulta `tool.call.toolview`. Su key es el nombre de la Tool; registrar una key existente puede reemplazar la tarjeta por defecto del producto. Cuando personalices solo una Tool recién añadida, verifica primero su schema con `Tool.listTools` y consulta después el `ToolCallOwnerProps` completo.

### Overlays y puntos de entrada locales

- Para toasts, avisos de estado y overlays de todo el marco, consulta primero `shell.overlay`; observa sus reglas de pointer-events y de ordenación.
- Cuando el objetivo seleccionado sea un Slot de overlay global, decide si la UI debe poder arrastrarse, cómo la muestra y la oculta el usuario y qué capas existentes debe cubrir o por debajo de cuáles debe permanecer.
- Para acciones pequeñas de la barra lateral, prefiere Slots interiores aditivos como `sidebar.footer.action`; no reemplaces toda la barra lateral.
- Para contenido complementario tras un turno de conversación, consulta `conversation.chat.turnTail` y registra según su selector de cadena y sus reglas de fallback.

## Temas y estilos

Determina primero el ámbito del cambio:

1. Tema global: consulta primero `Theme.listTokens` y, después, `{ "service": "theme" }` mediante `Service.listService` del Client. Aporta los valores claro y oscuro para cada sobrescritura como exige la consulta y conserva el disposer devuelto.
2. Los componentes propios del Package: usa `styles.insert(css)` y prefiere las variables CSS del tema para los colores.
3. Contenido visible nuevo: elige primero un Slot y decide después entre CSS local y tokens globales.

No manipules `document.body`, `window` ni selectores DOM del producto codificados. El Service de tema cambia los tokens pero no crea UI. Los Slots crean UI pero no reemplazan el sistema de temas.

## Llama al Host desde el Client

El Host registra un método privado de paquete con `harness.handle(method, handler)` y el Client lo invoca con `host.call(method, args)`. Esto es JSON RPC de Client→Host.

Host:

```js
return {
  apply(ctx) {
    harness.handle('read-state', async (args) => {
      return { value: args.key }
    })
  },
}
```

Client:

```js
return {
  async apply(ctx) {
    const result = await host.call('read-state', { key: 'demo' })
    console.log(result.value)
  },
}
```

Los argumentos y los valores de retorno deben ser JSON sin pérdida. No pases funciones, elementos React, instancias de clase, Contexts, Services ni otros objetos de runtime; devuelve `null` cuando no haya datos de respuesta. No registres un Remote Service público ni uses `ctx.remote` para la comunicación privada de paquete.

## Registra una Tool de modelo dinámica

El Host puede usar `harness` para registrar una Tool invocable en el siguiente paso del modelo. Consulta primero la firma actual de `harness` con `Builtin.listBuiltins` del Host y, después, inspecciona los nombres y los schemas de las Tools existentes con `Tool.listTools` para evitar conflictos.

Los argumentos y los valores de retorno de la Tool deben ser compatibles con JSON. `execute` es propietario del resultado de negocio; render y la presentación solo son propietarios de lo que ven el modelo y la UI nativa. El registro de la Tool debe pertenecer al Fiber actual del Plugin para que se elimine automáticamente después de stop o update.

## Gestiona los datos en vivo internos

Las instancias de Service, los payloads de Event, las props de Slot, las Session y Conversation Snapshots, el estado de las Tools y otros objetos de DSH/Cordis son datos en vivo internos.

No:

- llames a `JSON.stringify` o `structuredClone` sobre estos objetos o sus descendientes;
- los enumeres recursivamente, los copies por completo o los muestres en su totalidad;
- coloques objetos del Host en el estado de larga duración del Package o en los valores de retorno de RPC.

Lee solo los campos hoja que requiere la funcionalidad actual. Extrae las cadenas, los números, los booleanos y otros valores escalares mínimos antes de construir JSON propio.

## Versiones, aprobación y reparación

- Un Plugin es la instancia estable identificada por `pluginId`.
- Un Package es una versión de código inmutable identificada por `packageId`.
- Cada intento de activación tiene su propio `pluginRunId`.
- `currentPackageId` es la última versión correcta; no implica que el Plugin se esté ejecutando actualmente.
- `nextPackageId` es el objetivo en espera de aprobación, activándose, en espera de activación del Client o que falló más recientemente.

Elige el modo de `cordis_run` de la siguiente manera:

| Estado actual | Objetivo | modo |
| --- | --- | --- |
| Sin actual | Cualquier Package del Plugin | `run` |
| Con actual | El mismo Package | `run` |
| Con actual | Un Package distinto | `update` |
| Falló la actualización | `nextPackageId` | `update` para reintentar |
| Falló la actualización | `currentPackageId` | `run` para revertir |

Un Package del Client no autorizado devuelve `awaiting-approval`. Una marca de verificación simple autoriza solo el Package actual; una doble autoriza las versiones futuras del mismo Plugin. Un grant permanece tras un fallo técnico de runtime. Un Package autorizado devuelve `starting` y se completa de forma asíncrona en el navegador.

Después de un fallo técnico:

1. Usa `cordis_inspect_self(pluginId, packageId)` para leer el código fuente de la versión fallida y los diagnósticos exactos.
2. Si el error implica una capacidad desconocida, vuelve a listar y consultar el Provider correspondiente.
3. Define un Package nuevo en el mismo Plugin; no sobrescribas el Package fallido.
4. Vuelve a ejecutar con el `packageId` nuevo y el modo correcto.

No reintentes automáticamente después de que el usuario rechace la aprobación. Una actualización fallida no restaura automáticamente el Run físico antiguo; ejecuta explícitamente current cuando se requiera recuperación.

## Modifica @pluginId

Cuando el usuario identifique un objetivo con `@pluginId`, no crees otro Plugin. El contexto inyectado contiene solo la identidad, los punteros de versión y el Package base por defecto, no código fuente.

Modifícalo de la siguiente manera:

1. Lee el Package base con `cordis_inspect_self(pluginId, packageId)`.
2. Conserva la mitad del Host o del Client que no necesita cambiar y modifica solo el código objetivo.
3. Llama a `cordis_define` con `plugin.kind: 'existing'` y el `pluginId` original.
4. Usa el `packageId` devuelto; cuando exista current, activa la nueva versión con `update` en el caso habitual.

Si la referencia no está disponible, explica que el Plugin se eliminó, pertenece a otra Session o se perdió al reiniciar el proceso. No crees un reemplazo con el mismo nombre.

## Comprobaciones de fallos habituales

| Fallo | Comprueba primero |
| --- | --- |
| `service "x" is not declared` | Si el código usa `ctx.x` sin declarar `inject: ['x']` en el objeto del Plugin; cambia a `ctx.get('x')` con una comprobación de ausencia o declara una dependencia dura real |
| `cannot get property "timer" without inject` | Consulta el Service de temporizador y declara `inject: ['timer']` |
| Fallo de parseo del Client | Si el código usa JSX, TypeScript, import o un global no disponible |
| Fallo de registro del Slot | Si se consultó el subárbol en vivo, el Slot existe y las opciones, la key o el selector cumplen el protocolo devuelto |
| La UI carga pero la página informa de un error | Inspecciona el diagnóstico `client-render` y el stack; el error pertenece a un Run exacto, así que define un Package nuevo para repararlo |
| Fallo de `host.call` | El nombre del handler del Host, el `pluginRunId` actual, los argumentos JSON y las dependencias reales de Service dentro del handler |
| Fallo de actualización | Conserva la semántica de current/next; repara next y actualiza, o ejecuta current para revertir |
