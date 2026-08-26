# Agent Note: Fold the single compaction backend into its service package

Status: rejected — hay más backends de compactación planificados, así que los paquetes de Service Definition y del provider básico siguen separados.

English | [中文](2026-07-19-fold-compaction-package-split.zh.md) | Español

## Problema

La compactación está repartida entre `@deepseek-ai/dsh-compaction`, que es dueño de un servicio abstracto de dos métodos y de los tipos compartidos, y `@deepseek-ai/dsh-compaction-basic`, que es dueño del único provider completo. Las configuraciones enviadas cargan solo el paquete basic, y ningún paquete de producción consume de forma independiente el paquete de Service Definition salvo ese provider.

La separación añade un manifiesto de paquete, un README, una frontera de proyecto, una arista de dependencia, una clase abstracta de reenvío, entradas de catálogo generadas y cableado de composición, sin demostrar ninguna sustitución de backend. La [decisión de seams de capacidad](../../implemented/architecture/2026-06-13-capability-seams.es.md) exige una interfaz, una implementación y un consumidor reales en lugar de una separación preventiva; la [decisión de compactación](../../implemented/feature/2026-06-18-compaction-capability-seam.es.md) registra que su consumidor independiente quedó aplazado.

## Propuesta

Mover la implementación basic a `@deepseek-ai/dsh-compaction` y eliminar `@deepseek-ai/dsh-compaction-basic`. Mantener en un solo paquete `ctx.compaction`, `CompactionResult`, los helpers compartidos de transcript y de emparejamiento de herramientas, la configuración vigente y el algoritmo concreto de compactación.

Preservar `summarize()` como hook protegido de personalización. Un resumidor específico de despliegue puede hacer subclase o interceptar la llamada LLM existente sin exigir un segundo paquete de capacidad. Reintroducir un paquete de Service Definition separado solo cuando un segundo backend completo y un Consumer independiente necesiten sustitución.

Modificar la decisión de compactación ya implementada y la [propuesta de compactación recuperable](../../proposed/feature/2026-07-06-recallable-compaction.es.md) si esta propuesta se acepta, de modo que la propiedad de los paquetes tenga una única descripción duradera.

## Alternativas consideradas

**Mantener la separación porque puede llegar un backend remoto o de recall.** Una implementación futura posible no justifica la frontera actual de paquetes. El recall añade un consumidor de resultados de compactación, no necesariamente otra implementación, y un resumidor remoto puede usar el hook protegido.

**Mover el nombre del paquete del provider al paquete de Service Definition.** Conservar `compaction-basic` como nombre superviviente haría que el servicio del producto pareciera un backend opcional más. `compact` es la identidad estable del servicio ya usada por `ctx.compaction` y es el propietario más claro del paquete único.

## Criterios de aceptación

- `@deepseek-ai/dsh-compaction-basic` y sus metadatos de workspace/paquete se eliminan.
- `@deepseek-ai/dsh-compaction` es dueño de la configuración vigente, la clase de plugin, el algoritmo, los tipos, los eventos y los helpers compartidos.
- Los despliegues existentes pueden cargar el paquete superviviente con configuración y comportamiento visible para el modelo equivalentes.
- La compactación automática y la manual preservan cancelación, bloqueo, contabilidad de tokens, emparejamiento de herramientas, eventos duraderos, seqs citados de los eventos fuente, convergencia de reintentos y renderizado del transcript.
- Pasan las pruebas de composición del loader, unitarias, de turno desbocado, de cancelación, de instantánea y de compactación con modelo real; los catálogos generados y los grafos de módulos quedan al día.

## Riesgos

Esta es una contracción deliberada de nombres de paquete previa al lanzamiento. Los embedders que cargan `@deepseek-ai/dsh-compaction-basic` deben cambiar de paquete, y una futura sustitución de backend exigiría extraer de nuevo una frontera. El costo solo es aceptable mientras exista una implementación completa; la aceptación debería revisarse si un segundo backend aterriza primero.
