# Usa la Web UI

[English](index.md) | Español

Inicia la Web UI siguiendo el [README raíz](../../../README.es.md#run); el comando imprime su URL. Esta guía comienza cuando ese servidor ya está en ejecución. El proceso `dsh` usa el directorio desde el que se invoca como ubicación predeterminada del sistema de archivos, pero una Web UI recién iniciada no tiene ningún workspace seleccionado hasta que tú añadas uno.

## Configura un modelo

Abre **Settings → Models**, introduce una [clave de API de DeepSeek](https://platform.deepseek.com/) y guárdala. La ruta del modelo queda utilizable de inmediato sin reiniciar el servidor.

La [guía de configuración de modelos](./providers.es.md) cubre otros providers y endpoints personalizados compatibles con OpenAI.

## Elige un workspace

Haz clic en **Choose workspace**, añade el directorio del proyecto donde iniciaste `dsh` y selecciónalo. El compositor de sesiones permanece no disponible hasta que se selecciona un workspace.

## Ejecuta una tarea

Inicia una sesión y envía:

> Resume este repositorio e identifica sus paquetes principales.

El agent (agente) puede leer y editar archivos del workspace, ejecutar comandos, delegar trabajo y mantener un plan. La Web UI pregunta antes de las operaciones que requieren aprobación según la política de permisos activa.

## Continúa

- [Configura modelos](./providers.es.md)
- [Usa el SDK de Python](./python-sdk.es.md)
- [Usa otros modos de CLI](../../../apps/cli/README.es.md)
- [Desarrolla un plugin](../develop/basic/index.es.md)
