# Agent Note: Eliminar la vía dedicada del Plugin de repositorio

Status: implemented

[English](2026-08-09-remove-repository-plugin.md) | [中文](2026-08-09-remove-repository-plugin.zh.md) | Español

## Problema

La vía del Plugin de repositorio duplicaba la vía del bundle de perfil para instalar y componer paquetes de terceros. Añadía un manifest `.dsh-plugin`, un envoltorio generado, un ejecutable de preparación, una segunda caché de Git/paquetes, un builtin del Loader y adapters de Skill y MCP específicos de repositorio. Los bundles de perfil ya instalan especificaciones de paquetes npm o Git a través del gestor de paquetes del perfil, retienen la semántica normal de dependencias y ciclo de vida, y aportan una capa `cordis.patch.yml` ordenada capaz de montar Plugins de Cordis ordinarios.

La vía duplicada además exponía menos configuración que un bundle. Su lista `repositories` seleccionaba cadenas de fuente, pero el envoltorio generado montaba una entrada de código sin una config de Plugin suministrada por el usuario. La preparación específica de repositorio añadió por tanto código y trabajo de CI sustanciales sin convertirse en el mecanismo general de distribución de Plugin externo.

## Decisión

DeepSeek Harness tiene una única vía de distribución autónoma de Plugin externo: los bundles de perfil instalables. `dsh plugin --profile <name> add <package-or-git-spec>` registra la dependencia en el paquete del perfil, y el paquete instalado declara `dsh.bundle.patch` para aportar su capa de parche. El gestor de paquetes es dueño de la adquisición de fuente, las versiones, las dependencias, los ciclos de vida de construcción y su lockfile. El parche del bundle es dueño de la selección de Plugins de Cordis y de la config completa del Plugin.

Se eliminan el paquete `@deepseek-ai/dsh-repository-plugin`, el formato de autoría `.dsh-plugin`, el ejecutable `dsh-plugin-prepare`, el envoltorio generado, la caché inmutable de repositorios, la fila `repository-plugins` base y el carril de aceptación dedicado de GitHub. El subpath vendored sin uso `@cordisjs/plugin-loader/repository` y su dependencia pnpm empaquetada se eliminan con su único consumidor. Los directorios de caché de repositorios existentes son datos inertes del usuario; DSH ni los lee ni los elimina.

Los bundles componen los dueños existentes directamente. Un bundle que aporta Skills monta `@deepseek-ai/dsh-skill-filesystem`; uno que aporta servidores MCP monta `@deepseek-ai/dsh-mcp-client`; el comportamiento nativo monta un Plugin de Cordis compilado ordinario. Estos paquetes retienen sus propios contratos de validación, ciclo de vida, registro y desmontaje. No se retiene ningún analizador de compatibilidad ni migración desde `.dsh-plugin` bajo la política de compatibilidad de prerrelease.

Esta nota consolida las decisiones de la caché de repositorios eliminada, el formato estático, la integración de solo-config, la preparación respaldada por npm y la entrada de código confiable. Su motivación original sobrevive aquí: los usuarios autónomos necesitan composición externa propiedad del gestor de paquetes, las dependencias de Git y npm pueden ejecutar código confiable de ciclo de vida, las contribuciones estáticas de Skill y MCP deben reutilizar sus dueños existentes, y la identidad de la fuente pertenece a la especificación de dependencia del perfil y al lockfile. Sus envoltorios específicos de la implementación, generaciones de caché y protocolo de preparación ya no constriñen el producto.

## Alternativas consideradas

**Mantener el Plugin de repositorio como envoltorio de conveniencia sobre los bundles.** Rechazado porque preservaría dos comandos de instalación, dos formatos de manifest y dos identidades de fallo/caché para el mismo paquete. Una conveniencia incapaz de pasar config ordinaria de Plugin sigue además siendo menos capaz que el mecanismo que envuelve.

**Enseñar al envoltorio de repositorio a cargar un parche de bundle.** Rechazado porque la caché de repositorios y el protocolo de preparación seguirían duplicando la instalación de dependencias del perfil. Los paquetes de bundle ya se aceptan desde especificaciones npm, Git, file y link a través de pnpm.

**Mantener la caché de repositorios genérica del Loader por posibles consumidores futuros.** Rechazada porque no tiene consumidor actual tras eliminar el paquete y arrastra un runtime de gestor de paquetes fijado dentro de un paquete vendored adyacente al navegador. Una caché dedicada vuelve a justificarse solo si la activación en tiempo de configuración sin instalación explícita se vuelve un requisito del producto que las dependencias de perfil no puedan satisfacer; ese consumidor podrá elegir entonces su contrato de caché.

**Deshabilitar el Plugin de repositorio pero retener su formato en disco para migración.** Rechazado bajo la postura de prerrelease. Retener un analizador o un loader de compatibilidad mantendría vivo el contrato eliminado sin una obligación externa de compatibilidad.

## Consecuencias

- Los paquetes de terceros usan un único modelo de instalación y composición, con declaraciones ordinarias de dependencia y config completa de Plugin a nivel de parche.
- Instalar o actualizar un bundle externo es una operación explícita del gestor de paquetes `dsh plugin` y no una edición vigilada de la lista de fuentes. El HMR del parche de usuario sigue configurando filas aportadas por bundles instalados.
- La instalación de perfil exige `pnpm` en el `PATH` del host. Es aceptable para una operación explícita de gestión de paquetes y evita embarcar el runtime de gestor de paquetes fijado de la caché eliminada solo para la activación en tiempo de configuración.
- Los paquetes `.dsh-plugin` y los parches existentes de lista de fuentes de repositorio dejan de funcionar. Sus archivos de caché siguen siendo eliminables por el usuario pero no se migran ni se borran automáticamente.
- El runtime pnpm dedicado, el ejecutable de preparación, el generador de envoltorios, la configuración de CI de credenciales Git, la caché de repositorios y las pruebas específicas de repositorio desaparecen.
- Los recursos estáticos relativos al paquete necesitan una forma de ruta propiedad del bundle para que un bundle declarativo pueda apuntar a `dsh-skill-filesystem`, `dsh-mcp-client` u otro Plugin a archivos que embarca, sin pegamento de runtime a medida. Esa capacidad es propiedad del formato de bundle y no de un adapter de repositorio.

## Pruebas

Las puertas estáticas rechazan referencias obsoletas de paquete, config, documentación, grafo y workspace. La aceptación existente de CLI construido de `dsh plugin` cubre la inicialización de perfil, la instalación por gestor de paquetes, el descubrimiento de bundles y la reconciliación de capas. Los recursos declarativos de Skill y MCP relativos al paquete siguen siendo una brecha de cobertura nombrada en esta capa de eliminación.
