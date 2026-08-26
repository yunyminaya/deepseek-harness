# Agent Note: Eliminar los agentes stdio y Echo

Status: implemented

[English](2026-07-20-remove-stdio-and-echo-agents.md) | Español

## Problema

DeepSeek Harness exponía dos agentes de producto redundantes junto a los coding agents TUI y Headless. El agente stdio orientado a líneas duplicaba la interacción de terminal y la ejecución no interactiva con un protocolo mixto de prompt/salida. Echo duplicaba Headless como modelo mock sin red más una herramienta didáctica, convirtiendo un fixture de prueba en un agente orientado al usuario y en la vía de inicio rápido por defecto.

Ambos agentes llevaban superficies de soporte más allá de sus configuraciones hoja. Stdio era propietario de un plugin de UI, un paquete de app, una interfaz de SDK, una hoja REPL, un protocolo de prompt y tests del Loader. Echo era propietario de un comando ejecutable, un adaptador mock, una herramienta, una puerta de CI de demo, una entrada de grafo, referencias didácticas y un fixture de prueba compartido. Conservar cualquiera de esas vías de producto preservaría el agente redundante indirectamente.

La entrada y salida estándar siguen siendo límites de protocolo para ACP, JSON-RPC, MCP y los procesos hijo. Los adaptadores de modelo deterministas también siguen siendo válidos dentro de las pruebas. Esos mecanismos no justifican un agente de producto orientado a líneas o solo-mock.

## Decisión

Los agentes stdio y Echo se eliminan sin paquetes de compatibilidad, modos, comandos ni alias. La UI stdio y los paquetes de app, `examples/repl-agent`, `examples/echo-agent`, `demo:repl`, `demo:echo`, sus tests dedicados y los manifests, puertas, grafos y entradas de documentación de soporte se eliminan.

Los roles de aplicación restantes son explícitos:

- `@deepseek-ai/dsh-tui` es propietario de la ejecución interactiva de terminal. Rechaza los streams no-TTY antes del arranque del Loader; `apps/cli/config/base.cordis.yml` más el overlay `tui.cordis.yml` son propietarios de la composición de coding completa, con cobertura de PTY más instantánea de terminal en `apps/cli/tests/`.
- [`dsh --profile headless`](../../../../apps/cli/README.md) es propietario de la ejecución no interactiva. Su perfil `headless` es la composición de producto; `examples/headless-agent` es propietario de las instantáneas de replay, las suites genéricas de agente real y un driver de Loader sin clave no exportado.
- [`@deepseek-ai/dsh-acp-demo`](../../../../packages/examples/acp-demo/README.md) y `@deepseek-ai/dsh-sdk-jsonrpc-server` son propietarios de sus integraciones de protocolo con marco.

El modelo de proyecto SDK que llevaba la opción de interfaz de ejecución `stdio` se elimina con la [eliminación del toolchain de proyectos SDK](2026-08-11-remove-sdk-project-toolchain.es.md). La documentación de demo orientada al repositorio requiere una clave de API de DeepSeek y arranca con un producto ejecutable actual.

La validación sin clave es propiedad de los tests. El smoke del Loader Headless usa un adaptador fixture para ejercitar un round trip real de herramienta, la suite del binario compilado `dsh` fija la entrada y salida one-shot publicadas, la instantánea Headless de producto fija la persistencia, y el e2e de apagado PTY Headless fija la escalada de señales. Los tests específicos de paquete del Loader mantienen adaptadores deterministas junto a sus escenarios. Ninguno se expone como agente mock ejecutable.

## Verificación

La cobertura de Loader TUI y Headless ejecuta los paquetes de app reales en modos fuente y compilado. La cobertura de subprocesos con PTY está reservada al ciclo de vida de TUI; los demás smokes de punto de entrada usan el protocolo de pipe one-shot. Headless prueba sus contratos de tarea/resultado y de llamada de herramienta. Los grafos generados y las búsquedas del repositorio rechazan referencias obsoletas a paquetes, comandos, hojas, interfaces de SDK, `createStdioChat` y `StdioRuntime`.

El binario compilado `dsh` rechaza un lanzamiento de TUI por pipe antes del arranque del Loader y apunta a `dsh --profile headless`; `apps/cli/tests/built-bin.e2e.ts` fija la entrada one-shot de producto bajo Node plano, incluidos la salida y los argumentos inválidos. `examples/headless-agent/tests/headless.snapshot.ts` fija la persistencia de producto, mientras que `apps/cli/tests/headless-shutdown.e2e.ts` es propietario de la escalada de señales acotada. El driver JSONL solo-de-tests del ejemplo headless preserva las instantáneas de eventos canónicos ensambladas sin crear un segundo contrato CLI. Code Mode tiene instantáneas de TUI programáticas y un demo de overlay ACP. La integración de contexto de tiempo usa la composición de test Headless explícita para dos turnos ordenados, mientras que sus tests de paquete son propietarios del comportamiento más fino de tiempo transcurrido.

## Alternativas consideradas

- **Conservar el agente de líneas solo para pipes** — rechazado porque Headless tiene un contrato de tarea acotado, stdout puro de formato, completado durable y estado de salida de proceso.
- **Conservar, plegar o promocionar el helper readline como paquete** — rechazado porque tenía un único consumidor de app y ningún contrato intercambiable independiente. Plegarlo en la app stdio eliminaba un límite de paquete de soporte injustificado pero aún conservaba el producto redundante; una futura UI de líneas independiente necesita un segundo consumidor real antes de reintroducir ese paquete.
- **Conservar Echo como inicio rápido sin clave** — rechazado porque la primera experiencia de producto debería ejercitar el modelo real y el coding agent soportado, no un adaptador con guion con una herramienta a medida.
- **Conservar Echo solo como comando de demo de CI** — rechazado porque los fixtures Headless propiedad de tests cubren los mismos límites de Loader y artefacto compilado sin preservar una hoja de producto mock.
- **Eliminar todo mecanismo stdio o mock** — rechazado porque los protocolos con marco, la E/S de proceso y los adaptadores de test deterministas son infraestructura independiente, no los agentes eliminados.

## Consecuencias

- La ejecución de producto interactiva y no interactiva tienen cada una un propietario y una hoja de coding ejecutable.
- El repositorio no tiene ningún demo de agente orientado al usuario sin clave; los demos locales de agente requieren `DEEPSEEK_API_KEY`.
- CI conserva la cobertura de entrada real sin clave mediante fixtures de test en lugar de un comando de producto.
- Las configuraciones de agente stdio existentes y los comandos Echo fallan en lugar de traducirse.
- La interacción multi-turno por pipe en un proceso y el provider readline para `ask_user_question` no-TTY desaparecen intencionadamente; la reanudación cubre el trabajo multi-turno durable, y una composición no-TTY debe suministrar su propio provider de interacción.
