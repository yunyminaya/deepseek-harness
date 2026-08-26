# Agent Note: Source run without a managed installer

Status: implemented

English | [中文](2026-08-10-source-run-without-managed-installer.zh.md) | Español

## Problema

Un instalador de fuente propiedad del repositorio puede proporcionar un lanzador estable, worktrees de preparación aislados, subidas atómicas de versión, almacenamiento de reversión y flujos de mantenimiento compartidos para personalizaciones propias. También hace al repositorio responsable de un segundo ciclo de vida además del gestor de paquetes: instalación de dependencias del host, petición de credenciales, adopción de checkouts, propiedad de symlinks, coordinación de ramas de preparación, recuperación tras subida de versión y compatibilidad continuada entre el instalador y las skills de mantenimiento incluidas.

Ese ciclo de vida no es necesario para ejecutar ni desarrollar DeepSeek Harness desde un checkout de fuente. Mantenerlo amplía el espacio de estados de sistema de archivos y Git soportado sin mejorar el camino de ejecución nativo del repositorio.

## Decisión

El repositorio soporta la ejecución desde fuente a través de sus scripts `pnpm` de raíz. La entrada `dsh` de `package.json` lanza `apps/cli/src/bin.ts` directamente con `node --import tsx/esm`; la generación de artefactos es la operación separada `pnpm run build` definida por la [decisión de separar lanzamiento desde fuente y build](2026-08-12-separate-source-launch-from-build.es.md). El script del paquete reenvía los argumentos y hereda el entorno del llamador, incluido `NODE_USE_ENV_PROXY=1` cuando una versión de Node que lo soporta debe respetar `HTTP_PROXY` y `HTTPS_PROXY`. Los usuarios eligen Web con `pnpm dsh web` y la ejecución headless con `pnpm dsh --profile headless "task"`. El ejemplo independiente de ACP sigue disponible mediante `pnpm run demo:acp`.

El repositorio no distribuye un instalador de fuente, ni una batería de pruebas del instalador, ni skills que asuman un symlink gestionado `current` y worktrees de preparación con marca de tiempo. Los usuarios son dueños de la ubicación del checkout de fuente, de las actualizaciones de Git y de cualquier lanzador que creen fuera del repositorio.

## Alternativas consideradas

**Mantener el instalador pero documentar `pnpm run` como otro camino.** Esto conserva el lanzador gestionado y la capacidad de reversión, pero mantiene activos ambos contratos de ciclo de vida, incluidas las pruebas del instalador y las skills conscientes de la preparación.

**Mantener las skills genéricas de personalización y de publicación aguas arriba.** Sus reglas de seguridad pueden aplicarse más allá del diseño de preparación, pero los flujos enviados forman un sistema de mantenimiento acoplado: la personalización descubre el checkout de preparación instalado, la subida de versión realiza el relevo y la publicación aguas arriba se selecciona desde esos cambios personales. La guía general de contribución con Git ya pertenece a las instrucciones del repositorio y no exige skills incluidas en el producto.

**Sustituir el instalador por un script ligero de enlace de lanzador.** Esto reduce el comportamiento de instalación, pero sigue haciendo al repositorio responsable de mutar el PATH del host y de la propiedad del lanzador. Los scripts de fuente aportan los puntos de entrada sin ese estado.

## Consecuencias

Los usuarios de fuente invocan scripts del repositorio en lugar de un comando `dsh` instalado. El repositorio no proporciona relevo atómico de subida de versión ni checkout de reversión de preparación preservado, y no automatiza la integración ni la publicación aguas arriba de las modificaciones personales de la fuente. Un futuro mecanismo de distribución debe justificar su propiedad del estado de instalación y subida de versión, definir el comportamiento de recuperación y añadir pruebas y documentación de usuario sin que el camino de ejecución desde fuente dependa de él. Cualquier flujo futuro de publicación debe aislar una funcionalidad aprobada y obtener aprobación explícita antes de su primer push y PR de borrador.

La verificación cubre las referencias de todo el repositorio a los puntos de entrada eliminados, los enlaces de documentación, la frescura del aviso de terceros generado, el comando de fuente directo de `package.json` y una prueba de humo de CLI desde fuente a través del vector exacto de runtime `node --import tsx/esm`.
