# `@deepseek-ai/dsh-client-ui-reference`

[English](README.md) | [中文](README.zh.md) | Español

Fuente Web unificada de `@file` y `@session`. El navegador inicia juntas las llamadas Remote `fileReferences/list` y `sessionReferenceResolver/candidates` para un token sin comillas, ordena deterministamente los archivos antes que las sesiones con etiquetas de carpeta/archivo/sesión registradas en el locale, y renderiza las filas bajo encabezados de sección de archivo y sesión no seleccionables sin un título redundante de fuente `reference` cruda. Cualquier dominio de candidatos fallido degrada de forma independiente. Un token abierto `@"…` busca solo archivos.

Las selecciones de archivo conservan el texto natural definido por la gramática compartida `@path` como su forma serializada y de portapapeles oculta. Un archivo cierra el completado como una referencia en línea atómica mostrada con un glifo de archivo, nombre de archivo en color de negocio y sin cápsula. Un directorio permanece como texto de ruta editable plano con un glifo de carpeta y mantiene el menú activo en su barra final para que el usuario pueda descender otro nivel. Las rutas con espacios en blanco usan `@"path with spaces"`, y una comilla que el usuario abrió explícitamente permanece entre comillas.

Las selecciones de sesión insertan una referencia en línea atómica cuyo `ref` oculto y cuya representación de portapapeles son la mención canónica `@[label](dsh-session:…)` devuelta por el Host. Su forma visible es un glifo de burbuja de chat más el título de sesión en color de negocio, sin cápsula; la serialización nunca reconstruye la identidad desde ese título. El envío ordinario lleva la mención canónica a través de `session.prompt`; el servicio de referencia de sesión la valida y captura el contexto del modelo en `agent/pre-step`.

El export de `/client` es solo el cuerpo del plugin (`apply`/`inject`); la codificación de candidatos permanece interna al efecto de registración.

## Experiencia de modelo

Indirectamente, a través de `@deepseek-ai/dsh-file-reference-local` para la guía de rutas y `@deepseek-ai/dsh-session-reference` para las instantáneas de sesión preparadas.

#### Efecto en KV Cache

La navegación de candidatos no tiene efecto de modelo. Un archivo o sesión seleccionados cambian solo el sufijo del nuevo mensaje de usuario y cualquier contexto de referencia de sesión preparado por el Host que sigue a ese mensaje; el historial de objetivos anterior permanece sin cambios.

## Limitaciones conocidas y trabajo diferido

- **El fallo de candidatos es deliberadamente silencioso** — una llamada de descubrimiento Remote no disponible o fallida no produce filas para ese dominio. Un fallo de preparación de referencia de sesión ocurre tras la aceptación del prompt y termina ese turno del agent.
- **Sin escaneo de archivos en el navegador** — el completado Web requiere un provider `ctx.fileReferences` del Host montado; el navegador no puede caer a su propio sistema de archivos.
- **La búsqueda de sesiones sigue siendo solo de metadatos** — el descubrimiento filtra el id de sesión, el cwd y el título más reciente respaldado por log a través de `ctx.sessionReferenceResolver`; los cuerpos de mensaje y los transcripts completos no se buscan.
