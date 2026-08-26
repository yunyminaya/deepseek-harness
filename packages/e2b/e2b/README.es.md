# @deepseek-ai/dsh-e2b

[English](README.md) | Español

Propietario compartido del ciclo de vida de un sandbox E2B. Los adaptadores de sistema de archivos y de subprocesos inyectan `ctx.e2b`, esperan su único handle del SDK y, por tanto, habitan el mismo árbol de trabajo Linux remoto y el mismo mundo de procesos. El paquete fija `e2b@2.29.1`; el [mapa de la familia](../README.es.md) enumera la composición opcional.

## Configuración

```yaml
- id: e2b
  name: '@deepseek-ai/dsh-e2b'
  config:
    cwd: /home/user/workspace
    timeoutMs: 300000

- id: subprocess-e2b
  name: '@deepseek-ai/dsh-subprocess-e2b'

- id: fs-e2b
  name: '@deepseek-ai/dsh-fs-e2b'
```

`apiKey` es opcional y, de lo contrario, lee `E2B_API_KEY`; la clave configura la conexión del SDK del host y nunca se instala en el sandbox. `cwd` usa por defecto `/home/user/workspace` y debe ser una ruta POSIX absoluta. `timeoutMs` usa por defecto cinco minutos y controla la vida del sandbox; al expirar, el sandbox se elimina.

## Ciclo de vida y propiedad

La construcción inicia una creación de sandbox. Antes de resolver `getSandbox()`, el servicio crea `cwd` y el directorio privado de estado del adaptador `cwd/.dsh-e2b`, verifica que la ruta reservada sea un directorio real y no un symlink ni otro tipo de archivo, y después la pone en modo `0700`. Cada shell de comando E2B interno del adaptador recibe un `HOME` raíz nuevo y aleatorizado, de modo que el shell de login fijo del SDK no resuelva archivos de perfil del home de usuario mutable antes del comando de control.

La eliminación primero impide adquirir nuevos handles, después espera la configuración y elimina el sandbox. Un `SandboxNotFoundError` significa que expiró o que otro propietario ya lo eliminó, y se acepta como quiescencia. Un fallo inicial de configuración de directorios produce un intento de eliminación; el timeout de E2B configurado acota un segundo fallo. Los plugins de providers deben cargarse después de este propietario y eliminarse antes que él.

## Experiencia del modelo

Ninguna, porque este propietario compartido del runtime no registra contexto visible para el modelo; los adaptadores de providers y sus consumers son dueños de cualquier efecto renderizado.

#### Efecto en la KV Cache

Sin invalidación directa; este paquete no aporta tokens de petición.

## Limitaciones conocidas y trabajo pendiente

- **No es un runtime de todo el harness** — los servicios de Cordis, el estado agent/sesión, los registros de sesión, las peticiones LLM, los skills y los buffers del lado del SDK permanecen en el proceso del host.
- **El estado del sandbox es efímero** — la eliminación y el timeout borran el sandbox; la reconexión, la retención de sesiones en pausa o abandonadas (pause/leave), las plantillas, los volúmenes y las instantáneas quedan fuera de este POC.
- **No hay plataforma de despliegue configurada** — la política de red, la sincronización del workspace del host y el descubrimiento de sandboxes quedan fuera de este POC.
- **`cwd` es una convención de resolución, no de contención** — los adaptadores y los comandos pueden direccionar otras rutas del sandbox; el acceso de red de E2B conserva la política de la imagen base.
