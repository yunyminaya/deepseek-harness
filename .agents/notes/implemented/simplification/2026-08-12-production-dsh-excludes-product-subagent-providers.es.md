# Agent Note: Production dsh excludes product subagent providers

Status: implemented

English | [中文](2026-08-12-production-dsh-excludes-product-subagent-providers.zh.md) | Español

## Problema

`@deepseek-ai/dsh` recibe la clausura de dependencias de `@deepseek-ai/dsh-base`. Incluir ahí los providers de subagente de Codex y Claude Code hace que cada instalación de producción descargue código de integración opcional de productos y payloads grandes de CLI de plataforma, aunque no se use ninguna de las dos integraciones.

## Decisión

Esta decisión sustituye parcialmente solo la parte de inclusión por defecto de la [ubicación en el host compartido](../architecture/2026-08-10-product-subagent-providers-in-shared-host.es.md): `@deepseek-ai/dsh-base` no depende de los providers de subagente de Codex y Claude Code ni los monta. Cada paquete de provider es un Profile Bundle instalable directamente cuyo `dsh.bundle.patch` apunta a un único `cordis.patch.yml` propiedad del paquete. Cada parche aporta exactamente una fila de Host auto-provider y ninguna fila de herramienta Agent.

Los dos Bundles siguen siendo independientes. El Bundle de Codex es dueño del wrapper oficial fijado y de seis alias de plataforma; producción lanza el wrapper declarado por el paquete y nunca recurre a un `codex` del host. El Bundle de Claude Code es dueño del Agent SDK fijado y de la CLI de plataforma correspondiente; producción deja que el SDK seleccione esa CLI privada y nunca recurre a un `claude` del host. Instalar un Bundle no arrastra al otro, y la clausura de producción por defecto de `@deepseek-ai/dsh` no contiene ninguno de los dos providers ni ningún runtime de los productos. Cada Bundle instalado registra un provider dormido en el siguiente arranque del Profile, mientras que un Agent Preset decide de forma independiente si una nueva Session recibe la herramienta correspondiente. La instalación no inicia ningún producto, no autentica ninguna cuenta, no reescribe ajustes nativos ni concede acceso al modelo.

## Verificación

Las pruebas de paquete fijan ambos manifiestos de Bundle, los parches publicados, las filas exactas de auto-provider y las dependencias de runtime de los productos. La cobertura de Claude fija Agent SDK 0.3.220, Claude Code 2.1.220, los ocho paquetes de plataforma, la ejecución seleccionada por el SDK y el fallo por payload ausente sin recurrir al host. La cobertura de Codex fija el wrapper 0.147.0, los seis alias de plataforma, la ejecución declarada por el paquete, la quietud de los descendientes nativos y el mismo comportamiento de payload ausente. La validación del workspace deriva cada parche publicado de la declaración de su Bundle, no de un catálogo de paquetes. Las aserciones de package/base más evidencia real de producción con pnpm prueban las fronteras de dependencia por defecto y de producto seleccionado, mientras que la composición real de parches de Bundle y de Agent Preset cubre ninguno, cualquiera de los dos productos, ambos, la intersección de concesión de herramientas, la adopción en Sessions posteriores y cero procesos al arrancar.

## Alternativas consideradas

**Mantener los providers dormidos en el bundle base.** Los providers dormidos no inician procesos de ningún producto, pero sus paquetes siguen entrando en cada `npm install` de producción.

**Añadir un wrapper o un meta Bundle.** Un tercer paquete duplicaría la propiedad de la instalación y haría que la eliminación independiente fuera menos directa, sin aportar otra capacidad de runtime.

## Consecuencias

Instalar `@deepseek-ai/dsh` no descarga ningún provider de producto a través del bundle base. Un Profile puede añadir o quitar cada Bundle de provider de forma independiente; los cambios de disponibilidad de Host surten efecto en el siguiente arranque del Profile, y seleccionar un producto explícitamente acepta su payload privado de plataforma. Un Agent Preset redactado por separado concede cada herramienta visible para el modelo solo a las Sessions recién compuestas. No se introduce ningún paquete wrapper más allá de las distribuciones oficiales de los productos, ni meta Bundle, ni instalador dinámico, ni estado persistido de producto habilitado.
