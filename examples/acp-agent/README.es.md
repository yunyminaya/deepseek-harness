# Ejemplo acp-agent

[English](README.md) | Español

Servidor orientado a la automatización del [Agent Client Protocol](https://agentclientprotocol.com) sobre JSON-RPC por stdio. Está pensado para parent agents, providers de subagentes y otros clientes programáticos, no como UI de producto.

```sh
pnpm run demo:acp             # needs DEEPSEEK_API_KEY (repo-root .env or env)
pnpm run demo:code-mode       # same protocol with the Code Mode tool transport
```

La hoja carga la app de ACP, el adaptador DeepSeek, las pilas de bash y sistema de archivos con sandbox, una política de aprobación de un solo uso (one-shot), compactación, subagentes, flujos de trabajo, hooks, un índice derivado de consultas de sesión y el guard de repetición. La app crea un agent nuevo por cada `session/new`, persiste las sesiones en JSONL y mantiene stdout puro del protocolo. Los overlays opcionales añaden consultas de sesión, almacenamiento de spill en sistema de archivos, Code Mode o fetch web.

## Canal del protocolo

Stdout transporta solo ACP JSON-RPC delimitado por saltos de línea. `@deepseek-ai/dsh-acp-demo` no instala ningún logger de stdout; las adiciones de la hoja deben usar stderr para los diagnósticos.

El contrato de automatización — métodos soportados, contenido del prompt de referencia (baseline), salida de texto confirmado y las superficies de UI deliberadamente ausentes — vive en [`@deepseek-ai/dsh-acp`](../../packages/acp/acp/README.es.md).

## Workspaces de sesión y permisos

Cada `session/new` aporta un `cwd` absoluto. Las mutaciones de bash y sistema de archivos con sandbox resuelven `workspace-write` contra el cwd de esa sesión, de modo que las sesiones concurrentes pueden usar raíces de proyecto separadas; las raíces temporales de la plataforma siguen siendo espacio de scratch compartido y escribible ([contrato del sandbox](../../packages/sandbox/sandbox/README.md)). `DSH_PERMISSION_MODE` selecciona `workspace-write` o `danger-full-access` para el despliegue.

Bajo `workspace-write`, un reintento del modelo que solicite un acceso más amplio al sandbox dispara `session/request_permission` con `allow_once` y `reject_once`. El cliente decide programáticamente; el descarte o una respuesta no disponible fallan de forma cerrada (fail closed). El resultado seleccionado se aplica solo a ese reintento y se registra por la vía normal de tool-result/audit. El servidor nunca expone un selector de permisos ni persiste la política del cliente.
