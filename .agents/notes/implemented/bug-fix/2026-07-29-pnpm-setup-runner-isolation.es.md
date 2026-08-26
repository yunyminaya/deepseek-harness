# Agent Note: Aislar el setup de pnpm por runner de GitHub Actions

[English](2026-07-29-pnpm-setup-runner-isolation.md) | [中文](2026-07-29-pnpm-setup-runner-isolation.zh.md) | Español

Estado: implementado

## Problema

`pnpm/action-setup@v4` fija por defecto su destino de instalación en `~/setup-pnpm` y reemplaza ese directorio durante el setup. El failover de CI autoalojado ejecuta seis servicios de runner de GitHub Actions bajo un único usuario de VM, así que los trabajos concurrentes compartían el mismo destino. En la ejecución que lo reprodujo, tres trabajos entraron en el setup de pnpm en 73 milisegundos; uno de los setups eliminó el directorio de trabajo actual de otro proceso y dos trabajos fallaron en la inicialización de `uv_cwd` de Node. Un reintento en otro runner pasó, lo que hacía que el fallo dependiera del momento y no fuera una regresión de las pruebas del repositorio.

## Decisión

Cada paso `pnpm/action-setup` de [el flujo de CI principal](../../../../.github/workflows/ci.yml) y [el flujo maestro](../../../../.github/workflows/ci-master.yml) establece `dest: ${{ runner.temp }}/setup-pnpm`. Cada servicio de runner es dueño de su directorio temporal, así que un setup no puede reemplazar el directorio de instalación de otro runner. La reutilización del almacén persistente sigue separada mediante `PNPM_CONFIG_STORE_DIR`, según estableció la [decisión de aprovisionamiento de pnpm](../process/2026-07-26-pnpm-action-setup-for-symmetric-ci-caching.es.md).

[La prueba de regresión del flujo](../../../../scripts/ci-workflow.spec.ts) descubre cada paso `pnpm/action-setup` de `ci.yml` y `ci-master.yml` y rechaza uno sin el destino privado del runner. Esto mantiene los trabajos recién añadidos dentro del mismo límite de aislamiento.

## Alternativas consideradas

**Serializar los trabajos del failover.** Rechazada porque descarta el paralelismo previsto del pool de seis runners y convierte una colisión de directorios local a la acción en una cola entre trabajos por lo demás independientes.

**Asignar un usuario Unix separado a cada servicio de runner.** Esto también separaría `HOME`, pero traslada el invariante al aprovisionamiento externo de la VM y complica la propiedad del almacén persistente de pnpm deliberadamente compartido. El flujo ya recibe un directorio temporal privado por runner.

**Reintentar los pasos de setup fallidos.** Rechazada porque los reintentos solo reducen la tasa de colisión observada; otro setup concurrente puede volver a eliminar el mismo directorio compartido.

## Consecuencias

La instalación del ejecutable de pnpm es efímera y aislada por runner, mientras que las descargas de paquetes siguen usando el almacén persistente o en caché configurado. Los trabajos alojados usan el mismo destino explícito sin cambiar la política de caché. El flujo lleva tres líneas de configuración extra por paso de setup, y la prueba de regresión solo debe actualizarse si el aprovisionamiento de pnpm se mueve intencionadamente a otro mecanismo de aislamiento.
