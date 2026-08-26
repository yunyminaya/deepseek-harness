# Ejemplos de memoria MCP de terceros

[English](README.md) | Español

Estas tres **configuraciones de referencia desactivadas por defecto** conectan un sistema de memoria a DSH a través de [`@deepseek-ai/dsh-mcp-client`](../../packages/mcp/mcp-client/README.md). Elige una, o copia la misma fila MCP genérica para otro servidor.

Estas configuraciones de terceros se ofrecen solo como ejemplos de interoperabilidad. Su inclusión no implica respaldo, recomendación, colaboración ni soporte continuado por parte de DeepSeek.

## Qué hace DSH

DSH analiza el overlay Cordis seleccionado, arranca un comando stdio configurado o se conecta a una URL Streamable HTTP configurada, descubre las herramientas MCP y las expone como `mcp__<serverName>__<tool>`. DSH **no** descarga el servidor, ni inicializa su base de datos, ni elige su modelo o su provider de embeddings, ni crea una cuenta en la nube, ni migra datos del proveedor, ni supervisa un servicio HTTP aparte. Para stdio, el cliente genérico lanza y detiene el hijo con el ciclo de vida de los plugins de DSH; para HTTP, el servicio upstream debe estar ya en ejecución.

El bridge stdio elimina deliberadamente las variables de entorno cuyos nombres suelen identificar credenciales y todas las variables `DSH_*` antes de lanzar un hijo; las demás variables de entorno se siguen heredando. Cada ejemplo añade solo el override de referencia que necesita. Si una funcionalidad opcional del upstream necesita otro secreto, añade esa variable al `config.env` de la fila en lugar de poner el secreto directamente en el YAML.

## Elige una

| Sistema | Pin probado | Transporte | Prerrequisito del upstream |
|---|---:|---|---|
| [Memorix](https://github.com/AVIDS2/memorix) | `memorix@1.3.0` (`500792cad3144142293bfbb20acb4841c9f7fcfa`) | stdio | Node 22.18+ y `npm install --global memorix@1.3.0` |
| [MCP Reference Memory](https://github.com/modelcontextprotocol/servers/tree/main/src/memory) | `@modelcontextprotocol/server-memory@2026.7.4` (`6dd0a683e198783e30feabf7abaf42f925bd18b1`) | stdio | `npm install --global @modelcontextprotocol/server-memory@2026.7.4` |
| [Engram](https://github.com/Gentleman-Programming/engram) | `v1.20.0` (`ba9e46ced152c37a7cb9e576153c41995873e2fc`) | stdio | Go 1.25.10+ y `go install github.com/Gentleman-Programming/engram/cmd/engram@v1.20.0`, o el binario de lanzamiento correspondiente |

## Activa una

Pasa un overlay a DSH:

```sh
dsh web --patch "$PWD/examples/mcp-memory/memorix.cordis.yml"
```

Sustituye el nombre de archivo por `mcp-reference-memory.cordis.yml` o `engram.cordis.yml`. La ruta puede apuntar a un archivo copiado en cualquier lugar del disco. No hay ningún servidor de memoria en la composición distribuida, así que omitir `--patch` mantiene los tres desactivados.

Para conservar la selección entre ejecuciones, fusiona el único patch `insert` del archivo elegido en una capa de patch de usuario — `$DSH_HOME/profiles/<name>/cordis.patch.yml` para un solo perfil, o `$DSH_HOME/cordis.patch.yml` para todos los perfiles de la máquina. No copies sobre un archivo existente: puede que ya contenga patches de usuario no relacionados.

## Configuración del provider

### Memorix

```sh
npm install --global memorix@1.3.0
dsh web --patch "$PWD/examples/mcp-memory/memorix.cordis.yml"
```

Memorix funciona en modo heurístico local sin un LLM ni un servicio de embeddings. Configura los providers opcionales en el propio `~/.memorix/config.toml` de Memorix o en el `memorix.toml` del proyecto. El ejemplo conserva la identidad de proyecto Git de Memorix a partir del directorio de trabajo de DSH y usa el valor por defecto `~/.memorix/data` propio de Memorix. Establece `MEMORIX_DATA_DIR` antes de arrancar DSH para anularlo.

### MCP Reference Memory

```sh
npm install --global @modelcontextprotocol/server-memory@2026.7.4
dsh web --patch "$PWD/examples/mcp-memory/mcp-reference-memory.cordis.yml"
```

Este servidor de referencia almacena un grafo de conocimiento local y expone las herramientas entity, relation, observation, read, search y open. No necesita ningún servicio de modelo ni de embeddings. El ejemplo almacena su JSONL en `$HOME/.dsh-mcp-reference-memory.jsonl` en lugar del directorio del paquete npm instalado. Establece `MEMORY_FILE_PATH` antes de arrancar DSH para anularlo.

La búsqueda es coincidencia de subcadenas sin distinción de mayúsculas sobre nombres, tipos y observaciones de entidades, no recuperación semántica. El servidor no añade embeddings, ni resumen automático, ni resolución de conflictos, ni una política de olvido.

### Engram

```sh
go install github.com/Gentleman-Programming/engram/cmd/engram@v1.20.0
dsh web --patch "$PWD/examples/mcp-memory/engram.cordis.yml"
```

Engram es dueño del almacenamiento y de la selección de proyecto: usa `~/.engram` por defecto, detecta el proyecto Git desde el directorio de trabajo de DSH y acepta `ENGRAM_DATA_DIR` o `ENGRAM_PROJECT` como overrides de entorno.

## Instrucción de modelo compartida opcional

Añade esta instrucción breve y neutra respecto al proveedor a tus instrucciones de modelo existentes si las descripciones de herramientas del servidor no disparan el uso de memoria de forma fiable:

> Cuando el usuario te pida recordar algo, llama a una herramienta de escritura de memoria. Cuando la información histórica pueda ser relevante, busca en la memoria y usa los resultados relevantes.

Esto es solo una guía aditiva. Los ejemplos no sustituyen la persona del system prompt de DSH.

## Verifica la escritura, el recuerdo en sesión nueva y el uso

Usa un valor único y mantén el ámbito de almacenamiento del provider sin cambios durante todo el proceso:

1. En la sesión A de DSH, pregunta: `Remember that my validation drink is lapsang-<unique suffix>.` Confirma que el modelo llamó a la herramienta de escritura del provider y que la herramienta devolvió éxito.
2. Crea la sesión B de DSH en el mismo Host en ejecución. No copies la conversación de la sesión A. Pregunta: `What is my validation drink? Check memory.` Confirma que el modelo llamó a la herramienta de búsqueda o recuerdo del provider y devolvió el valor.
3. Todavía en la sesión B, pregunta: `Use that preference to suggest one drink for the meeting.` Confirma que la respuesta usa el valor recordado.

Se necesita una sesión nueva de DSH; no un reinicio del Host. El reinicio o el HMR solo hacen falta después de que un hijo MCP se estrelle, porque el cliente genérico actual no se reconecta automáticamente; sus registros de herramientas permanecen hasta la liberación del plugin o una re-sincronización con éxito, y las llamadas pueden fallar contra el transporte cerrado. El descubrimiento inicial es asíncrono, así que espera a las herramientas `mcp__...` del provider antes de enviar el primer prompt de validación.

## Trae otro servidor MCP

Copia los mismos campos de la entrada y usa un `id` y un `serverName` únicos:

```yaml
- insert:
    - id: memory-my-server
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: my-memory
        transport: stdio
        command: my-memory-mcp
        args: []
        env: {}
        cwd: !!js process.cwd()
```

Para un servidor remoto, usa `transport: streamable-http`, `url` y `headers` en su lugar. La instalación, la identidad, la autenticación, los modelos, los embeddings, la persistencia y el licenciamiento específicos del provider siguen siendo responsabilidad del provider.
