# headless-agent

[English](README.md) | Español

Este directorio es dueño de la composición de prueba de replay y de modelo real para un coding agent headless: DeepSeek V4 + herramientas locales de bash y sistema de archivos + delegación de subagentes + flujos de trabajo e iteración Ralph con agent nuevo + `todo_write` + persistencia JSONL. Monta explícitamente el agent spine compartido, un agent raíz, la persistencia y la política de checkpoints; no es un segundo punto de entrada del producto.

## Ejecútalo

```sh
# repo root .env (gitignored) or exported env:
#   DEEPSEEK_API_KEY=sk-…
#   DEEPSEEK_BASE_URL=https://…   # optional; defaults to the public API
pnpm dsh --profile headless "fix the failing test in this workspace"
```

El comando de producto es [`dsh --profile headless`](../../apps/cli/README.es.md): acepta una tarea no vacía, crea y persiste una sesión nueva, imprime el texto final del asistente y sale.

Las suites de instantáneas ejecutan la configuración de este directorio a través de [`tests/fixtures/headless-driver.ts`](tests/fixtures/headless-driver.ts), un proceso solo de prueba no exportado que emite los eventos canónicos de sesión como JSONL antes de su registro de resultado. Ese flujo es infraestructura de prueba, no un formato de salida de CLI soportado. Las sesiones hijas solo aparecen a través de los eventos y resultados de herramientas del parent.

## Overlay del POC E2B

[`e2b.cordis.yml`](e2b.cordis.yml) sustituye los providers locales de sistema de archivos y subproceso por un sandbox E2B compartido, conservando `dsh-bash-local` y las mismas herramientas orientadas al modelo. Coloca `E2B_API_KEY` junto a `DEEPSEEK_API_KEY` en el `.env` de la raíz (en gitignore) y ejecuta la composición en vivo con compuerta de credenciales, que maneja FS, Bash, PTY y LSP en un solo sandbox y prueba la eliminación final:

```sh
pnpm exec vitest run --config vitest.e2e.config.ts packages/e2b/e2b/tests/composition.e2e.ts
```

El overlay crea el mismo cwd absoluto dentro del sandbox, pero no sube ni monta el workspace del host. Las mutaciones de archivos y Bash existen solo en E2B; Cordis, las llamadas al modelo, el estado de agent/sesión, los logs de sesión, los skills y los buffers del SDK permanecen en el host. La composición termina su sandbox al producirse el timeout y la liberación. Es un POC de composición de providers, no una migración del harness completo ni una funcionalidad de sincronización de workspace.

## Configuración avanzada

[`advanced.cordis.yml`](advanced.cordis.yml) añade Code Mode y las herramientas de Cordis a la composición de prueba.
