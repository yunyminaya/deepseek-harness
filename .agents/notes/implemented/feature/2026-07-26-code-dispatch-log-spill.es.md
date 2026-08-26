# Agent Note: Spill de la copia durable de los resultados de sub-dispatch de Code Mode

Status: implemented

[English](2026-07-26-code-dispatch-log-spill.md) | Español

> Ámbito: limitar el contenido del evento `tool/code-dispatch` con la implementación de spill existente. La [nota base del host](2026-07-26-code-dispatch-ui-foundation.es.md) aceptó deliberadamente el log ilimitado y difirió el soporte de spill a este cambio; la [nota de live parallel](2026-07-26-code-mode-live-parallel-dispatch.es.md) define el par de eventos que procesa este listener.

## Problema

Después de añadir el registro (logging) de despacho de contenido completo, un programa `run_code` que lee un archivo grande escribía el texto renderizado completo en el log de sesión sin límite ni política de spill, mientras que los resultados nativos se limitaban a `maxInlineBytes` antes de registrarse. Esto trataba de forma distinta los resultados con más probabilidad de ser grandes: los sub-calls están pensados para el trabajo de datos en volumen, y cada turno afectado añadía megabytes al JSONL.

## Decisión

**Un waterfall `tools/code-dispatch-log` en el registro, con la política de spill como su primer listener.**

- **Punto de extensión**: `tools/code-dispatch-log` es un waterfall filtrado por ámbito que el bridge ejecuta sobre cada sub-dispatch asentado antes de añadir `tool/code-dispatch`. El bridge recibe el invocador privado `shapeDispatchLog` del registro como closure de capacidad en `RunCodeBridgeOptions`; el waterfall es el contrato público, y el invocador no añade ningún método de servicio. Si un listener lanza una excepción, el invocador informa con seguridad de cualquier valor lanzado y usa el contenido asentado original. La carga útil de `CodeDispatchLog` lleva la ejecución externa, la clave de enrutamiento `agent`, la identidad de la sub-call y el contenido por defecto: la proyección del resultado renderizado que llevaría un `tool/result` nativo, mientras que el programa recibe el `value` estructurado. Un listener solo puede reemplazar la copia durable, que el modelo nunca ve. El listener se ejecuta como trabajo rastreado fuera de la ruta de resultado del programa. Cuando hay más de `maxParallelSubCalls` tareas de log pendientes, el bucle de commit ordenado espera, de modo que un backend de spill lento limita los inicios de sub-call posteriores en lugar de acumular E/S pendiente ilimitada. El asentamiento de la ejecución sigue esperando a cada tarea dentro del turno abierto.
- **Política**: `dsh-spill-policy` registra un listener para este evento y usa el mismo código de reemplazo que su listener de resultados de modelo: el mismo límite `maxInlineBytes`, vista previa y localizador, invariante dentro del límite y respaldo de mejor esfuerzo. El artefacto de spill se etiqueta `dispatch` bajo el id de la sub-call. Las UIs y el replay leen su texto completo a través de la misma ruta usada para los resultados nativos derramados, de modo que ambos tipos de resultado se renderizan con la misma información.
- **Una diferencia deliberada**: el listener de resultados de modelo omite `read` para evitar un bucle `read → spill → read otra vez`. El listener del log de despacho también reemplaza el contenido sobredimensionado de las sub-calls de `read`, porque una copia de log no es contexto de modelo, por lo que ese bucle no puede ocurrir, y `read` es la herramienta con más probabilidad de producir una entrada de log grande.

## Alternativas consideradas

**Aplicar un límite de bytes simple dentro del bridge sin almacenamiento de spill.** Rechazada: el truncamiento sin localizador pierde datos que el replay o las UIs pueden necesitar y restaura el renderizado de «resumen truncado» menos informativo que eliminaron cambios anteriores.

**Derramar dentro del bridge directamente llamando a `ctx.spillStore` desde `code-mode.ts`.** Rechazada: el registro exigiría la capacidad de spill. El waterfall mantiene esta política junto con las demás decisiones de spill y permite que las composiciones la omitan; omitir `maxInlineBytes` sigue convirtiendo al listener en un no-op.

**Reutilizar `tools/post-execute` para las llamadas anidadas en lugar de un evento nuevo.** Rechazada: post-execute puede cambiar el resultado visible para el programa, por lo que las llamadas anidadas lo omiten deliberadamente y los programas reciben datos completos. La copia durable necesita un listener separado que se ejecute después de que el programa tenga su valor.

## Consecuencias

Las entradas de despacho de Code Mode en el log de sesión tienen ahora el límite de bytes configurado, y la entrada de Known Limitations del README sobre el registro de despacho ilimitado ahora apunta aquí. Los logs antiguos con contenido de despacho sobredimensionado siguen reproduciéndose porque los campos del evento no han cambiado; solo los añadidos futuros contienen menos texto. La UI web renderiza la salida de las sub-calls derramadas como texto de vista previa y localizador a través de la misma ruta que los resultados nativos, sin ningún caso especial.
