# Agent Note: Propiedad de presentación de herramienta del cliente

Status: implemented

[English](2026-08-08-client-tool-presentation-ownership.md) | Español

## Problema

Client Runtime ya emparejaba eventos de llamada/resultado de herramienta por `callId` y podía recuperar la topología raíz/subllamada de eventos Code Dispatch, pero la vista Chat también era propietaria de la ubicación de herramienta en el flujo de conversación, la composición de árbol de llamada recursiva, el despacho por nombre de herramienta, el respaldo genérico, los modelos de tarjeta y los renderizadores de herramienta de primera parte. `ui-conversation` por lo tanto tenía que interpretar cada nombre de herramienta comercial; mover componentes React individuales no cambiaba esa propiedad, y eliminar renderizadores atómicos dejaba las subllamadas sin un propietario de presentación.

La presentación de herramienta necesitaba un propietario independiente sin añadir un segundo registro junto a los slots de cliente ni hacer que cada renderizador de herramienta atómico entendiera la estructura raíz/subllamada.

## Decisión

Herramienta es un concepto de presentación de UI de cliente de primera clase. `@deepseek-ai/dsh-client-ui-tool` es propietario de la composición raíz/subllamada, el despacho de renderizador atómico por nombre de herramienta del wire, el respaldo genérico, los modelos de tarjeta y la salida de detalles. Los plugins comerciales registran solo sus renderizadores atómicos de herramienta y no modifican la conversación ni la sesión.

El ensamblaje de datos de conversación sigue la [decisión de nodo comercial de conversación posterior](2026-08-09-client-conversation-node-assembly.md). La Definición de herramienta de `ui-conversation` empareja eventos de llamada/resultado de sesión raíz, extrae bordes Code Dispatch en bloques `ToolCallBlock.subCalls` recursivos y emite un nodo `tool-call` de Chat estable. Esta responsabilidad de datos maneja solo identidad y topología de herramienta oficial; no interpreta la presentación para nombres de herramienta concretos.

[`ChatView`](../../../../packages/client/ui-conversation/src/client/chat/ChatView.tsx) solo coloca entradas genéricas [`ChatNodeSeat`](../../../../packages/client/ui-conversation/src/client/chat/ChatNodeSeat.tsx) en el orden de instantánea de Chat. Un Seat despacha `'conversation.chat.node'` por `node.kind`; [`ui-tool`](../../../../packages/client/ui-tool/src/client/apply.ts) registra la entrada `tool-call`, y [`ToolCallTree`](../../../../packages/client/ui-tool/src/client/tool/ToolCallTree.tsx) recorre recursivamente el bloque raíz. Cada nivel raíz o hijo despacha a través del mismo slot hijo con clave/sesión `'tool.call.toolview'` con `entryKey: toolName`, cayendo al respaldo `GenericToolCard` cuando no existe ningún registro.

Un plugin de herramienta comercial recibe un `ToolCallBlock` estándar, identidad, cwd del espacio de trabajo y acciones del host; no lee sesión, contexto ni el ensamblador de conversación. Skill sigue siendo una herramienta ordinaria y usa la misma ruta de registro con clave de slot que otras herramientas comerciales.

El panel de detalles es un segundo punto de presentación de herramienta, no el propietario del árbol de llamada. `ui-conversation` localiza la llamada seleccionada y delega su cuerpo de salida mediante `'conversation.details.tool'`; `ui-tool` reutiliza el modelo de tarjeta, mientras que el respaldo de conversación retiene texto de resultado sin formato cuando el plugin está ausente.

## Ruta de tiempo de ejecución y renderizado

```text
Ventana de eventos de sesión
  -> Definición de herramienta -> nodo tool-call de Chat (ToolCallBlock recursivo)
  -> ChatView -> ChatNodeSeat(entryKey = tool-call)
  -> ToolCallTree
       -> recursión root/subCalls[]
       -> tool.call.toolview(entryKey = toolName)
            |- vista atómica registrada
            `- respaldo GenericToolCard
```

## Límite de propiedad

| Propietario | Es propietario de | Explícitamente no es propietario de |
|---|---|---|
| Motor de conversación de Runtime de cliente | Identidad de contexto, ubicación, reproducción de historial, publicación de nodos de vista | Significado de evento de herramienta, árbol de llamada, renderizador de herramienta |
| Definición de herramienta de `ui-conversation` | emparejamiento de llamada/resultado, topología Code Dispatch, `ToolCallBlock` running/settled/interrupted, ancla de ordenamiento de Chat | despacho por nombre de herramienta, modelos de tarjeta, estructura React recursiva |
| vista Chat de `ui-conversation` | orden de nodo con clave, anclas de desplazamiento, selección y acciones del host | ciclo de vida de herramienta, composición de subllamada, renderizadores atómicos de herramienta |
| `ui-tool` | renderizado recursivo raíz/subllamada, despacho con clave atómico, respaldo, modelos de tarjeta y salida de detalles | extracción de evento de sesión, ordenamiento de Chat |
| Plugin de herramienta comercial | renderizadores atómicos para uno o más nombres de herramienta del wire | colocación raíz/subllamada, emparejamiento de ciclo de vida, proyectores de sesión |

## Verificación

Las pruebas de `ui-conversation` fijan el emparejamiento de llamada/resultado de la Definición de herramienta, Code Dispatch, interrupción e identidad con clave running→settled sin importar renderizadores de producción de `ui-tool`. Las pruebas de `ui-tool` montan el host de conversación real y fijan recursión raíz/subllamada, despacho con clave, respaldo genérico, selección, detalles y tarjetas de herramientas concretas. Las pruebas web ensambladas cubren la ruta con ambos plugins cargados.

## Alternativas consideradas

**Mantener slots de herramienta atómicos bajo cada vista de conversación.** Rechazado: cada vista repetiría la composición raíz/subllamada y el registro de herramienta se dividiría por vista. El renderizador de herramienta entero ocupa un slot de nodo comercial en una vista, mientras que herramienta posee el despacho atómico.

**Mover solo componentes React de herramienta y modelos de tarjeta.** Rechazado: la conversación todavía despacharía por nombre de herramienta y recorrería recursivamente subllamadas, así que mover archivos no crearía un límite de propiedad.

**Crear un registro de proyector/extracción específico de herramienta.** Rechazado: el ensamblador de conversación general ya es propietario de identidad de contexto, ventanas de historial y publicación. Un segundo registro de Runtime crearía dos autoridades de ciclo de vida.

**Dejar que cada renderizador de herramienta atómico recorra sus subllamadas recursivamente.** Rechazado: un registrante atómico debería entender una llamada de herramienta sin saber si es raíz o hija. `ToolCallTree` maneja la estructura recursiva una sola vez.

**Dejar que `ui-conversation` importe componentes de `ui-tool` directamente.** Rechazado: esto invertiría la dependencia de características y haría obligatoria la presentación de herramienta. Los slots preservan la carga independiente, el ciclo de vida y el comportamiento de respaldo.

## Consecuencias

`ui-conversation` ya no depende de la presentación para nombres de herramienta concretos, y raíz y subllamadas no pueden derivar a diferentes rutas de despacho. Los paquetes comerciales pueden ser propietarios independientes de renderizadores atómicos de herramienta; si `ui-tool` está ausente, el ensamblaje de datos de conversación sigue siendo válido, los nodos de Chat usan el respaldo genérico y los detalles retienen resultados sin formato.

El costo es una dependencia explícita de `ui-tool` en el slot de nodo comercial y el espacio de nombres de configuración regional declarado por conversación, más un slot hijo específico de herramienta. La Definición de herramienta sigue en `ui-conversation` porque este cambio no divide paquetes; puede moverse más tarde a través de la costura de registro de conversación sin cambiar la propiedad de presentación registrada aquí.