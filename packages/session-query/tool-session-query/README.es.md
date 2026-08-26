# @deepseek-ai/dsh-tool-session-query

[English](README.md) | Español

Herramientas de modelo autorizadas por el workspace sobre `ctx.sessionQuery`. Este paquete opcional depende solo de la interfaz unificada y registra `session_search`, `session_event_search`, `session_trace`, `session_event_trace` y `session_event_read`; las composiciones de host distribuidas no lo montan por defecto.

## Configuración

| Clave | Valor por defecto | Significado |
|---|---:|---|
| `maxSearchResults` | `100` | Máximo de coincidencias autorizadas ajenas a uno mismo recogidas en las páginas internas del provider |
| `searchTimeoutMs` | `30000` | Plazo cooperativo adjunto a ambas herramientas de búsqueda de texto completo |

El llamante proviene exclusivamente de `ToolExecution.exec.agent`. El acceso entre sesiones exige igualdad exacta entre los valores de `cwd` de la sesión objetivo y de la sesión llamante; un llamante sin `cwd` solo puede inspeccionarse a sí mismo. La búsqueda nunca expone cursores del provider, offsets, tamaños de página ni un límite controlado por el modelo. Como una búsqueda consume internamente cursores del provider ligados a la generación, ambas herramientas de búsqueda se ejecutan exclusivamente con llamadas de herramientas hermanas; las tres herramientas exactas de rastreo/lectura optan por la ejecución en paralelo. Cada ejecutor exacto pasa su señal de ejecución sin cambios a través de la autorización y del rastreo/lectura del servicio, de modo que la cancelación espera la limpieza cooperativa de la persistencia y conserva la razón exacta de la señal. Las marcas de tiempo en la frontera de herramienta exigen una `Z` explícita o un offset numérico y se convierten en filtros inclusivos de milisegundos de época.

`session_search` omite siempre la sesión llamante. Los ids de padre solicitados se deduplican y se comprueban contra la autoridad del workspace del llamante antes del FTS; solo los ids autorizados llegan al provider, mientras que las suposiciones ausentes y las de otros workspaces se comportan de forma idéntica y el marcador raíz permanece ORed de forma independiente. Un `session_event_search` de la sesión actual se detiene inmediatamente antes del step que lo invocó, de modo que la salida activa del asistente y la llamada de herramienta registrada no pueden coincidir consigo mismas. Los objetivos directos se autorizan antes de las lecturas de rastreo, evento o título. La salida de linaje reemplaza las fronteras no autorizadas de ancestros y descendientes con marcadores que no contienen ningún id de sesión oculto.

Cada llamada de confianza a `ctx.sessionQuery` cruza un sanitizador de frontera de modelo. La cancelación del llamante se comprueba primero y se conserva exactamente. Los diagnósticos disponibles del corpus y del provider, incluidas las causas anidadas inspeccionables con seguridad, se registran internamente con el mejor esfuerzo; los fallos no imprimibles usan un marcador de posición de log fijo. El formateo de diagnósticos y la clasificación de errores están protegidos de forma independiente, de modo que una causa no imprimible no puede escapar ni impedir un error exterior clasificado con seguridad, mientras que una clasificación o un registro inseguros recurren al código y al mensaje fijos `SESSION_QUERY_TOOL_FAILED`. Los errores locales de validación de argumentos y de autorización conservan sus mensajes precisos propiedad de la herramienta.

El paquete no realiza deliberadamente ningún truncado de bytes ni de caracteres y no importa un backend de spill. Los despliegues que necesitan una salida en línea acotada montan `@deepseek-ai/dsh-spill-policy`, que puede reemplazar el texto renderizado tras la ejecución conservando el resultado completo.

## Experiencia de modelo

### Prompt del sistema

#### Lo que ve el modelo

El modelo recibe una sección fija de guía de historial previo.

##### Guía de historial previo

```markdown
Use session_search to find relevant work from prior sessions, or session_event_search to search earlier events in one session. Search results are cursor-free and workspace-scoped. Follow a useful hit with session_trace, session_event_trace, or session_event_read when you need lineage, relationships, or exact data.
```

#### Efecto en tokens

Hay una sección concisa fija en cada solicitud mientras el plugin está montado.

#### Efecto en la caché KV

Estable en el prefijo mientras el plugin y el texto de guía no cambien.

### Schemas de herramientas

#### Lo que ve el modelo

El modelo ve los schemas generados [`session_search`, `session_event_search`, `session_trace`, `session_event_trace` y `session_event_read`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-session-query). Los filtros de búsqueda añaden tokens de schema fijos, mientras que los cursores, las rutas de workspace, la paginación de salida y los límites de resultados controlados por el modelo siguen ausentes.

#### Efecto en tokens

Se envían cinco schemas fijos de solo lectura en cada solicitud mientras son visibles.

#### Efecto en la caché KV

Estable en el prefijo mientras la visibilidad de las herramientas y las definiciones no cambien.

### Resultados de herramientas

#### Lo que ve el modelo

Cada llamada con éxito emite un bloque de texto plano. Los resultados de búsqueda incluyen títulos y fragmentos de mejor coincidencia; los rastreos incluyen todas las relaciones autorizadas; las lecturas de eventos incluyen el JSON objetivo íntegro. La spill policy genérica puede reemplazar el texto en línea sobredimensionado con su vista previa, su localizador opaco y su pista de recuperación.

#### Efecto en tokens

Los resultados dependen de los datos y permanecen en el historial de herramientas registrado hasta la compactación; `maxSearchResults` acota el número de coincidencias de búsqueda.

#### Efecto en la caché KV

El texto de resultados de solo anexión sigue al prefijo de solicitud reutilizable y no invalida las entradas de caché anteriores.

## Limitaciones conocidas y trabajo diferido

- La búsqueda devuelve como máximo el tope del despliegue y pide al modelo que acote su consulta cuando existen más coincidencias; no ofrece ningún token de continuación.
- La identidad del workspace es una igualdad conservadora de cadenas exactas de `cwd`, de modo que las rutas equivalentes por symlink no comparten autoridad.
- Las composiciones personalizadas sin la spill policy genérica aceptan payloads completos de rastreo y de evento en línea.
