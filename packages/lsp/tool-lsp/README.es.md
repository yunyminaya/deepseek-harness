# @deepseek-ai/dsh-tool-lsp

[English](README.md) | Español

La **herramienta `lsp`** orientada al modelo sobre `ctx.lsp`: una herramienta de solo lectura con cuatro operaciones para una navegación precisa del código. Posee el schema del modelo, la guía del prompt, la conversión de coordenadas, los límites y el formateo de resultados, y la presentación en la UI; no importa ningún provider.

Plugin de namespace (`name` / `inject` / `Config` / `apply`, sin export por defecto). Inyecta `tools`, `lsp` y `systemPrompt`.

## La herramienta

`lsp` acepta `operation` (`goToDefinition` | `findReferences` | `goToImplementation` | `hover`), `file_path`, `line` y `character`. `line` y `character` son coordenadas de cursor positivas de base 1 en UTF-16; la herramienta las convierte a las posiciones de base 0 del seam y vuelve a convertir las ubicaciones renderizadas. `findReferences` incluye las declaraciones para que el análisis de impacto no omita el sitio de definición. El provider, el id de lenguaje, la raíz del workspace, los límites, el timeout, la inicialización y el ejecutable quedan fuera de la entrada del modelo.

La herramienta exige la raíz del workspace desde el `header.cwd` de la sesión, sin alternativa: si falta, falla como `LSP_WORKSPACE_REQUIRED` antes de consultar. Su resultado canónico es la unión completa normalizada del Service Definition: `{ kind: "locations", locations, resolvedWorkspaceUri }` o `{ kind: "hover", hover }`; Code Mode puede inspeccionar directamente cada ubicación adquirida y rango de base 0. El renderizado nativo proyecta las entradas estables `path:line:character` agrupadas por archivo contra la URI canónica del workspace del provider en lugar de aplicar reglas de ruta de la plataforma host al cwd de la sesión. Una URI `file:` se convierte en una ruta relativa al workspace dentro de esa URI o en una ruta absoluta derivada de la URI fuera de ella; las URIs malformadas y las que no son `file:` permanecen verbatim. Las ubicaciones vacías y el hover `null` son respuestas exitosas sin resultados; los payloads malformados del provider siguen siendo errores estructurados.

## Configuración

| Clave | Predeterminado | Significado |
|---|---|---|
| `maxLocations` | `100` | Mayor número de ubicaciones renderizadas antes de un marcador de omisión. |
| `maxResultChars` | `16000` | Mayor resultado completo renderizado, incluidos los metadatos de truncamiento. |
| `timeoutMs` | `60000` | Presupuesto de timeout de la llamada de herramienta, aplicado por `dsh-tool-call-timeout-policy`; cubre el ciclo de vida completo encolado de apertura/consulta/cierre y no es configurable por el modelo. |

## Experiencia del modelo

### Prompt del sistema

#### Qué ve el modelo

Una sección del system prompt (orden 112) presenta LSP como una ayuda de precisión con el siguiente texto:

##### Guía verbatim

```markdown
Use search/read for ordinary navigation. Use lsp when textual matches are ambiguous or before a change requires precise definitions, implementations, or references. Positions are one-based line and character (UTF-16) at the cursor; an off-symbol position may return no results. findReferences always includes the declaration.
```

#### Efecto de tokens

Coste fijo de guía en cada solicitud mientras el plugin esté activo.

#### Efecto de KV Cache

Estable en el prefijo mientras el ámbito del plugin y el texto de guía no cambien; la activación o la disposición pueden invalidar la reutilización desde esta sección.

### Schema de la herramienta

#### Qué ve el modelo

El modelo ve el [schema `lsp` generado](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-lsp).

#### Efecto de tokens

Coste fijo de schema en cada solicitud mientras esté habilitado; el presupuesto `timeoutMs` nunca se envía al modelo.

#### Efecto de KV Cache

Estable en el prefijo mientras la definición visible de la herramienta y su orden no cambien; el ciclo de vida del registro o las restricciones con ámbito pueden invalidar la reutilización desde el primer token de schema que cambie.

### Resultados

#### Qué ve el modelo

Líneas de ubicación `path:line:character` agrupadas por archivo o texto de hover normalizado, limitadas primero por `maxLocations` y después por `maxResultChars`; los marcadores de omisión y truncamiento se incluyen dentro del tope de caracteres completo. Estos topes afectan solo a la presentación Native/modelo, no al valor canónico. Los resultados vacíos usan líneas distintas `No results.` / `No hover information.`.

#### Efecto de tokens

Limitado por resultado de herramienta mediante `maxResultChars`, con `maxLocations` acotando además el número de elementos de navegación.

#### Efecto de KV Cache

Los resultados de herramienta se añaden después del prefijo de solicitud cacheado y no lo invalidan directamente.

### Presentación en la UI

#### Qué ve el modelo

Nada. El cliente renderiza una tarjeta de búsqueda genérica — `{ card: 'generic', kind: 'search', title, locations: [{ path, line }] }` — cuyo título derivado de los args lleva la operación y el cursor de base 1; el seguimiento enfoca la línea consultada mientras el título conserva la columna.

#### Efecto de tokens

Efecto de tokens directo nulo porque el renderizado es solo del lado del cliente.

#### Efecto de KV Cache

Ninguno; la presentación en la UI queda fuera de la solicitud del modelo.

## Limitaciones conocidas y trabajo diferido

- **Coordenadas de cursor UTF-16** — las columnas son exactas para el protocolo, pero difíciles de contar para un modelo alrededor de caracteres no BMP; una posición fuera de símbolo puede devolver resultados vacíos, así que el prompt explica la convención sin fomentar un uso amplio de LSP ([Agent Note del seam](../../../.agents/notes/implemented/architecture/2026-07-15-lsp-capability-seam.es.md)).
- **Sin promesa de completitud entre servidores** — los servidores admitidos pueden devolver resultados vacíos o parciales según la preparación de la indexación; la herramienta no promete completitud entre lenguajes ni servidores.
