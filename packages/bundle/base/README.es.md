# `@deepseek-ai/dsh-base`
[English](README.md) | Español

El núcleo dsh compartido como bundle de profile: [`cordis.patch.yml`](cordis.patch.yml) inserta cada fila de plugin de base — adaptadores de modelo, la selección compartida de [`agent-default-model`](../../core/agent-default-model/README.es.md), herramientas, persistencia, política, ajustes/credenciales, telemetría y los providers de subagentes spawn/fork del núcleo — sobre la raíz de profile vacía, como primera capa de la lista `dsh.profile.bundles` de cada profile. Los providers opcionales de Codex y Claude Code quedan fuera de este paquete y de su cierre de dependencias de producción; un Profile instala cualquiera de los dos [Bundles de provider de producto](../../subagent/README.es.md) solo cuando lo necesita. El cierre de producción predeterminado de `@deepseek-ai/dsh` por tanto no incluye ninguno de los dos providers de producto, ni el Claude Agent SDK, ni el wrapper de Codex y sus cargas de plataforma. Las capas de bundle posteriores (p. ej. [`dsh-web-app`](../web-app/README.es.md)) y el `cordis.patch.yml` de profile del usuario sobrescriben estas filas por id; un patch reemplaza todo el `config` de una fila, así que los valores específicos de modo viven en los bundles de modo, no aquí. El paquete no tiene API de runtime; el compositor de profiles resuelve el patch a través del campo de manifest `dsh.bundle.patch`, nunca mediante código.

El patch condiciona ambas pilas de shell por plataforma en sus propias filas: `bash-sandbox`/`tool-bash` llevan `disabled: !!js process.platform === 'win32'` (bash no tiene runner en Windows), y sus gemelos `pwsh-sandbox`/`tool-pwsh` se montan solo en win32 con la expresión invertida — un único archivo de patches compartido, exactamente una pila de shell por host. La superficie de permisos se mantiene exactamente como en POSIX: `sandbox`/`sandbox-policy` imponen la política de efectos de archivo mediante el runner de token restringido de ACL de Windows (la cadena win32 de `dsh-sandbox-local` → `@deepseek-ai/dsh-sandbox-windows-acl`), el conmutador de permisos y el servicio de aprobación se ejecutan sin cambios, y `fs-sandbox` sigue cercando las escrituras de `ctx.fs` — montar `dsh-fs-local` junto a él duplicaría el registro de `ctx.fs` y haría fallar la carga. Un host de Windows que prefiera el ejecutor pwsh local sin confinar o el acceso completo sobrescribe estas filas mediante su profile o el `cordis.patch.yml` de home (la receta de restauración de bash debe ser completa: desactivar `pwsh-sandbox`/`tool-pwsh` Y reactivar `bash-sandbox`/`tool-bash` — ambas familias de ejecutores registran el mismo servicio `bash`, así que una receta incompleta falla en voz alta en la carga). Los hosts POSIX ven desactivadas las filas de pwsh.

El conjunto de filas y su justificación se documentan en línea en el archivo de patches; el [grafo de composición generado](../../../apps/cli/composition.md) lo renderiza.

## Experiencia de modelo

Indirectamente, mediante las filas insertadas: este bundle selecciona la base de prompts sin persona distribuida, el conjunto de herramientas y el adaptador de DeepSeek que los bundles de modo especializan, y no aporta texto visible para el modelo propio.

#### Efecto de KV Cache

Ninguno directamente; el paquete de cada fila insertada es dueño de su efecto.

## Limitaciones conocidas y trabajo diferido

- **Un patch reemplaza configs de fila enteras** — las sobrescrituras de profile deben reafirmar cada campo que una fila conserva; no hay capa de deep-merge.
- **La concesión de temp de Windows es un subdirectorio privado por sesión** — `workspace-write` confina las escrituras al workspace más el subdirectorio temp propio de la sesión (`<temp>\dsh-<hash>`, TMP/TEMP reescritos para los hijos confinados); `read-only` no concede nada. Ver `@deepseek-ai/dsh-sandbox-windows-acl`.
