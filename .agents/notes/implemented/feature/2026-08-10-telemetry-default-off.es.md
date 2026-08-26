# Agent Note: SessionTelemetryBackend exige opt-in explícito

Status: implemented

[English](2026-08-10-telemetry-default-off.md) | Español

## Problema

DeepSeek Harness tiene dos flujos de telemetría saliente. Durante las pruebas internas, la base compartida montaba la telemetría con un endpoint de producción empotrado, y ambos flujos informaban por defecto para ayudar a diagnosticar los problemas reportados: el backend OTel de sesión podía exportar contenido completo de sesión, datos de herramienta, prompts y rutas de workspace cuando su modo se omitía, mientras que el flujo del launcher dsh-sdk lo hacía incondicionalmente. Una instalación nueva por tanto permitía el reporte saliente sin una elección positiva de despliegue.

## Decisión

Ambos flujos usan `DSH_TELEMETRY_MODE` como su ajuste de consentimiento positivo. Los valores no establecidos y vacíos se resuelven a `DISABLED`. `@deepseek-ai/dsh-session-telemetry-otel` también resuelve un `mode` omitido a `DISABLED`, lo que no construye ningún provider OTel, processor ni exporter y deja el feedback en el log de sesión local. La base dsh compartida conserva montada la fila de backend para que el feedback desactivado pueda explicar aun así que no se compartió nada. Un despliegue opta por compartir el Session Log a través de `FULL` o `FEEDBACK_ONLY`; solo `FULL` permite además el reporte del launcher dsh-sdk. Cualquier `DSH_TELEMETRY_DISABLED` no vacío sigue siendo un opt-out duro autoritativo antes de la carga. La [decisión de montaje por defecto](2026-07-31-web-telemetry-default-mount.es.md) sigue siendo dueña del endpoint, la cadencia de batching y los ajustes de drenaje en salida.

El launcher dsh-sdk lee la misma variable sin parsear `cordis.yml` ni arrancar Cordis. `FULL` permite el reporte; `FEEDBACK_ONLY`, `DISABLED`, los valores no establecidos y los vacíos lo deniegan. El consentimiento se congela desde el entorno lanzador antes de que el comando se ejecute, porque `dsh-sdk start` carga un `.env` de proyecto y el código del proyecto puede mutar `process.env`: resolver después permitiría que un proyecto concediera el reporte de su propia configuración, lo que la [decisión de titularidad de fuentes de configuración](../architecture/2026-08-04-configuration-source-ownership.es.md) deniega para todo el espacio de nombres `DSH_*`. Un modo no soportado deniega en lugar de lanzar en ese límite, ya que la telemetría nunca puede cambiar el resultado de un comando. Esta regla sustituyó al consentimiento del launcher activado por defecto antes de que el launcher y su propuesta fueran eliminados por la [eliminación del toolchain de proyectos SDK](../simplification/2026-08-11-remove-sdk-project-toolchain.es.md).

El [README de referencia CLI](../../../../apps/cli/reference/README.es.md) documenta la postura de despliegue: la subida del Session Log está desactivada por defecto, `DSH_TELEMETRY_MODE=FEEDBACK_ONLY` y `DSH_TELEMETRY_MODE=FULL` son las dos elecciones de opt-in, y las exportaciones explícitamente habilitadas pueden contener contenido completo de sesión. El [aviso de onboarding de etapa de pruebas](2026-08-13-shared-modal-product-onboarding.es.md) restaurado no contiene texto de telemetría, así que el producto sigue sin presentar ningún prompt sobre habilitar la subida.

## Alternativas consideradas

**Conservar los valores por defecto de opt-out y mejorar la divulgación.** Rechazado porque la divulgación no convierte una configuración ausente en una autorización positiva para enviar datos, especialmente cuando la telemetría de sesión puede contener contenido local completo.

**Fijar la telemetría de sesión en `FEEDBACK_ONLY` por defecto.** Rechazado porque registrar feedback seguiría disparando una subida sin que un despliegue habilitara explícitamente el reporte saliente. El valor por defecto debe mantener local tanto la sesión como su feedback.

**Añadir marcadores de consentimiento a nivel de proyecto.** Rechazado porque `DSH_TELEMETRY_MODE` ya expresa el consentimiento para ambos flujos; otra entrada de configuración crearía ajustes en conflicto y exigiría parseo específico del launcher.

**Eliminar ambas implementaciones de telemetría.** Rechazado porque los despliegues internos siguen necesitando el reporte explícito `FULL` y el limitado a feedback, y el flujo del launcher sigue siendo útil bajo `FULL`.

## Consecuencias

Los perfiles y proyectos nuevos no hacen ninguna solicitud de red de telemetría. Los despliegues internos seleccionan un modo para ambos flujos: `FEEDBACK_ONLY` permite solo el compartir Session Log disparado por feedback, mientras que `FULL` habilita además el reporte del launcher. El opt-out duro existente sigue siendo efectivo, y los modos de subida conservan su validación de endpoint, responsabilidad de redacción, batching y comportamiento de apagado.
