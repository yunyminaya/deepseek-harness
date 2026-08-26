# @deepseek-ai/dsh-pwsh-sandbox

[English](README.md) | Español

Implementación de PowerShell consumidora de sandbox del [seam de ejecución `ctx.shell`](../shell/): cada comando se ejecuta como `pwsh -NoLogo -NoProfile -NonInteractive -Command <command>` **confinado a través de `ctx.sandbox`**, con el modo seleccionado, la aplicación y los hechos de denegación marcados en cada resultado asentado. El gemelo pwsh de [`@deepseek-ai/dsh-bash-sandbox`](../bash-sandbox/), espejo llamada por llamada según la [decisión de ejecutor y herramienta pwsh](../../../.agents/notes/implemented/feature/2026-08-01-pwsh-tool-and-executor.es.md) — la sustancia del confinamiento es neutral respecto a la plataforma: en Windows el seam de sandbox resuelve a la cadena de runners de token restringido ACL ([`@deepseek-ai/dsh-sandbox-windows-acl`](../../sandbox/sandbox-windows-acl/)), en Linux/macOS a bwrap/Landlock/Seatbelt.

El ejecutor hereda la mecánica de procesos de [`@deepseek-ai/dsh-pwsh-local`](../pwsh-local/) y consume su seam a nivel de argv (`argv()` / `runArgv()` / `startArgv()` / `onProcessDone()`) para envolver la invocación exacta de pwsh a través del provider. La política de sandbox (modo + raíz del espacio de trabajo) NO es configuración de este paquete: viaja en cada llamada desde `ctx.sandboxPolicy` (las llamadas de herramienta pasan la política resuelta de la sesión llamante; las llamadas directas recurren a la política de despliegue).

## Comportamiento

- `danger-full-access`: los comandos se ejecutan sin cambios a través del ejecutor local; los resultados llevan `sandbox: { mode, denied: false }`.
- Modos confinados (`read-only`, `workspace-write`): el argv de pwsh se envuelve con `ctx.sandbox.confine()`; la negativa del runner a lanzar falla en modo cerrado con `SANDBOX_UNAVAILABLE` (throw en primer plano, hecho `runnerFailed` en segundo plano), y una escritura denegada se clasifica contra las `denialSignatures` del backend seleccionado en `sandbox.denied`.

## Experiencia del modelo

### El confinamiento funciona; la denegación aflora como fallo de comando

#### Lo que ve el modelo

El propio stderr del comando confinado (p. ej. `Access to the path '...' is denied.` bajo el runner ACL de Windows); la capa de herramientas convierte las denegaciones clasificadas en la superficie estándar de permiso denegado exactamente igual que para la herramienta bash.

#### Efecto en tokens

Ningún texto visible para el modelo más allá del stderr del comando y la superficie de denegación estándar de la capa de herramientas.

#### Efecto en la caché KV

Ninguno directamente; la superficie de denegación pertenece a la capa de herramientas.

## Limitaciones conocidas y trabajo diferido

- **Las lecturas no están restringidas** en Windows (el runner ACL restringe solo las escrituras); el límite de lectura está documentado en `@deepseek-ai/dsh-sandbox-windows-acl`.
- **La autoridad temp de workspace-write en Windows es privada** por par sesión/espacio de trabajo en vivo; las llamadas sin agent reciben un directorio privado nuevo por invocación. La raíz temp ambiental nunca se concede, y el runner reescribe TMP/TEMP al directorio privado antes de lanzar.
- **El modo read-only de Windows no concede ninguna raíz escribible explícita pero sigue siendo parcial** porque el token restringido debe conservar Everyone. Los objetos cuyo DACL concede a Everyone acceso de escritura — incluidos los opens compatibles del dispositivo NUL — siguen siendo autoridad ambiental; la redirección `> $null` de PowerShell sigue funcionando sin abrir NUL.
