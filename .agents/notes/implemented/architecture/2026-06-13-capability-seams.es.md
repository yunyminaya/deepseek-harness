# Agent Note: Seams de capacidad — roles de Service Definition / Service Provider / Consumer

Status: implemented

[English](2026-06-13-capability-seams.md) | Español

## Problema

El harness tiene capacidades reemplazables — hoy la ejecución de bash, mañana ejecutores en sandbox o remotos y providers de modelos alternativos. Una capacidad tiene tres preocupaciones que cambian a ritmos distintos y por razones distintas: el *contrato* (qué es la capacidad), la *implementación* (cómo se ejecuta) y la *API del consumidor* (contra qué programan el modelo y el resto de plugins). Empaquetarlos juntos en un solo paquete acopla esos ritmos de cambio — sustituir un ejecutor local por uno en sandbox obligaría a modificar los schemas de herramienta que ve el modelo, aunque el contrato orientado al modelo nunca hubiera cambiado.

Esto es distinto de «quién proporciona frente a quién necesita una capacidad en tiempo de ejecución», que Cordis ya resuelve con servicios + `inject` (un provider registra `ctx.shell`; un consumidor declara `inject: ['bash']` y su fiber queda pendiente hasta que el servicio existe). Ese mecanismo es necesario pero no dicta los límites entre paquetes; este Agent Note sí lo hace.

## Decisión

Una capacidad reemplazable tiene **tres roles**:

1. **Service Definition** — el `Service` de Cordis y los tipos de vocabulario que son dueños de `ctx.<key>` y dependen solo del vocabulario que el contrato necesita (p. ej. `dsh-shell`: `ShellExecutor`, `ShellRunResult`, `ShellProcess`). Una definición puede ser una clase abstracta o un servicio de registro concreto; nunca es un `interface` de TypeScript.
2. **Service Provider** — un plugin que suministra o registra una implementación (p. ej. `dsh-bash-local`: subprocesos, muertes de grupos de procesos, truncado de spill files). Los providers en sandbox y remotos son paquetes hermanos que implementan o se registran contra la misma Service Definition.
3. **Consumer** — aquello contra lo que programan el modelo y los plugins (p. ej. `dsh-tool-bash`: el schema `bash`, con los handles de segundo plano registrados en el runtime genérico de trabajos). Los Consumers inyectan la clave de servicio y nunca importan tipos específicos del provider.

Los nombres de los roles usan mayúscula inicial: **Service Definition**, **Service Provider** y **Consumer**. Los usos genéricos de `provider` y `consumer` permanecen en minúscula.

Los Service Providers y los Consumers evolucionan entonces de forma independiente: un ejecutor en sandbox reemplaza a `dsh-bash-local` sin tocar ningún schema de herramienta.

Normalmente los roles usan paquetes separados cuando evolucionan de forma independiente, pero la separación no es obligatoria cuando los roles son de verdad un solo asunto: el seam de LLM pliega la Service Definition y el Consumer en `dsh-llm` (el Consumer es el propio bucle, no una superficie de schema reemplazable) con los adaptadores como paquetes de Service Provider. No dividas preventivamente — una capacidad con un solo provider concebible y un solo Consumer sigue siendo un solo paquete hasta que aparece un segundo.

## Terminología: «seam» nombra el trío, no la interfaz

Un **seam** es la capacidad completa — los tres roles juntos: una **Service Definition** (el `Service` de Cordis dueño de `ctx.<key>` y del vocabulario), uno o más **Service Providers** y uno o más **Consumers**. `packages/shell` es el ejemplo canónico — `dsh-shell` / `dsh-bash-local`+`dsh-bash-sandbox` / `dsh-tool-bash`. Un paquete puede ser dueño de varios roles, pero un solo rol no es el seam. El término «seam» se reserva para esta capacidad completa; nombra cada componente por su rol, clase, servicio, contrato o punto de extensión. El [glosario](../../../../docs/glossary.es.md#capability-seam) es la entrada canónica.

## Alternativas consideradas

- **Combinar siempre los roles** — rechazado porque vuelve a acoplar Service Definitions, providers y Consumers que cambian de forma independiente.
- **`@cordisjs/plugin-capability`** — un eje completamente distinto: es un servicio de *seguridad* de permisos/capacidades (permisos con nombre con herencia, probados contra una sesión mediante `ctx.capability.test`), candidato para el trabajo diferido de permisos/sandbox en la puerta deny/ask de `tools/pre-execute`, NO un mecanismo para intercambiar implementaciones. Confundir los dos («capability») es la trampa que nombra este Agent Note.

## Consecuencias

Separar los roles añade paquetes y código repetitivo (`package.json`, `tsconfig`, README y el cableado de inyección). A cambio, los Service Providers y los Consumers se publican y versionan de forma independiente, y un nuevo backend nunca pone en riesgo el contrato orientado al modelo. [AGENTS.md](../../../../AGENTS.md) y [architecture.md](../../../../docs/architecture.es.md) recogen la regla; el trío de bash es la plantilla de referencia. Este Agent Note registra por qué los roles que cambian de forma independiente normalmente se separan, mientras que los asuntos genuinamente compartidos pueden permanecer plegados.
