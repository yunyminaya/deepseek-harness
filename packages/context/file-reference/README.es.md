# `@deepseek-ai/dsh-file-reference`

[English](README.md) | Español

Seam de descubrimiento de referencias a archivos y gramática `@file` segura para navegador compartida por las interfaces de usuario respaldadas por host. `ctx.fileReferences.list(agent, query, signal)` devuelve candidatos de archivo o directorio solo con ruta para el agent direccionado; los providers concretos son dueños del acceso al namespace, la clasificación, la caché y la invalidación. El mismo contrato es invocable remotamente como el método Remote unario `fileReferences/list` (`@Remote` en la Service Definition, cancelado a través de la señal final reservada), por lo que los consumidores de navegador llaman a `ctx.remote.fileReferences.list` sin una ruta de API Proxy.

`activeAtToken()` reconoce un token `@path` o un `@"path with spaces` abierto solo al inicio de la entrada o tras un espacio, para que el texto similar al correo no abra la finalización. `formatFileMention()` emite la ortografía de prompt coincidente, añade `/` a los candidatos de directorio, preserva una comilla abierta explícitamente y rechaza caracteres de control o comillas incrustadas que la gramática del editor no pueda representar de forma segura.

Seleccionar un candidato no lee ni adjunta contenidos de archivo. El `FILE_REFERENCE_PROMPT` exportado es guía estable que un provider puede instalar cuando el agent direccionado puede llamar a `read`.

## Model Experience

Indirectamente, a través de `@deepseek-ai/dsh-file-reference-local`, que contribuye condicionalmente la guía estable de referencia a archivos de este paquete.

#### Efecto de caché KV

La interfaz y la gramática no añaden tokens de petición por sí mismas; una sección de prompt propiedad del provider determina el comportamiento de la caché.

## Limitaciones conocidas y trabajo diferido

- **Los candidatos de ruta son de asesoramiento** — el seam no demuestra que una herramienta posterior del sistema de archivos orientada al modelo pueda acceder al mismo namespace; los despliegues deben alinear el provider con la implementación efectiva de `read`.
- **Sin objeto de referencia de contenido de archivo** — los archivos seleccionados siguen siendo texto de prompt ordinario y requieren una llamada de herramienta explícita del modelo antes de que sus contenidos sean visibles para el modelo.
