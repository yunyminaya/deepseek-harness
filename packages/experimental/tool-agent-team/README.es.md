# @deepseek-ai/dsh-experimental-tool-agent-team

[English](README.md) | Español

Adaptador orientado al modelo y con ámbito para [`ctx.agentTeams`](../agent-team/README.es.md). Instala la política de Agent Teams y las herramientas de colaboración en cada ámbito de Lead implícito y de compañero de equipo persistente. Las definiciones de Team con ámbito sombrean los controles globales de subagente continuable heredados con el mismo nombre, de modo que una composición que monte ambos debe deshabilitar las definiciones heredadas.

## Config

```yaml
- id: tool-agent-team
  name: '@deepseek-ai/dsh-experimental-tool-agent-team'
  config:
    freshProvider: spawn
    forkProvider: fork
```

`freshProvider` y `forkProvider` seleccionan providers de subagente continuable registrados. La política de modelo fija crea compañeros de equipo solo cuando la persona usuaria pide explícitamente Agent Teams o compañeros de equipo.

## Herramientas y autoridad

El [catálogo de herramientas](../../../docs/tool-catalog.es.md#deepseek-aidsh-experimental-tool-agent-team) generado es dueño de los schemas exactos. El adaptador aporta la creación de compañeros de equipo; la entrega de pares silenciosa y de despertar; el listado de la plantilla, la espera y la interrupción solo del Lead; y las operaciones de creación/listado/consulta y de actualización de comparar-y-establecer de tareas.

Cada herramienta exige el Agent que llama exacto. `spawn_teammate` e `interrupt_agent` imponen la autoridad del Lead dentro de `ctx.agentTeams`, no solo en sus descripciones. Todos los miembros pueden comunicarse con cualquier par y usar el tablero de tareas. Las mutaciones de tarea conservan las comprobaciones de propietario/Lead y de revisión del dominio.

`send_message` tiene éxito una vez que el correo es persistente y nunca despierta un destino inactivo. `followup_task` además convierte el mensaje en el siguiente turno del destino y puede reanudarlo en frío. Un resultado `queued` es trabajo persistente aceptado y no debe reintentarse. La disponibilidad de una tarea no inicia a una persona propietaria. Antes de armar su espera de arista de entre 10,000 y 3,600,000 milisegundos, `wait_agent` comprueba si hay otro miembro en ejecución o en aprovisionamiento; si no lo hay, devuelve `noProgress` de inmediato con instrucciones de volver a listar y usar `followup_task`. De lo contrario, espera una arista del Team posterior a la llamada, con 30,000 milisegundos por defecto, y los llamadores vuelven a listar tras el despertar o el agotamiento del tiempo porque los cambios anteriores no se reproducen.

El plugin escucha la publicación de Agent e instala sus registros a través del ámbito de ese Agent. La creación nueva y la reanudación en frío reciben por tanto el mismo conjunto de herramientas/prompts antes de la primera solicitud del modelo. La liberación del Agent y el HMR del plugin eliminan cada registro con ámbito; recargar el plugin instala un conjunto nuevo en cada miembro aún en vivo sin cambiar su Activation de continuación.

## Experiencia del modelo

### Política del Team y herramientas

#### Qué ve el modelo

Una sección de política estable declara el rol/nombre/id exactos del Team, el requisito de delegación explícita, el comportamiento de cwd compartido, la recuperación de versiones obsoletas del sistema de archivos, el riesgo de Bash/formateadores/generadores de código, la coordinación de tareas y ámbitos de escritura, la entrega silenciosa frente a la de despertar, la regla del buzón de no reintentar y el deber del Lead de esperar antes de responder. Los diez schemas de Team, de `spawn_teammate` a `team_task_update`, aparecen solo en los ámbitos de los miembros del Team.

#### Efecto de tokens

La política y el schema fijos cuestan en cada solicitud de miembro del Team. Las llamadas de herramienta añaden resultados JSON compactos de plantilla, tarea, espera o recibo. El contenido de los pares lo conserva el dominio del Team en el historial del destino.

#### Efecto de KV Cache

Estable en el prefijo mientras la generación del plugin del Team, la configuración, el rol/nombre del miembro y los schemas permanezcan sin cambios. La línea de identidad por miembro difiere entre Agents. Los resultados de herramienta y los mensajes de par se anexan después del prefijo de solicitud reutilizable.

## Limitaciones conocidas y trabajo aplazado

- **La política de prompt es coordinación, no confinamiento** — no puede impedir que Bash o los procesos externos escriban archivos solapados.
- **Sin creación autónoma de equipos** — las tareas ordinarias no desencadenan delegación salvo que la persona usuaria lo pida explícitamente.
- **Sin controles Web** — la presentación de la plantilla y del tablero de tareas en el navegador queda fuera de este paquete de runtime.
