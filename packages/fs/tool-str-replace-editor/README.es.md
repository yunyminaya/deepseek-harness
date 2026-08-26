# @deepseek-ai/dsh-tool-str-replace-editor

[English](README.md) | Español

Un `str_replace_editor` independiente orientado al modelo sobre `ctx.fs`. Se puede componer con Bash persistente, Bash de un solo uso, Bash con sandbox u otra superficie de terminal.

## Configuración

| Clave | Valor por defecto | Significado |
|---|---:|---|
| `maxOutputChars` | `16000` | Caracteres de prefijo que se conservan en las vistas de archivo y de directorio. |
| `description` | Editor command guide | Descripción de la herramienta orientada al modelo. |

## Herramienta

El schema ofrece `view`, `create`, `str_replace` e `insert` sobre rutas absolutas. Las vistas de archivo usan números de línea basados en 1 y conservan las tabulaciones del contenido, de modo que el texto mostrado sigue siendo entrada válida de reemplazo literal; las vistas de directorio omiten las entradas ocultas, de dependencias y de caché de Python y descienden dos niveles. Un fallo de metadatos de `view`, `str_replace` o `insert` registra la ausencia confirmada antes de devolver `FS_NOT_FOUND`, de modo que un `create` posterior puede recuperar una ruta eliminada externamente mediante el flujo de creación con guarda de la política montada; la ausencia nunca autoriza `str_replace` ni `insert`. El reemplazo exige una única coincidencia literal y solo informa errores en el vocabulario público de `old_str`. Insert sigue el límite de inserción seleccionado basado en 0 sin añadir un salto de línea final implícito. Las mutaciones conservan las tabulaciones fuera de la edición solicitada.

## Experiencia de modelo

### Schema de la herramienta

#### Lo que ve el modelo

El [schema generado de `str_replace_editor`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-str-replace-editor), incluida la `description` configurada. El plugin no aporta ninguna sección independiente de prompt del sistema.

#### Efecto en tokens

Coste fijo de schema mientras `str_replace_editor` está visible.

#### Efecto en la KV cache

Estable en el prefijo mientras la `description` y el schema configurados permanecen sin cambios.

### Resultados de la herramienta

#### Lo que ve el modelo

Las vistas devuelven texto numerado o un listado de directorio superficial. Las llamadas exponen las ubicaciones de archivo, y las llamadas create/replace exponen tarjetas de diff a las superficies de presentación. Las mutaciones devuelven confirmaciones concisas. Las vistas largas conservan su prefijo y añaden un aviso de recorte.

#### Efecto en tokens

Depende de los datos y está acotado por `maxOutputChars` más el aviso fijo de recorte.

#### Efecto en la KV cache

Los resultados de herramienta de solo añadidura siguen el prefijo de petición reutilizable.

## Limitaciones conocidas y trabajo pendiente

- Las operaciones se dirigen a texto UTF-8; los archivos binarios no se admiten.
- `str_replace` rechaza a propósito las coincidencias cero o múltiples y no tiene argumento `replace_all`.
- Cada mutación pasa por `fs/write-intent` o `fs/edit-intent`, resuelve la política de sandbox de la sesión actual y delega el cumplimiento en los plugins de sistema de archivos y de política montados.
