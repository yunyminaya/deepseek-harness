# `@deepseek-ai/dsh-file-reference-local`

[English](README.md) | Español

Implementación en el sistema de archivos local de `ctx.fileReferences`. Mantiene una `WorkspaceFileSearch` acotada por agent, enraizada en el `cwd` de esa sesión y con respaldo en el cwd del proceso host. El índice clasifica los listados directos de directorio para consultas que contienen `/`; en caso contrario clasifica difusamente un índice recursivo acotado; nunca sigue symlinks de directorio.

Los eventos de resultado de herramienta invalidan el índice reutilizable del agent direccionado para que la finalización posterior observe mutaciones probables del workspace. La disposición del agent libera ese índice y su contribución de prompt con ámbito; la disposición del plugin espera a cada fiber de prompt y libera todas las búsquedas en caché.

## Configuración

| Clave | Por defecto | Contrato |
|---|---:|---|
| `maxResults` | `20` | Máximo de candidatos clasificados devueltos para una consulta. |
| `maxEntries` | `10000` | Máximo de archivos y directorios indexados por workspace de agent. |
| `excludedDirectories` | `[".git", "node_modules"]` | Nombres base de directorio omitidos del recorrido y de los candidatos. |

Todo valor numérico debe ser un entero seguro positivo. Los nombres excluidos deben ser nombres base no vacíos sin `/` ni `\\`.

## Model Experience

### Guía de referencia a archivos cuando `read` está disponible

#### Lo que ve el modelo

Cuando el agent direccionado tiene una herramienta `read` efectiva, el provider contribuye esta sección estable del system prompt:

##### Instrucción de referencia a archivos

```markdown
Paths prefixed with @ are files explicitly referenced by the user. Use the read tool when their contents are needed; do not claim to have inspected a file before reading it.
```

#### Efecto de tokens

Condicional y fijo: la única frase está presente mientras `read` sea visible para el agent direccionado; la búsqueda de candidatos en sí no añade tokens, y una ruta seleccionada contribuye solo sus caracteres ordinarios de mensaje de usuario.

#### Efecto de caché KV

La frase estable se une al prefijo del system prompt. Montar o quitar este provider, o cambiar si `read` es visible, cambia ese prefijo; las consultas, los candidatos y las invalidaciones de índice no.

## Limitaciones conocidas y trabajo diferido

- **Namespace local al host** — el provider escanea el sistema de archivos del host del Harness, por lo que las implementaciones remotas o virtuales de `read` requieren un provider cuyo namespace coincida con la herramienta.
- **Índice de asesoramiento acotado** — los workspaces muy grandes pueden omitir rutas después de `maxEntries`, y los directorios excluidos o ilegibles no aparecen.
- **Sin semántica de archivos de ignorar** — `.gitignore` y otros archivos de ignorar del proyecto no influyen en el descubrimiento; solo se excluyen los nombres base de directorio configurados.
