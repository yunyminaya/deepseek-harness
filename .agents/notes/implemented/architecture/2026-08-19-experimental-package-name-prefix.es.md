# Agent Note: Marcar los paquetes experimentales en los nombres npm

Status: implemented

[English](2026-08-19-experimental-package-name-prefix.md) | Español

## Problema

La colocación de directorios, los manifests privados y el filtrado de la familia de release mantienen los paquetes experimentales fuera de los releases, pero un especificador npm o una fila de configuración de Cordis no exponen ese estatus. Un nombre de paquete de aspecto estable puede copiarse en otra composición sin que el lector vea que su contrato público completo sigue siendo experimental.

## Decisión

Todo paquete directamente bajo `packages/experimental/` usa el prefijo npm `@deepseek-ai/dsh-experimental-*`. El gate de restricciones de workspace descubre esos manifests y rechaza un prefijo ausente junto a los requisitos existentes de `private: true` y `publishConfig` omitido.

Agent Teams usa `@deepseek-ai/dsh-experimental-agent-team` desde `packages/experimental/agent-team` y `@deepseek-ai/dsh-experimental-tool-agent-team` desde `packages/experimental/tool-agent-team`. Los imports de paquetes, las filas de configuración de Cordis, los catálogos generados y los metadatos del repositorio usan esos nombres sin alias de compatibilidad.

La promoción mueve un paquete a su grupo de rol de producto, retira `experimental-` de su nombre npm y actualiza cada referencia del repositorio de forma atómica. La política de compatibilidad de pre-release permite ese renombrado sin un paquete de alias.

## Alternativas consideradas

**Conservar nombres npm de aspecto estable usando solo el directorio y los metadatos de release para el estatus experimental.** Esto minimiza el churn de promoción, pero los especificadores de import y las filas de configuración ocultan el estatus del paquete y no pueden llevar la regla de colocación solo-repositorio a la revisión.

**Usar un sufijo experimental.** Un prefijo agrupa todo paquete experimental bajo un namespace npm buscable y hace visible el estatus antes del rol de producto; un sufijo dispersaría ese marcador después de los nombres específicos del rol.

## Consecuencias

Los imports experimentales y las filas de configuración identifican su estatus de soporte sin consultar el layout del repositorio. El comando de restricciones de nivel superior y su test unitario enfocado impiden que un paquete experimental recién añadido omita el prefijo.

La promoción renombra deliberadamente imports, configuración, referencias generadas y metadatos. Ningún paquete de compatibilidad preserva el nombre experimental.
