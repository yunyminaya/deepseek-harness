# Agent Note: Escritura por lotes acotada de persistencia de sesión

Status: implemented

[English](2026-08-08-bounded-session-persistence-write-batching.md) | Español

## Problema

Las respuestas en streaming pueden emitir muchos eventos `assistant/chunk` en un intervalo corto. El coordinador de persistencia anterior programaba una anexión de backend tan pronto como una cola inactiva recibía un evento. Los eventos que llegaban mientras esa anexión estaba activa compartían un lote de seguimiento, pero un backend rápido todavía podía producir muchas anexiones durables pequeñas. Cada anexión JSONL crea y sincroniza un frame Zstandard o sufijo sin formato, mientras que cada anexión SQLite abre y confirma una transacción e incrementa la revisión de sesión.

Eliminar eventos de chunk o reemplazarlos con mensajes ensamblados reduciría el almacenamiento lógico, pero también cambiaría el log de eventos, la reproducción, los números de secuencia, las marcas de tiempo y los seq de chunk citados por los mensajes de asistente. El problema de amplificación de escritura no requiere ese cambio semántico mayor.

### Línea base cuantificada

Los fixtures del repositorio hacen concreto el volumen lógico. Decodificar las filas empaquetadas actuales en [`goal-multi-turn-actions`](../../../../apps/web/tests/snapshots/goal-multi-turn-actions/session.jsonl) produce 2.098 eventos: 2.017 chunks (96,1%). Sus líneas JSONL desempaquetadas ocupan 332.647 de 379.225 bytes de evento (87,7%), mientras que el empaquetado de chunk reduce el archivo confirmado a 89.176 bytes y 182 filas de almacenamiento, incluyendo 23 filas de chunk empaquetadas. [`permission-policy-context`](../../../../apps/web/tests/snapshots/permission-policy-context/session.jsonl) produce 813 eventos: 746 chunks (91,8%) y 118.935 de 184.821 bytes de evento desempaquetados (64,4%); su archivo empaquetado es 84.917 bytes y 123 filas de almacenamiento, incluyendo 14 filas empaquetadas. Estos son fixtures deterministas rastreados, no una distribución de carga de trabajo de producción, pero demuestran por qué eliminar chunks reduciría el volumen lógico y por qué el diseño de fila empaquetada existente ya elimina gran parte de su costo de sobreinserción JSON.

SQLite almacena una fila por evento lógico, así que esos mismos logs lógicos retendrían 2.098 y 813 filas de evento respectivamente; el procesamiento por lotes no cambia esos recuentos. JSONL escribe un frame Zstandard y fsync por lote de anexión durable, mientras que SQLite realiza una transacción y un incremento de revisión de sesión por lote. Los archivos de tiempo de ejecución no registran los límites de anexión anteriores, así que los recuentos de filas de fixture no pueden presentarse honestamente como recuentos de fsync o transacción.

El límite de programación es determinista. Con un sumidero que se resuelve inmediatamente, el controlador inmediato anterior podía emitir una anexión por cada evento que llegara después de completar la anexión anterior. Una prueba de controlador admite 20 eventos a 10 ms de intervalo: la ventana fija de 200 ms entrega los 20 a una anexión. Esta es una reducción de 20 a 1 para ese cadencia, no una proporción universal. Eventos dispersos, vaciados obligatorios, escrituras anteriores lentas y diferentes tasas de llegada producen diferentes tamaños de lote.

## Decisión

Los plugins JSONL y SQLite de primera parte exponen `writeBatchMaxDelayMs`, un entero positivo no mayor que el límite de temporizador de Node. Su valor predeterminado es `200`. Cada plugin resuelve el valor al cargar y lo pasa a `PersistenceCoordinator`; el coordinador sigue siendo el único propietario del comportamiento de procesamiento por lotes.

Cada sesión en vivo recibe un `SessionWriteBehind` privado del paquete. Cuando su cola pendiente cambia de vacía a no vacía, el controlador inicia una ventana fija. Los eventos posteriores se unen a ese lote sin restablecer el plazo: esto es coalescencia acotada, no debounce. Cuando el plazo expira, el controlador entrega el prefijo pendiente completo a la serialización y ruta `appendBatch` por id existente. Como máximo una escritura por sesión está activa. Los eventos admitidos durante esa escritura forman un nuevo prefijo pendiente con su propio plazo fijo; si ese plazo expira antes de que la escritura activa se complete, el nuevo prefijo se inicia inmediatamente después.

`writeBatchMaxDelayMs` limita solo la espera de procesamiento por lotes intencional del controlador. La programación del bucle de eventos, la inicialización, una operación serializada anterior y la E/S de backend pueden retrasar la finalización durable, así que la opción no es un SLA de fsync o pérdida por fallo duro.

`session/flush` cancela cualquier espera restante y se convierte en una barrera de quiencia compartida. Drena el intento activo y cada evento admitido mientras la barrera está en ejecución antes de resolverse. El retiro de sesión y la disposición del backend usan esa misma barrera, así que el desmontaje del ciclo de vida nunca espera el temporizador de procesamiento por lotes. La política de checkpoint continúa colocando barreras obligatorias antes de solicitudes de modelo y efectos secundarios de herramienta de nivel superior.

Cada evento permanece durable en su orden y forma originales. El controlador copia cada evento al admitirlo; no se elimina ni reescribe ningún `assistant/chunk`, `seq`, `time`, metadatos de superficie o registro de almacenamiento. JSONL por lo tanto puede codificar más eventos en un frame de anexión, y SQLite puede insertar más filas de evento en una transacción, sin cambiar el formato en disco ni la versión del esquema.

Una anexión en segundo plano fallida restaura su lote completo antes que cualquier evento pendiente más nuevo, reporta el fallo una vez y pausa el reintento automático. El siguiente evento recién admitido abre una ventana fija nueva; un flush explícito, retiro o disposición reintenta inmediatamente y presenta un fallo repetido a su persona que llama. Esto evita un bucle de fallo impulsado por temporizador mientras preserva el límite de flush recuperable existente.

Esta decisión reemplaza solo la cadencia de programación inmediata en [Collapse live persistence into one flush controller](../simplification/2026-07-23-collapse-persistence-flush-state.md). Esa nota sigue siendo autoritaria para un controlador por sesión en vivo, lotes fallidos retenidos, serialización por id, retiro y disposición quieta. El [coordinador de persistencia compartida](2026-06-18-shared-persistence-write-coordinator.md) sigue siendo el propietario del límite de enlace del backend.

## Alternativas consideradas

**No persistir eventos de chunk en streaming.** Rechazado aquí: cambia la autoridad basada en eventos y la semántica de recuperación en lugar de solo la cadencia de escritura física. El rechazo de [mensaje ensamblado](../../rejected/simplification/2026-06-20-assembled-assistant-messages-only.md) existente sigue siendo el guardarraíles hasta que un reemplazo sin pérdida de información defina independientemente la reproducción, bifurcación, enlaces a eventos fuente citados, secuencia y comportamiento de fallo. La [decisión de fila empaquetada](2026-07-26-packed-chunk-rows-by-default.md) sigue siendo la optimización del tamaño de almacenamiento JSONL complementaria.

**Escribir solo en checkpoints semánticos.** Rechazado: maximiza el procesamiento por lotes pero hace que la ventana ordinaria de pérdida por fallo dependa de una política montada por separado. Las escrituras en segundo plano acotadas preservan el progreso entre checkpoints mientras los flush obligatorios mantienen su contrato de ordenamiento más fuerte.

**Debounce desde el último evento.** Rechazado: una respuesta en streaming continua podría posponer su primera escritura indefinidamente. Una ventana fija desde el primer evento pendiente proporciona un límite superior real en la espera de coalescencia intencional.

**Implementar temporizadores por separado en JSONL y SQLite.** Rechazado: la programación, la retención de fallos, las carreras de flush y el desmontaje son preocupaciones del ciclo de vida neutrales del backend. Duplicarlas reabriría la deriva que `PersistenceCoordinator` eliminó.

## Verificación

Las pruebas del controlador usan un reloj falso para probar la ventana fija de 200 ms no restablecedora; barreras de flush inmediato y compartido; eventos admitidos durante una barrera; una cola excesiva detrás de una escritura activa; retención de fallos ordenada; reintento automático pausado; y reintento explícito de un fallo en segundo plano superpuesto. Las pruebas del coordinador ejecutan el controlador a través de notificaciones de sesión, retiro, reclamación de colisión y desmontaje. Las suites JSONL y SQLite retienen su cobertura de contrato de formato de almacenamiento, transacción, recuperación y persistencia compartida.

## Consecuencias

Las ráfagas de eventos de alta frecuencia normalmente producen menos operaciones de anexión durables mientras preservan el recuento exacto de eventos lógicos. La reducción depende de la tasa de llegada y la latencia del backend: una ráfaga dentro de una ventana de 200 ms se convierte en un lote, mientras que los flush obligatorios y los eventos dispersos todavía pueden producir lotes pequeños.

Esta decisión no limita el recuento o bytes de eventos pendientes detrás de un backend lento, y no reduce las filas de SQLite ni el log lógico decodificado. Un límite de memoria demostrado o una política de retención lógica requeriría su propio contrato de fallo y reproducción en lugar de otra regla de temporizador oculta.

Un evento admitido puede permanecer solo en memoria durante la ventana configurada, y luego mientras la programación o el trabajo de backend están pendientes. Los despliegues eligen un valor más pequeño para una ventana ordinaria de pérdida más estrecha o un valor más grande para un procesamiento por lotes más fuerte. Los límites de durabilidad explícitos permanecen sin cambios y omiten la espera.

El nuevo módulo profundo da al temporizador, la escritura activa, el prefijo pendiente, la pausa de reintento y la barrera un único propietario. `PersistenceCoordinator` retiene la inicialización y la serialización de identidad; los backends retienen solo las primitivas de almacenamiento durable. Ni `SESSION_FORMAT_VERSION` ni `SCHEMA_VERSION` de SQLite cambian.