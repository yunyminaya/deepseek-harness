# Agent Note: Contrato del servicio de invariantes propiedad de los paquetes

Status: implemented

[English](2026-07-19-package-owned-invariant-service.md) | Español

## Problema

Las comprobaciones de invariantes de tiempo de ejecución abarcan trazas de sesión, estado de agent, despacho con ámbito y reconstrucción de peticiones. Poner todas las comprobaciones en un único paquete de diagnósticos obliga a ese paquete a importar vocabularios de producto de dominios no relacionados, centraliza las pruebas lejos de sus propietarios y exige que el paquete central cambie cada vez que un paquete de producto añade o elimina una comprobación.

Los despliegues que optan por diagnósticos necesitan algo más que la presencia o ausencia de un plugin. Esa composición transporta las contribuciones de invariantes conocidas y a la vez permite un interruptor global de apagado y diagnósticos selectivos por paquete. La selección debe permanecer estable cuando un paquete carga después o se recarga bajo HMR, y las contribuciones deshabilitadas no deben permitir que dos plugins reclamen en silencio el mismo nombre de paquete.

La propiedad de paquetes también debe ser exhaustiva. Sin una regla mecánica del repositorio, un paquete nuevo puede omitir el companion, la dependencia o el cableado de publicación y permanecer invisible para los diagnósticos hasta que un mantenedor note el hueco.

## Decisión

### Un único servicio de registro, contribuciones propiedad de los paquetes

`@deepseek-ai/dsh-invariants` es un plugin de servicio de Cordis independiente del producto que registra `ctx.invariants`. Posee la configuración, la unicidad del registro, el ciclo de vida de las fibras hijas y los fallos atribuidos al paquete. No importa ningún paquete de session, agent, scope o agent-loop y no contiene ninguna de sus comprobaciones.

Cada paquete del workspace publica un plugin companion `./invariant` que registra su nombre completo exacto de npm. Un companion comprueba una relación con sentido de eventos o de datos mutables cuando su propietario tiene una; si no, lleva una explicación específica del propietario para su instalador vacío. Los marcadores de propiedad generados y las aserciones sintéticas de forma de API están prohibidos por la [Agent Note de contratos de tiempo de ejecución](2026-07-19-package-invariant-runtime-contracts.es.md) que la sigue. Las entradas raíz de los paquetes no importan ni registran diagnósticos implícitamente, de modo que cargar un paquete raíz no cambia la comprobación en tiempo de ejecución ni exige el servicio de invariantes.

### Configuración y selección

```ts
interface Config {
  enabled?: boolean
  package_allowlist?: string[]
  package_blocklist?: string[]
}
```

Los valores por defecto son `enabled: true`, `package_allowlist: []` y `package_blocklist: []`. Para un nombre de registro completo, la selección es:

```ts
export function selected(enabled: boolean, package_allowlist: RegExp[], package_blocklist: RegExp[], packageName: string): boolean {
  return enabled
    && (
      package_allowlist.length === 0
      || package_allowlist.some(pattern => pattern.test(packageName))
    )
    && !package_blocklist.some(pattern => pattern.test(packageName))
}
```

Las coincidencias de blocklist anulan las de allowlist. Cada entrada de lista es un source de regex de JavaScript sensible a mayúsculas compilado con `new RegExp(pattern)`. La coincidencia no está anclada salvo que los llamadores aporten `^` y `$`; la sintaxis delimitada por barras y las banderas no se interpretan. El arranque rechaza los sources en blanco, con espacios alrededor, inválidos o duplicados dentro de cualquiera de las dos listas. Un source que no coincida con ningún paquete cargado sigue siendo válido porque el orden de registro, la carga posterior y el HMR no deben cambiar la validez de la configuración.

### Registro y propiedad del fallo

El límite público de registro es `ctx.invariants.register(packageName, installer)`. Reserva un registro activo por nombre completo de paquete npm incluso cuando los filtros deshabilitan la instalación, y devuelve el disposer de efectos. Liberar el companion o el servicio libera la reserva y todo el estado de la contribución.

Un instalador habilitado se ejecuta en una fibra hija de Cordis dedicada, propiedad del servicio. `InvariantInstaller.inject` declara explícitamente la API de servicios de la fibra hija; el registro no lleva metadatos de dependencias específicos del producto. El servicio hace join de la promise devuelta por el instalador antes de que el registro tenga éxito, de modo que las comprobaciones de arranque asíncronas siguen siendo transaccionales. El instalador recibe un reporter `fail(message)` enlazado. Llamarlo lanza una subclase de `Error` llamada `InvariantError` con el código estable `INVARIANT` y el `packageName` que registra; no extiende una base de error de paquetes de producto.

La preparación del registro es transaccional. Si un instalador falla tras registrar listeners, la fibra hija se libera por completo y la reserva de nombre se suelta antes de que el fallo escape. Los registros filtrados no crean ninguna fibra hija, pero conservan su reserva hasta la liberación. Recargar un companion empieza por tanto con un único estado de instalador limpio; las contribuciones con estado reconstruyen sus líneas base a partir de sus servicios propietarios.

El antiguo punto de entrada de plugin funcional y el constructor de `InvariantError` de un argumento no se conservan como APIs de compatibilidad. El repositorio está en prelanzamiento y todos los puntos de llamada migran juntos al servicio y al error atribuido al paquete.

### Companions con estado iniciales y propiedad exhaustiva

| Entrada de companion | Nombre de registro | Comprobaciones propiedad |
|---|---|---|
| `@deepseek-ai/dsh-session/invariant` | `@deepseek-ai/dsh-session` | secuencia de sesión, contención de turno/paso y traza de llamada/resultado del mismo paso |
| `@deepseek-ai/dsh-agent/invariant` | `@deepseek-ai/dsh-agent` | transiciones de estado de agent |
| `@deepseek-ai/dsh-scope/invariant` | `@deepseek-ai/dsh-scope` | presencia del carrier de eventos con ámbito y consistencia del subject |
| `@deepseek-ai/dsh-agent-loop/invariant` | `@deepseek-ai/dsh-agent-loop` | reconstrucción de la petición de modelo |

Estos cuatro propietarios aportaron las comprobaciones con estado iniciales. La decisión posterior de contratos de tiempo de ejecución añade comprobaciones para otros diecisiete propietarios con relaciones reales de eventos o datos mutables y registra companions vacíos justificados para el resto. Cada companion es una exportación `./invariant` empaquetada por separado con sus propias declaraciones y una forma de plugin de namespace segura para el Loader; el companion del propio paquete de servicio importa su tipo de servicio local para evitar una autodependencia.

`verify-package-invariants` descubre cada paquete del workspace y rechaza el source de companion ausente, los marcadores generados, los instaladores vacíos sin explicación, los instaladores no vacíos que omiten o ignoran el reporter, los nombres de registro ajenos o sin resolver, las exportaciones `./invariant` o los archivos publicados ausentes, las dependencias peer/development de invariantes y las referencias de proyecto ausentes, y los overrides de bundle que omiten la entrada del companion.

### Mapa semántico de eventos con ámbito

El resolver generado de subject de eventos con ámbito vive en `dsh-scope`, junto al contrato y el invariante que lo consumen. `gen-scoped-events` usa el Programa de TypeScript raíz para enumerar las declaraciones `this: Scoped<Base>`, inferir los tipos de clave de enrutado a partir de llamadas reales `scopeTarget(base, key)` y exigir un subject de payload inequívoco o un marcador explícito de no soportado. El mapa de tiempo de ejecución comprometido no importa ningún paquete propietario de eventos, de modo que la completitud semántica no expande el cierre de tiempo de ejecución ni del servicio ni del paquete de scope.

### Composición de ejemplo y salida del SDK

El agent spine de ejemplo monta el servicio y las cuatro subrutas de companions con estado, reenviando `enabled`, `package_allowlist` y `package_blocklist` al servicio. La composición de Cordis generada del SDK emite las mismas entradas. Una entrada de subruta añade su paquete npm raíz instalable en lugar de tratar la subruta como un nombre de paquete. Los árboles de configuración TUI y Web del `dsh` publicado omiten el servicio y los companions según la [decisión de configuración publicada](../simplification/2026-08-03-omit-invariants-from-shipped-config.es.md).

Las restricciones del workspace reconocen el bundle de invariantes separado, y las exportaciones de paquetes, las referencias de proyecto, la configuración de build, las declaraciones de dependencias y el lockfile describen los mismos metadatos de publicación. Los catálogos de configuración generados, los grafos de módulos y la documentación de API se derivan de esas fuentes.

## Pruebas

Las pruebas del servicio cubren los valores por defecto, la deshabilitación global, la selección allow/block, la precedencia de blocklist, el anclaje, la coincidencia no anclada, la sensibilidad a mayúsculas, la configuración inválida, los patrones sin coincidencias, el registro tardío, la propiedad duplicada, la liberación, el rollback y el re-registro bajo HMR. Los propietarios con comprobaciones ejecutables mantienen el comportamiento positivo y negativo junto al source del companion.

Las pruebas de composición cubren el reenvío del spine estándar y las entradas del SDK generadas. Las pruebas del Loader preservan el namespace de cada companion, mientras que los smokes de Node plano compilados ejercitan las exportaciones de subruta compiladas. El gate de frescura de eventos con ámbito vuelve a ejecutar su análisis semántico de Program.

Cada configuración de Vitest carga un host de pruebas que monta un servicio explícitamente habilitado antes del primer plugin de una raíz de Cordis ordinaria y añade el companion del paquete bajo prueba. Una topología exhaustiva monta todos los companions de paquetes una vez; las pruebas enfocadas de servicio y propietarios construyen su propia topología de invariantes para poder ejercitar la deshabilitación, el filtrado, el rollback y la recarga sin propiedad duplicada. Las pruebas del gate también ejecutan la función `apply` de cada companion y verifican que llama a `register` con su nombre del manifest, en lugar de aceptar solo el texto del source.

## Alternativas consideradas

- **Conservar todas las comprobaciones en `dsh-invariants`.** Rechazada porque el registro seguiría importando todos los dominios de producto comprobados, los cambios de propietario exigirían ediciones centrales y las pruebas de paquetes seguirían despegadas de los contratos que protegen.
- **Dejar que las entradas raíz registren comprobaciones implícitamente cuando `ctx.invariants` exista de casualidad.** Rechazada porque el comportamiento de la raíz dependería del orden de composición y de la presencia opcional del servicio, los diagnósticos no podrían seleccionarse de forma independiente, y la carga de paquetes ocultaría un efecto de registro fuera de un companion explícito.
- **Descubrir automáticamente cada archivo `invariant.ts` en tiempo de ejecución.** Rechazada porque el descubrimiento por sistema de archivos/paquetes no es un contrato de propiedad en tiempo de ejecución, vuelve ambigua la publicación empaquetada y no puede expresar el orden de carga explícito de Cordis ni la instalación de dependencias. La generación en tiempo de build, la verificación y el host de pruebas pueden enumerar el árbol de source porque validan la completitud del repositorio en lugar de componer un despliegue publicado.
- **Validar las entradas allow/block contra el conjunto de paquetes actualmente cargado.** Rechazada porque un patrón sin coincidencias puede apuntar intencionadamente a una contribución cargada después o bajo HMR; el orden de carga actual no debe determinar la validez de la configuración.

## Consecuencias

- Los paquetes de producto poseen y prueban sus aserciones relacionales mientras el servicio sigue independiente del producto.
- Cada paquete paga el coste de publicación y dependencia de un companion; solo los propietarios con una relación de tiempo de ejecución con sentido añaden coste de listeners o de rastreo de estado.
- Las composiciones que montan los diagnósticos pueden deshabilitar todas las comprobaciones o seleccionar nombres de paquete sin cambiar su árbol de plugins.
- Las entradas de companion explícitas hacen visible el coste de diagnóstico y la propiedad en la configuración de Cordis y en las exportaciones de paquetes.
- Una contribución ejecutable seleccionada añade una fibra hija y su coste de listeners/estado; una contribución vacía seleccionada no tiene coste de listeners ni de rastreo de estado, mientras que los registros filtrados conservan solo la propiedad del nombre.
- Los sources de regex son configuración de despliegue y permanecen fijos hasta que el servicio se recarga.
- Las raíces de Vitest ordinarias instalan el companion seleccionado del paquete de pruebas propietario; una topología exhaustiva paga una vez el coste completo de fibras hijas por la cobertura de registro de todo el repositorio.
- La validación del almacenamiento de sesión, el instantaneado (snapshotting), la congelación, la validación de eventos de origen citados y la aceptación de superficie siguen siempre activos y no se ven afectados por la selección de invariantes.
