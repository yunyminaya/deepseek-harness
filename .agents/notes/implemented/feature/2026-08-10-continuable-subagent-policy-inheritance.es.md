# Agent Note: Herencia de política de subagentes continuables — el log durable del hijo es dueño de la instantánea en tiempo de delegación

Status: implemented

[English](2026-08-10-continuable-subagent-policy-inheritance.md) | Español

## Problema

El driver en proceso de un solo disparo siembra en sus hijos las anulaciones de sandbox/aprobación del padre desde la [decisión de herencia de política en proceso](2026-07-25-subagent-policy-inheritance.es.md), pero la ruta continuable nunca lo hizo: la materialización de `SubagentContinuationManager` aplicaba solo la composición del hijo y el registro de activación de setup. El bundle por defecto cablea ambas herramientas de delegación como `backgroundMode: continuable`, así que en un despliegue por defecto todo hijo de background caía silenciosamente en los valores por defecto del despliegue — un padre cambiado a `danger-full-access` producía hijos atascados en `workspace-write` cuya operación fuera del workspace levantaba una solicitud de aprobación, y la postura de aprobación `'never'` sin supervisión de un padre revertía a pedir confirmación ([dsh-external/issues#334](https://github.com/dsh-external/issues/issues/334)).

## Decisión

El par captura/append se movió del driver de un solo disparo al módulo hijo compartido del seam (`dsh-subagent/src/child-agent.ts`), el único hogar declarado de la composición hijo compartida: `captureDelegatedPolicyOverrides(parent)` hace una instantánea de `sandboxPolicy.overrideOf(parent.session)` a través del `ctx.get` opcional y fija la política de aprobación del hijo en `'never'` ([decisión de aprobaciones fijadas](2026-08-10-subagent-approval-pinned-never.es.md)), y `appendDelegatedPolicyOverrides(childSession, overrides)` añade los eventos `source: 'delegation'`. El driver de un solo disparo y el gestor de continuación las llaman a ambas, así que las dos rutas no pueden divergir.

`startContinuable` captura antes de su primer await (`prepareContinuable`), el mismo límite de «un cambio posterior del padre pertenece al futuro del padre» que el de un solo disparo. La instantánea viaja en `MaterializeInputs.create`, así que solo la materialización nueva añade los eventos durante el setup no publicado, después de cualquier seed de fork. Una reanudación en frío no pasa entradas `create` y no añade nada: el log del hijo persistido ya porta los eventos de delegación, y reproducir el log ES el estado. El log durable del hijo — no la Activation actual, no el padre que reanuda — es dueño de la política efectiva del hijo, así que un cambio del padre entre épocas de residencia nunca cambia retroactivamente a un hijo durable.

## Alternativas consideradas

- **Una contribución al registro de activación de setup** (`registerContinuableSetup`) — rechazada: una contribución recibe solo el contexto del hijo, así que no puede capturar las anulaciones del padre en el límite de delegación; el registro se aplica en la reanudación en frío además de en la creación nueva, lo que re-añadiría o re-capturaría; y nada ata la captura de una contribución al prefijo síncrono de la llamada start, así que se perdería la garantía de captura previa al await.
- **Re-capturar las anulaciones del padre en la reanudación en frío** — rechazada: un hijo reanudado cambiaría de política en silencio con los cambios posteriores del padre, rompiendo la semántica de instantánea en la delegación y haciendo que la política efectiva dependa del momento de reanudación en lugar del propio log del hijo. Un padre que quiera un hijo reanudado bajo una política nueva vuelve a delegar.
- **Importar la lógica inline del driver de un solo disparo desde el gestor de continuación** — rechazada: el paquete de Service Definition no puede depender de su propio paquete de provider, y duplicar el par captura/append en `continuation.ts` invita a la deriva; `child-agent.ts` ya contiene todos los demás pasos de composición compartidos.
- **Sembrar los eventos en el turno de seed del descriptor** — rechazada: el valor de captura no se conoce cuando se ensambla el seed para cada llamador, y el precedente de un solo disparo ya establece los appends de setup no publicado como el orden que coloca los hechos heredados tras el historial de fork con `firstLiveSeq` intacto.

## Consecuencias

- La delegación de background del bundle por defecto (`backgroundMode: continuable`) ahora hereda una anulación de sandbox explícita del padre y fija al hijo en aprobaciones `'never'`; las composiciones sin ninguno de los dos servicios de política se comportan sin cambios.
- `dsh-subagent` gana tipos de peer opcionales sobre `dsh-sandbox-policy` y `dsh-user-approval` (el patrón `ctx.get` que usaba el driver de un solo disparo); `dsh-subagent-in-process-driver` elimina por completo sus peers de servicio de política e imports de tipos y delega en los helpers compartidos.
- La suite continuable (`packages/subagent/subagent/tests/continuation-inheritance.spec.ts`) fija el seeding de inicio nuevo, la captura previa al await, la omisión por defecto, la estabilidad de la instantánea en reanudación en frío y la precedencia del seed de fork; el escenario de instantánea ACP `subagent-continuable-inheritance` fija el evento de delegación del hijo y el contexto de ejecución de solo lectura a través de la aplicación ensamblada y falla cuando se elimina la captura.
- Los providers fuera de proceso (`acp`, `dsh-sdk`, `claude-code`, `codex`) no soportan hijos continuables (`prepareContinuable` ausente), y sus hijos de un solo disparo conservan su propia política de despliegue (`inheritsParentContext = false`); la propagación de política entre procesos sigue fuera de alcance.
