# Agent Note: Estado de colaboración específico del plan

Status: implemented

[English](2026-07-22-plan-specific-collaboration-state.md) | Español

## Problema

La primera implementación del plan-mode introdujo un registro genérico de modos con nombre aunque el producto solo publicara `plan`. `ModeConfig.modes`, la validación de nombres de definición, `ctx.modes.list()`, el fallback de definiciones retiradas y un modo `review` sintético en los tests existían solo para soportar hipotéticos modos de colaboración futuros. El comportamiento específico de producción — la guía de plan, `/plan` y `exit_plan_mode` — seguía viviendo en el mismo paquete, así que la API genérica no aislaba un mecanismo reutilizable de la política de plan.

La palabra «mode» también abarca dominios no relacionados. El modo sandbox es una política de aplicación propiedad de `ctx.sandboxPolicy` y registrada como `sandbox/mode`; el plan mode es una postura de colaboración que contribuye guía y una salida revisada. Tratar ambos como instancias de una única abstracción de modos con nombre oscurecería su propiedad independiente. El vocabulario genérico de un transporte no es evidencia de que el harness necesite un dominio de modos genérico.

El plan mode también necesita una postura durable, un artefacto de plan revisable, un límite humano explícito y reconstrucción del request a través de reanudación y fork. Esos requisitos pertenecen a la funcionalidad de plan incluso después de eliminar el registro genérico y las proyecciones ACP interactivas.

## Decisión

El plan mode es propietario de un paquete de producto específico del plan: `@deepseek-ai/dsh-plan-mode` en `packages/plan/plan-mode/`. El hecho durable es `plan/mode: { active: boolean }`, plegado por `foldPlanMode(events)` con `false` como valor de log vacío. `ctx.planMode.get(agent)` devuelve `{ active, pending? }`, y `set(agent, active)` registra la selección aplicada en el límite. Las vallas de pre-step, retry, fallo de append y destrucción preservan la misma propiedad de transición de estado.

La configuración es exactamente `{ section: string }`. El paquete registra la sección fija `plan:policy`, `/plan [message]`, la forma exacta de salida directa `/plan off` y `exit_plan_mode` en sí. `/plan` a secas selecciona activo; cualquier otro argumento no vacío lo selecciona primero y luego envía el texto recortado a través de `agent.steer()`, convirtiendo el texto en un mensaje de usuario ordinario registrado en el paso afectado. `/plan off` selecciona inactivo sin entrada de modelo y puede cancelar una entrada que aún está pendiente en el límite. La herramienta de salida permanece registrada mientras el plan mode está inactivo para que el catálogo de herramientas del request se mantenga estable.

Las composiciones orientadas a humanos son propietarias de la selección y revisión del plan. Esta nota mantenía originalmente el selector `default`/`plan` a nivel de protocolo de ACP como un adaptador sobre el servicio booleano; [ACP como protocolo solo de automatización](2026-07-23-acp-automation-only-protocol.es.md) reemplaza esa proyección en el cable, así que la composición ACP ahora no monta ni plan mode ni un protocolo de selección de modos.

El modo sandbox y la política de aprobación siguen siendo ejes de aplicación separados. El plan mode ni los lee ni los escribe, y la simplificación no introduce ningún tipo base compartido, registro ni abstracción de preset entre esos conceptos.

### Contrato de límite y de modelo

`plan/mode` es solo-log y no-superficie, así que la reanudación, el fork y la compactación recuperan el estado sin un espejo en vivo. Un agente generado empieza inactivo porque no existe ninguna opción de plan en tiempo de creación. Las selecciones de usuario pendientes se vacían antes del ensamblado del request afectado en el pre-step inicial o de continuación, o en un retry de recuperación de request; un append durable fallido deja la intención pendiente para un límite posterior.

El estado activo contribuye la sección del despliegue en el orden de prompt 50. El estado inactivo no contribuye ninguna sección, mientras que `exit_plan_mode` permanece registrado en ambos estados, así que una transición cambia la cabecera de request registrada pero no los schemas nativos de herramienta ni el SDK de Code Mode. Una transición impulsada por el usuario añade un único aviso de origen plugin solo cuando la última cabecera de request describía el estado opuesto; una selección previa al primer request o de suma neta cero no añade ninguno, y una salida de herramienta aprobada se apoya en su resultado de herramienta en lugar de un segundo aviso.

### Salida revisada

`exit_plan_mode` exige un agente llamador en plan mode activo y un plan markdown no vacío que comience con un encabezado. La pregunta de user-questions lleva ese plan exacto como detalle y ofrece `Approve` o `Keep planning` más comentarios de texto libre. Solo una selección `Approve` sin texto personalizado consiente; cualquier otra respuesta permanece en plan mode y devuelve comentarios correctivos al modelo. Una salida aprobada se convierte en una selección pendiente silenciosa, dejando la guía de plan activa para el resto del lote de herramientas actual y eliminándola antes del siguiente request.

La herramienta renderiza el plan enviado como una tarjeta genérica titulada por su primer encabezado. Un provider de user-questions ausente o fallido, una revisión fallida o la destrucción del plugin mientras la revisión está pendiente fallan de forma segura y dejan `/plan off` manual como vía de escape humana.

## API eliminada

- El mapa arbitrario de definiciones, la expresión regular de nombres de modo, las reglas de nombres reservados y el bucle de comandos por definición.
- `ModeDefinition`, el mapa de definiciones resuelto, `ctx.modes.list()`, el estado get/set con valor de cadena y el manejo de modos desconocidos o retirados.
- Los casos de modo `review` solo-de-tests y las afirmaciones de que se pueden añadir más modos mediante configuración.
- Los nombres genéricos `mode/set` y `mode:policy`; el paquete de plan ahora es propietario de `plan/mode` y `plan:policy`.

## Alternativas consideradas

**Conservar un registro genérico privado y exponer solo plan hoy.** Rechazado porque la maquinaria de nombres/config sin uso seguiría manteniéndose y probándose sin un segundo consumidor de producción. Un estado de colaboración futuro puede establecer el seam compartido correcto a partir de dos casos concretos.

**Plegar la política de sandbox o de aprobación en el estado de plan.** Rechazado porque la guía de colaboración, el confinamiento de ejecución y las decisiones de permiso tienen propietarios, semánticas de ciclo de vida y consumidores distintos. Un tope de sandbox propiedad del modo también hace que una selección de sandbox explícita del usuario parezca tener éxito mientras no hace nada en silencio.

**Dejar que un transporte de presentación sea propietario del estado de plan.** Rechazado porque TUI, Web, reanudación, fork, ensamblado de prompt y la herramienta de salida necesitan el mismo hecho registrado con independencia de cualquier transporte individual. Los adaptadores de presentación son propietarios solo de sus proyecciones.

**Dividir un trío de capability-seam o poner el estado en el agent loop.** Rechazado porque el plan mode no tiene backend intercambiable, mientras que los puntos de extensión existentes de sesión, prompt, herramienta, comando y ciclo de vida ya proporcionan todos los hooks necesarios.

**Poner los flips en mensajes de superficie o almacenar los planes en archivos.** Rechazado porque la postura es un hecho solo-log y el argumento de la herramienta ya registra el plan revisable. La duplicación en superficie gasta contexto de modelo, mientras que un directorio de planes crea un segundo hogar durable.

**Filtrar herramientas por una allowlist de nombres por plan o una pila de políticas global.** Rechazado porque la mutabilidad es una propiedad de cada herramienta, incluidas las futuras y las MCP, en lugar de una lista que cada despliegue de plan deba mantener. Los metadatos de efectos pueden establecer una política compartida solo cuando exista un consumidor concreto; hasta entonces, el plan mode es guía, no un límite de seguridad.

**Revisar a través del seam de aprobación o en prosa.** Rechazado porque una revisión de plan no es una decisión de permiso, necesita el artefacto exacto y el texto libre correctivo, y debe tener una llamada de herramienta registrada como su transición estructurada. El seam de user-questions suministra ese contrato.

## Verificación

- Los tests de paquete conservan el orden de límites, el retry, el fallo de append, la destrucción HMR, el ensamblado de prompt, los schemas nativos y de Code Mode estables, los resultados de revisión y la cobertura de invariantes a través del servicio booleano.
- Los tests de comandos cubren `/plan` a secas, `/plan <message>`, `/plan off` activo, la cancelación de entrada pendiente, la idempotencia inactiva, la ausencia de `/mode` y `/review`, y la eliminación con alcance de efectos.
- Los escenarios TUI sin clave entran por `/plan <message>`, salen por `/plan off`, y prueban que cada `plan/mode` confirmado precede a la cabecera de request que cambia, que el mensaje de entrada se registra bajo la guía de plan y que el request posterior a la salida omite esa guía.
- El arco completo de revisión de `exit_plan_mode` está probado a nivel de paquete pero no tiene instantánea de aplicación ensamblada después de retirar los escenarios ACP interactivos; los escenarios TUI actuales sin clave cubren solo la entrada de comandos y la salida directa.

## Consecuencias

La implementación tiene un vocabulario para una funcionalidad publicada. Añadir otra postura de colaboración es una decisión de diseño explícita en lugar de una entrada de configuración, y los clientes de automatización no adquieren controles de modo humano a través de ACP. La migración rechaza intencionadamente los logs antiguos de `mode/set` y la configuración antigua de `modes.plan.section` bajo la política de formato pre-release del repositorio.

El estado de plan sigue siendo reconstruible y los schemas de herramientas siguen siendo estables, pero una selección pendiente inactiva se pierde si el proceso sale antes del siguiente límite. Entrar o salir del plan mode cambia el prompt desde el orden 50 en adelante, y un modelo que ignora la guía aún puede mutar a menos que el despliegue configure por separado la política de sandbox, aprobación o sistema de archivos.
