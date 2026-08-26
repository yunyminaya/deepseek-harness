# Agent Note: Incubar Agent Teams como paquetes experimentales privados

Status: implemented

[English](2026-08-18-experimental-agent-teams-packages.md) | Español

## Problema

Agent Teams necesita el registro de sesión real, el ciclo de vida de subagente, las herramientas, los ejemplos, las instantáneas y los chequeos del repositorio mientras sus contratos de servicio y herramienta siguen cambiando. Colocar esos paquetes en un grupo de rol de producto los convierte en miembros de la familia de release dsh y les da la misma expectativa de publicación que a los paquetes estables.

Un directorio experimental sin paquete actual imponía antes reglas de colocación, dependencias, promoción y release a ningún consumidor. Agent Teams suministra el consumidor concreto, pero el directorio necesita exclusión mecánica de release y aislamiento de dependencias en lugar de un estatus solo documental.

## Decisión

`packages/experimental/agent-team` y `packages/experimental/tool-agent-team` son paquetes de workspace privados. La [decisión de nombres de paquetes experimentales](2026-08-19-experimental-package-name-prefix.es.md) es dueña de sus nombres npm y del renombrado de promoción; esta nota es dueña de su colocación, exclusión de release y aislamiento de dependencias.

El conjunto de pack y publish de dsh y el publisher de baseline local excluyen todo manifest bajo `packages/experimental/`. `release:dsh` sigue avanzando sus versiones de manifest con la versión dsh compartida sin crear tags de release. Las restricciones de workspace exigen que cada paquete experimental fije `private: true` y omita `publishConfig`. El mismo chequeo de nivel superior rechaza `dependencies`, `optionalDependencies` y `peerDependencies` desde paquetes de release, apps de release o el runtime de Python hacia un paquete experimental. Los paquetes experimentales pueden depender de paquetes de release y entre sí; las pruebas pueden usarlos a través de `devDependencies`, y los ejemplos pueden cargarlos explícitamente.

La identidad continuable de hijo reservada para el llamador genérico y el drenaje selectivo de hijos directos permanecen en el servicio Subagent estable. Son dueños de la identidad de Subagent y del ciclo de vida de Activation sin importar ni nombrar a Agent Teams; el servicio de Team experimental los consume en la dirección permitida.

El estatus experimental cambia solo las expectativas de publicación y compatibilidad. Los paquetes conservan los requisitos ordinarios del repositorio de documentación, invariantes, ciclo de vida, seguridad, unitarios, de composición real e instantáneas. La promoción exige revisión de los contratos públicos, las limitaciones, la evidencia de pruebas, el payload de release, los dependientes de runtime y un dueño nombrado que acepte las obligaciones de paquete estable.

## Alternativas consideradas

**Mantener Agent Teams en un grupo de rol de producto y describirlo como opt-in.** La composición opt-in controla el comportamiento del modelo pero no excluye los paquetes de la publicación ni impide que los paquetes estables tomen dependencias de runtime sobre ellos.

**Reservar un grupo experimental vacío.** Un directorio sin paquete actual no tiene dueño ni mecanismo de release que probar. El grupo existe solo mientras los paquetes concretos necesiten su tratamiento aplicado.

**Mover los prerrequisitos de Subagent al directorio experimental.** La asignación de identidad de hijo y el teardown de Activation pertenecen al dueño de Subagent y no contienen contrato específico de Team. Moverlos o duplicarlos invertiría la dependencia o partiría un ciclo de vida entre paquetes.

## Consecuencias

Agent Teams puede usar el grafo completo del repositorio y los chequeos de calidad sin entrar en los tarballs oficiales ni convertirse en dependencia de runtime soportada. Un paquete de release no puede exponer Team hasta que los paquetes de Team sean promovidos, así que los experimentos de CLI y Web usan composiciones de ejemplo o experimentales explícitas en lugar de los bundles base entregados.

La agrupación por rol de producto es menos directa mientras los paquetes incuban. La promoción crea churn de rutas y nombres npm según lo especifica la decisión de nombres de paquetes experimentales.
