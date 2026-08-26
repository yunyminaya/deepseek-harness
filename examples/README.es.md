# Ejemplos

[English](README.md) | Español

Demostraciones ejecutables de las principales interfaces y puntos de extensión de DeepSeek Harness. Cada directorio hijo es dueño de su configuración, sus prerrequisitos, sus comandos y su comportamiento detallado.

## mcp-memory

Overlays opcionales que conectan servidores de memoria de terceros soportados a través del cliente MCP genérico. Ver la [referencia del ejemplo de memoria](mcp-memory/README.es.md).

## headless-agent

Un agent no interactivo que acepta una tarea, la ejecuta y emite un formato de salida seleccionable, legible por máquina o por humanos. Ver la [referencia del ejemplo headless](headless-agent/README.es.md).

## jsonrpc-agent

Un coding agent sin supervisión, manejado a través del SDK de Python y JSON-RPC. Ver la [referencia del ejemplo JSON-RPC](jsonrpc-agent/README.es.md).

## web-cordis

Un agent autorreferencial que puede inspeccionar y modificar su árbol de plugins Cordis en memoria. Ver la [referencia del ejemplo web-cordis](web-cordis/README.es.md).

## web-schedule

Un overlay Web opcional (opt-in) para recordatorios duraderos locales a la Session. Soporta retardos `after_seconds` positivos de segundos enteros y destinos `at` absolutos mediante `schedule_create`, `schedule_list` y `schedule_delete`; los recordatorios activos persisten en la Session original, se reanudan cuando esa Session vuelve a estar activa y no se ejecutan mientras está fría. Ejecuta `dsh web --patch examples/web-schedule/cordis.yml`; ver [web-schedule/README.md](web-schedule/README.es.md) para la autoridad de tiempo absoluto y los límites de entrega y recuperación.

## acp-agent

Un servidor de automatización del Agent Client Protocol para clientes programáticos, con soporte de sesión, permiso y cancelación. Ver la [referencia del ejemplo ACP](acp-agent/README.es.md).
