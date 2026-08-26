# @deepseek-ai/dsh-storage-json

[English](README.md) | Español

Backend JSON para el [hub de almacenamiento](../storage/README.md.es.md): un archivo `<unit>.json` legible por humanos por unidad bajo una raíz configurada, registrado como backend `json`. Diseño: [Agent Note de almacenamiento KV de dominio](../../../.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.es.md).

## Modelo

- El estado de la unidad en memoria es autoritativo; cada primitiva de escritura republica el archivo completo mediante escritura temporal + fsync + sustitución atómica con `rename()`. Un archivo de unidad es siempre el estado neto actual completo — la legibilidad es la razón de existir de este backend; la escala es trabajo del backend SQLite.
- Un archivo ausente se abre como una unidad vacía y se materializa en la primera escritura. Un archivo extraño o inparseable rechaza con `malformed-medium`; una versión almacenada que difiere del descriptor rechaza con `version-mismatch` (sin migración, postura pre-release).
- El orden de escritura entre llamadas pertenece al llamador (la cadena de escritura de la capa de dominio); cada llamada individual es atómica y durable una vez que resuelve.

## Configuración

| Clave | Tipo | Valor por defecto | Significado |
|---|---|---|---|
| `root` | string | obligatorio — sin valor por defecto (un fallback a cwd dispersaría los archivos) | Directorio que contiene los archivos de unidad; se crea `0o700` bajo demanda |

## Experiencia del modelo

### Registros de dominio almacenados

#### Qué ve el modelo

Nada. Este backend no aporta prompt, herramienta ni schema; persiste datos de dominio no-sesión detrás de `ctx.storage` solo para consumidores del lado del host.

#### Efecto de tokens

Cero tokens en solicitudes en vivo.

#### Efecto de KV Cache

Ninguno — el backend nunca toca prefijos de solicitud en vivo.

## Limitaciones conocidas y trabajo diferido

- La durabilidad en Windows depende del `rename()` de libuv (`MoveFileExW` con sustitución) sin una bandera explícita de write-through; está previsto bajar aquí el helper de publicación write-through Win32 más estricto del backend de session-log cuando aterrice la faceta de append-log (consulta la sección de migración de la Agent Note).
- Sin bloqueo de escritura entre procesos: dos procesos escribiendo la misma raíz pueden intercalar sustituciones de archivo completo (la última escritura gana). El consumidor actual es el despliegue de un único proceso host; la historia multiproceso queda diferida según la tabla de fuera de alcance de la Agent Note.
