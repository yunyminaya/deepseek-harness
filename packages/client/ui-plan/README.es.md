# @deepseek-ai/dsh-client-ui-plan

[English](README.md) | [中文](README.zh.md) | Español

Chip de estado del modo plan, un plugin de superficie puramente de navegador. La mitad de navegador ocupa el asiento único `conversation.input.plan` declarado por la conversación (a la derecha del control de modo de acceso); la mitad de nodo es un apply vacío (la fila del roster). El comportamiento del plan en sí — el comando `/plan`, el estado `plan/mode` confirmado por límite o por inactividad, la unidad de proyección `plan` y la sección de política — es propiedad de [`@deepseek-ai/dsh-plan-mode`](../../plan/plan-mode/README.md), compuesto de forma independiente en el roster del host.

El modo plan se entra por la ruta del comando `/plan`: los usuarios pueden elegir Plan en el menú de Comandos `+` del compositor o teclear `/plan`, mientras que este paquete no renderiza ningún control de plan inactivo. Mientras el objetivo efectivo de la proyección `plan` calculada por el host es el modo plan (`pending ? !active : active` — un valor plegado del host, no optimismo del cliente, así que un frame que llega corrige el chip en cualquier dirección), el asiento renderiza el botón de estado «Plan ×» en color de advertencia, que ejecuta `/plan off` a través de `command.execute`; si no, el asiento permanece vacío — un host sin modo plan (o un Draft sin sesión) no muestra nada. Mientras el modo plan es el objetivo efectivo, el placeholder del textarea del compositor cambia a la pista de tarea de plan — «describe your task to generate plan», localizada a través del espacio de nombres de locale `conversation` de ui-conversation (las claves `placeholder.plan` / `hint.plan`) y compartida verbatim con la pista del comando `/plan` reclamado (renderizada por el compositor desde la misma proyección; los placeholders aportados por el dueño ganan).

El chip lleva la descripción accesible «Plan mode on, press to turn off». Los fallos de admisión (`matched: false`, errores de negocio, fallos de transporte) afloran como un error en línea y el chip permanece hasta que la proyección confirma la salida.

El modelo sale del modo plan a través de la herramienta estable `exit_plan_mode`; su revisión de plan usa el canal de preguntas Web compuesto.

## Experiencia de modelo

Indirectamente, a través de la línea de comando `/plan off` que despacha el chip: `@deepseek-ai/dsh-plan-mode` es dueño de la sección de política visible para el modelo, del schema de la herramienta de salida y del estado registrado que impulsa esa línea, mientras que este paquete solo renderiza la proyección y envía lo que un usuario podría teclear igualmente.

#### Efecto en KV Cache

Entrar o salir del modo plan cambia la sección activa `plan:policy` del system-prompt y por tanto el prefijo de la solicitud; el chip en sí no añade contenido al prompt.

## Limitaciones conocidas y trabajo diferido

- **El modo plan es orientación, no un sandbox de ejecución** — los despliegues que requieren planificación de solo lectura forzada deben componer las políticas independientes de sandbox y aprobación.
- **El chip pertenece al compositor por defecto** — una interacción pendiente de todo el compositor, como la revisión de plan, reemplaza temporalmente la InputBar y su chip.
- **Sin control de plan inactivo** — la entrada usa la fuente de Comandos compartida; una sesión con la capacidad pero en modo inactivo no muestra ningún affordance de plan en la fila de herramientas.
