# Agent Note: Remove the separate CLI demo

Status: implemented

English | [中文](2026-08-08-remove-cli-demo.zh.md) | Español

## Problema

Después de que [`dsh --profile headless`](../architecture/2026-08-06-app-owned-command-line.es.md) se convirtiera en el comando de una sola vez del producto, `@deepseek-ai/dsh-cli-demo` seguía siendo un segundo paquete de aplicación para el mismo trabajo. Arrastraba otro ejecutable, gramática de argumentos, composición de app, ciclo de vida de cancelación, contrato de salida text/JSON/stream-JSON, artefacto construido, superficie documental y batería de pruebas. Los dos puntos de entrada además ensamblaban árboles distintos, así que una demo exitosa no probaba el perfil `headless` enviado y los usuarios tenían que elegir entre comandos solapados.

Las baterías de reproducción siguen necesitando eventos canónicos de sesión para fijar el comportamiento ensamblado del backend. Esa necesidad de pruebas no exige un comando publicado ni un contrato de compatibilidad.

## Decisión

Eliminar `@deepseek-ai/dsh-cli-demo` por completo: su paquete, bin, parser, plugin de app, formatos de salida, pruebas, referencias del workspace, entradas de catálogo generado y documentación activa. No queda ningún alias ni paquete de compatibilidad. Los usuarios de fuente invocan el comando del producto mediante `pnpm dsh --profile headless`; este es dueño del texto final por stdout, los diagnósticos de fallo por stderr, la persistencia, el estado de salida y el apagado.

`examples/headless-agent` pasa a ser una composición explícita de pruebas. Sus configuraciones del Loader montan `@deepseek-ai/dsh-agent-spine-demo`, un agente raíz, la persistencia JSONL y la política de checkpoints como filas separadas en lugar de ocultarlas tras un bundle de app. El paquete del nivel de soporte `@deepseek-ai/dsh-loader-smoke` es dueño del helper compartido de turno de agente directo; los drivers locales del ejemplo, no exportados, seleccionan su configuración del Loader y renderizan los eventos canónicos como JSONL. Solo las pruebas los lanzan, no tienen bin y no definen un formato de salida de producto soportado.

## Alternativas consideradas

- **Mantener `dsh-cli-demo` como alias o wrapper de `dsh --profile headless`.** Rechazada porque un segundo bin y paquete preservaría dos propietarios descubribles sin añadir capacidad.
- **Mover las banderas JSON y stream-JSON a `dsh --profile headless`.** Rechazada porque ningún consumidor actual del producto las exige; adoptar el viejo protocolo de la demo engordaría el contrato canónico de CLI solo para ahorrar maquinaria de pruebas.
- **Eliminar las instantáneas de eventos canónicos junto con el paquete.** Rechazada porque fijan un comportamiento ensamblado visible para el modelo que la aceptación de producto por texto final no puede observar.
- **Mantener el plugin de app pero eliminar solo su bin.** Rechazada porque la composición oculta seguiría duplicando el perfil headless explícito y escondería qué servicios monta la hoja de pruebas.

## Consecuencias

Este cambio rompe de forma intencionada. `dsh-cli-demo`, sus opciones de `--output-format` y los imports desde `@deepseek-ai/dsh-cli-demo/src/cli.ts` dejan de resolver. No hay sustituto público del flujo de eventos en este cambio; los llamadores usan `dsh --profile headless` para la ejecución de una sola vez y deben elegir una API de protocolo existente cuando necesiten automatización estructurada.

El repositorio conserva la cobertura de reproducción del backend mediante infraestructura exclusiva de pruebas, mientras que la prueba de humo del producto y la aceptación del bin construido ejercitan `dsh --profile headless`. Un paquete separado de una sola vez solo puede volver si es dueño de un protocolo versionado genuinamente independiente que no pueda pertenecer al lanzador del producto; una segunda grafía o un shim de salida no bastan.

## Verificación

Pruebas de humo del Loader centradas cubren la composición explícita en modo fuente y en modo construido bajo Node plano, las pruebas de instantánea comparan su JSONL canónico y sus registros persistidos, la aceptación del producto cubre `dsh --profile headless`, y las puertas de documentación y de grafo/catálogo generados rechazan referencias vivas al paquete eliminado. El archivo congelado de Agent Notes sigue siendo evidencia histórica y no se reescribe.
