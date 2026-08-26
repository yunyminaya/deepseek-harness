# Agent Note: Cancelación requerida a través de seams de capacidad alcanzables por herramientas

Status: proposed

[English](2026-07-19-required-cancellation-through-tool-capability-seams.md) | Español

## Problema

El [contrato de cancelación del registro de herramientas](../../implemented/architecture/2026-07-19-cooperative-tool-cancellation.es.md) implementado hace `exec.signal` requerido en todo cuerpo de herramienta, pero muchas interfaces de capacidad asíncronas alcanzadas desde esos cuerpos siguen aceptando una señal opcional. Una herramienta puede por tanto satisfacer su propio tipo mientras deja caer accidentalmente la cancelación en la siguiente llamada del mismo proceso.

Esa brecha es transitiva. Una herramienta de filesystem puede llamar a la resolución de rutas y al I/O, una herramienta web puede llamar a un provider, una herramienta bash puede llamar a un ejecutor, y una herramienta compuesta puede iniciar o esperar tareas, subagentes o flujos de trabajo. Si cualquier operación await que controla trabajo propiedad de la herramienta acepta la omisión, TypeScript no puede probar que la cancelación sigue disponible en el límite que es dueño del efecto secundario.

Requerir señales en cada función asíncrona del repositorio sería excederse. Algunas operaciones no son alcanzables desde herramientas, algunas consultas síncronas no pueden esperar ni ser dueñas de trabajo en curso, y el trabajo explícitamente desprendido tiene un nuevo dueño tras una entrega deliberada.

## Propuesta

Requerir un `AbortSignal` en cada operación de capacidad asíncrona del mismo proceso que sea alcanzable desde un cuerpo de herramienta mientras la herramienta aún es dueña de la operación o la espera. El requisito puede ser un parámetro posicional o un campo de petición de solo lectura requerido según la forma existente del seam dueño, pero la omisión debe fallar la compilación de TypeScript.

Cada llamante directo suministra una señal de la que es dueño o que propaga desde su propio contexto de operación requerido. Las implementaciones pueden derivar un deadline hijo o un ámbito de cancelación, pero la señal derivada permanece vinculada a la señal aguas arriba durante la vida delegada. Las implementaciones de capacidad no sintetizan señales de nunca-abortar, no usan cancelación ambiental async-local, ni validan `AbortSignal` en runtime solo para repetir el contrato tipado del mismo proceso.

La migración comienza con un inventario desde cada `ToolDefinition.execute()` de primera parte a través de las llamadas de capacidad que espera. Luego cambia junta cada seam coherente de Service Definition / Service Provider / Consumer, incluidos tests y documentación de API generada. PRs separados pueden migrar las familias de filesystem, shell/task, web/provider, workflow/subagent, code-runtime y similares para que cada cambio siga siendo revisable, pero ninguna interfaz migrada conserva una sobrecarga opcional de compatibilidad bajo la política pre-release del repositorio.

### Límite de alcance

La propuesta incluye operaciones de capacidad asíncronas cuya finalización o cancelación sigue siendo parte de la vida de la herramienta invocante, incluidas las operaciones de inicio antes de la transferencia de titularidad, la ejecución en primer plano, las lecturas y escrituras, las peticiones de provider, las esperas y la limpieza o el dispose que la herramienta espera.

La propuesta excluye la búsqueda de registro síncrona, las comprobaciones de disponibilidad, el renderizado de schemas, la clasificación de argumentos y otras operaciones que no pueden retener trabajo asíncrono. También excluye el trabajo tras una entrega explícita de titularidad desprendida: una vez que una tarea, workflow, worker o agent hijo se ha publicado con éxito a un nuevo dueño de ciclo de vida, el controller de ese dueño gobierna la vida desprendida. La operación de inicio iniciante sigue requiriendo la señal del llamante hasta que la entrega se compromete, y cualquier llamada posterior de herramienta que espere trabajo desprendido requiere su propia señal de invocación.

La cancelación opcional puede permanecer en entradas de parser, config, JSON de modelo/herramienta, formato durable/de archivo, worker, proceso o cable cuando el protocolo externo la hace opcional. El límite dueño debe resolver esa entrada en una señal requerida del mismo proceso antes de llamar a un seam de capacidad migrado.

## Alternativas consideradas

**Dejar las señales aguas abajo opcionales porque los cuerpos de herramienta ahora reciben una.** Rechazado porque la disponibilidad en el callback externo no hace la propagación type-safe; la omisión sigue siendo legal en cada llamada de capacidad opcional.

**Imponer la propagación con reglas de lint o inspección de callbacks.** Rechazado porque las comprobaciones de sintaxis no pueden identificar de forma fiable titularidad, señales derivadas, capas de abstracción ni el settlement quiescente correcto. Los parámetros de interfaz requeridos expresan el contrato donde TypeScript puede comprobar cada llamante.

**Pasar `ToolRunContext` a través de cada capacidad.** Rechazado porque las capacidades necesitan cancelación, no identidad de herramienta, estado de agent ni diferimiento de contexto. Pasar el contexto mayor acopla servicios reutilizables al registro de herramientas y oscurece el seam estrecho.

**Usar una señal ambiental async-local.** Rechazado porque la propagación oculta hace difícil auditar la titularidad y la entrega desprendida, complica los tests y deja que llamadas se vinculen silenciosamente a la vida equivocada.

**Añadir señales por defecto o de nunca-abortar en las implementaciones de capacidad.** Rechazado porque los valores por defecto borran al dueño faltante en lugar de exponerlo en compilación.

**Migrar cada capacidad en el cambio implementado del registro de herramientas.** Rechazado porque los cambios de interfaz transitivos abarcan familias de capacidad independientes. Mantener esta propuesta separada preserva la decisión del registro implementado y deja que cada seam profundo migre con tests enfocados.

## Criterios de aceptación

- Un inventario mapea cada cuerpo de herramienta de primera parte a las operaciones de capacidad asíncronas que puede alcanzar antes de la entrega de titularidad.
- Cada interfaz de capacidad en alcance requiere `AbortSignal`, y los tests de contrato de compilación prueban que la omisión falla.
- Interfaz, implementación, consumidor directo, helper de test, ejemplo y referencias de API generada migran juntos sin sobrecargas de compatibilidad ni centinelas de nunca-abortar de producción.
- Los deadlines derivados y los ámbitos wrapper permanecen vinculados a la señal del llamante, y los tests de integración prueban que la cancelación llega al dueño del efecto secundario y que el trabajo esperado alcanza la quiescencia.
- Las consultas síncronas y el trabajo explícitamente desprendido post-entrega permanecen fuera del requisito, con las transiciones de titularidad documentadas y probadas donde existe ambigüedad.
- La validación de runtime se añade solo en un límite realmente sin tipar, no para repetir un campo o parámetro de TypeScript requerido.
- El typecheck de nivel superior, la cobertura, las instantáneas, la documentación, el module-graph, el build, la higiene, los demos y las puertas de artefactos construidos pasan después de cada migración coherente.

## Riesgos

**Radio de explosión transitivo grande.** Un parámetro requerido puede exponer de golpe a muchos llamantes directos. Migra por familia de capacidad coherente y usa los fallos de typecheck como el inventario completo de llamantes.

**Clasificación incorrecta del trabajo desprendido.** Excluir una operación de inicio demasiado pronto puede desprender trabajo antes de que la publicación se comprometa; requerir la señal del padre para siempre puede dejar que una herramienta completada cancele trabajo legítimamente desprendido. Cada entrega necesita un punto de commit explícito, un nuevo dueño, un comportamiento de rollback y un camino de fallo quiescente.

**Confusión de titularidad de la señal.** Una capacidad que almacena una señal prestada más allá de la vida delegada puede vincular trabajo a un llamante obsoleto. Las interfaces y los tests deben distinguir señales de operación prestadas de controllers propiedad de servicios de larga vida.

**Cumplimiento mecánico sin cooperación.** Un parámetro requerido prueba la disponibilidad, no la observación ni el reenvío. Los tests de integración en los límites de proceso, worker, socket, provider y tarea siguen siendo necesarios para probar el comportamiento.

**Sobre-alcance de APIs síncronas o no relacionadas.** Requerir cancelación donde no existe trabajo asíncrono añade ruido y debilita la señal del contrato. El inventario registra por qué cada operación es alcanzable por herramientas y portadora de vida antes de cambiarla.
