# Agent Note: Contratos de invariantes de paquete con sentido

Status: implemented

[English](2026-07-19-package-invariant-runtime-contracts.md) | Español

## Problema

El servicio de invariantes propiedad de los paquetes hizo exhaustiva la publicación y el registro, pero su primera línea base generada aceptaba instaladores vacíos. Una continuación sustituyó después esos vacíos por aserciones genéricas sobre nombres de plugins, inyecciones, efectos, métodos de servicio y ejemplos fijos de librerías puras. Esas aserciones hicieron ejecutable a todos los companions sin hacer el sistema más seguro: TypeScript, el arranque de Cordis, las pruebas de paquetes y las pruebas de carga de módulos ya hacen cumplir esas formas, mientras que el servicio de invariantes debería detectar estados de tiempo de ejecución imposibles.

Un invariante de tiempo de ejecución útil relaciona observaciones a lo largo del tiempo o a través de una estructura de datos mutable. Ejemplos: un evento terminal sin su inicio, un delta de LLM para un bloque que no está abierto, o un resultado durable cuya identidad difiere de su petición. Limitarse a confirmar que un método declarado existe, que un plugin tiene el nombre esperado o que un ejemplo constante sigue devolviendo un valor conocido no es una relación de ese tipo.

Algunos paquetes no poseen de verdad ninguna relación observable de forma continua. Las utilidades puras, los paquetes solo de composición, los adaptadores delgados, los binarios y los paquetes de soporte de pruebas pueden tener contratos importantes, pero esos contratos se hacen cumplir mejor con tipos, comprobaciones de carga, pruebas unitarias enfocadas o pruebas de integración. Exigir una aserción de tiempo de ejecución sintética para esos paquetes optimizaría para satisfacer un gate en lugar de detectar corrupción.

## Decisión

### El registro es exhaustivo; las aserciones deben tener sentido

Cada paquete del workspace publica un companion `./invariant` construido por separado y registra su nombre exacto de paquete npm. Un companion hace una de dos cosas:

- instala una comprobación propiedad del paquete sobre un stream de eventos o una estructura de datos mutable relevante e informa de las violaciones a través de su reporter `fail(message)` enlazado; o
- usa un instalador vacío cuya declaración lleva un comentario específico del propietario `No runtime invariant:` que explica por qué el paquete no tiene ninguna relación de tiempo de ejecución plausible que observar.

La forma vacía es una conclusión arquitectónica explícita, no un marcador de posición generado. Un cambio futuro del paquete que introduzca estado mutable o un protocolo de eventos debe sustituir la explicación por la comprobación correspondiente.

El servicio central `dsh-invariants` posee solo la configuración, la unicidad del registro, el ciclo de vida de las fibras hijas, el rollback, la liberación y el fallo atribuido al paquete. No expone helpers genéricos de forma de plugin, forma de servicio ni aserciones de arranque, y no importa ningún paquete de producto.

### Comprobaciones implementadas

El workspace actual de 103 paquetes tiene 21 companions ejecutables y 82 companions vacíos justificados.

| Propietario | Relación de tiempo de ejecución |
|---|---|
| `dsh-session` | Crecimiento estricto de secuencia, contención de turno/paso y emparejamiento de llamada de herramienta/resultado del mismo paso. |
| `dsh-agent` | Estado de agent no repetido y transiciones terminales de liberación (disposal). |
| `dsh-scope` | Presencia del carrier de eventos con ámbito y consistencia del subject enrutado. |
| `dsh-agent-loop` | Reconstrucción explícitamente marcada y congelada de la petición del loop a partir del log de eventos de la sesión. |
| `dsh-llm` | Gramática de bloques del stream, coincidencia de tipo/índice de deltas, uso único, bloques cerrados y finalización terminal. |
| `dsh-llm-retry` | Los registros de reintento durables identifican el paso cerrado más reciente del turno abierto, siguen siendo únicos por paso, crecen monótonamente y se mantienen dentro de los límites de reintento y de timer no negativo. |
| `dsh-tools` | Etapas pre/execute/post monótonas e instantáneas finales inmutables de ejecución/resultado. |
| `dsh-system-prompt` | Restricciones autoritativas de sección de ensamblaje, herramienta y datos de variables. |
| `dsh-compaction` | Emparejamiento de inicio/resumen/fin de compactación, extremos de rango, recuentos de tokens y presencia de resumen correcto. |
| `dsh-hook-protocol` | Correlación de invocación/resultado de hooks, dialecto, identidad y restricciones de duración. |
| `dsh-sandbox-policy` | Los eventos durables `sandbox/mode` usan el vocabulario cerrado de modos de sandbox. |
| `dsh-fs` | Los eventos de decisión/observación del sistema de archivos llevan identidades de destino y versión utilizables. |
| `dsh-goal` | Las instantáneas de goal durables preservan la atribución de origen, el contenido renderizado, las revisiones, las relaciones de ciclo de vida y marcas de tiempo, y las rondas admitidas secuencialmente. |
| `dsh-goal-round-driver` | Los mensajes de continuación originados en goals coinciden con el prompt reconstruido a partir del estado durable de goal precedente. |
| `dsh-subagent` | Los eventos de añadir/quitar provider y de inicio/fin de hijos preservan identidad y emparejamiento. |
| `dsh-permission-presets` | Las decisiones de permiso durables nombran un preset de la tabla de permisos activa. |
| `dsh-user-approval` | Los registros de aprobación asked/decided se emparejan por llamada y usan resultados y políticas válidos. |
| `dsh-workflow` | Los eventos de inicio/fin de flujo de trabajo y de agent hijo preservan los metadatos de ejecución, la identidad, el resultado, el recuento y las relaciones de error. |
| `dsh-jobs` | Las instantáneas de tareas actuales y terminales preservan las relaciones de id/kind, propietario, estado y marcas de tiempo. |
| `dsh-tool-todo` | Las instantáneas durables de la lista completa usan elementos recortados únicos y estados cerrados. |
| `dsh-time-context` | Las lecturas de reloj atribuidas al plugin concuerdan con el turno abierto de la sesión, la siguiente posición de pre-paso y la línea base transcurrida; el tiempo renderizado se parsea y no es posterior a su evento. |

Los companions respaldados por sesión validan los eventos durables existentes al cargar, usando el prefijo que precede a cada candidato cuando la relación depende del orden de los eventos. Otras comprobaciones observan el límite de eventos en vivo autoritativo o el resultado mutable del servicio. La validación se ejecuta antes de la publicación cuando aceptar un evento inválido comprometería de otro modo un estado malo.

### Gate del repositorio y pruebas

`verify-package-invariants` descubre cada paquete del workspace y hace cumplir el source del companion, el registro por nombre exacto, la forma de Loader solo con nombres, las exportaciones `./invariant`, los archivos de publicación, las dependencias, las referencias de TypeScript y las entradas de bundle. Su regla AST rechaza los marcadores generados, las exportaciones por defecto y los instaladores vacíos sin explicación. Un instalador no vacío debe aceptar y usar el reporter de fallos, y el registro debe pasar esa función `install` local comprobada. El gate deliberadamente no infiere la calidad semántica de los nombres de método ni de las llamadas a helpers.

Vitest monta `InvariantRegistry` con `{ enabled: true }` para cada topología de pruebas de paquete y carga el companion propietario. El mapeo de rutas de la subruta invariant resuelve los companions de source en lugar de la salida compilada obsoleta. Las suites enfocadas cubren las observaciones válidas e inválidas de cada companion ejecutable, y la topología exhaustiva pasa cada companion de source por la normalización real de namespaces del Loader. Tras validar cada mapa de publicación con el gate estructural, un gate de artefactos prepara los archivos `lib/` declarados en el manifest, importa la autorreferencia `./invariant` compilada bajo Node plano y repite esa comprobación de forma de Loader, de modo que un companion que importe un chunk de tiempo de ejecución no declarado falla antes del lanzamiento. Las pruebas que sintetizan streams de eventos deben producir un ciclo de vida circundante válido, salvo que la prueba afirme intencionadamente una violación.

## Alternativas consideradas

- **Conservar los companions vacíos generados.** Rechazada porque un marcador de posición sin explicación puede sobrevivir después de que un paquete gane una relación de tiempo de ejecución con sentido.
- **Exigir una aserción a cada paquete.** Rechazada porque las aserciones de presencia de método, forma de plugin y ejemplo fijo duplican contratos más fuertes de tipos, carga y pruebas unitarias sin comprobar la consistencia en tiempo de ejecución.
- **Conservar los helpers genéricos de forma en el servicio.** Rechazada porque difuminan la validación de API en tiempo de compilación con los invariantes de tiempo de ejecución y fomentan suposiciones de producto definidas centralmente.
- **Mover las comprobaciones de producto al servicio.** Rechazada porque el vocabulario de producto, las dependencias, las pruebas y la propiedad del cambio pertenecen al paquete que emite los datos.
- **Registrar los companions implícitamente desde las entradas raíz.** Rechazada porque el orden de composición y la presencia opcional del servicio crearían efectos ocultos.

## Consecuencias

- Cada paquete tiene una propiedad y un cableado de publicación visibles, pero solo los paquetes con una relación de tiempo de ejecución plausible añaden listeners o rastrean estado.
- Los companions vacíos siguen siendo decisiones revisables con explicaciones específicas del paquete, y fallan el gate si se elimina la explicación.
- Las declaraciones de tipos, la cargabilidad de Cordis, los metadatos de plugins, las APIs de métodos de servicio y el álgebra pura siguen cubiertos por sus gates propietarios de compilación, carga, unitarias o integración.
- Los fallos de tiempo de ejecución identifican el paquete npm propietario y señalan una observación inconsistente en lugar de repetir una forma de API requerida.
- Los contratos originales del servicio de selección, precedencia de blocklist, propiedad duplicada, rollback, liberación y HMR permanecen sin cambios.
