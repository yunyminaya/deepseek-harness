# Agent Note: Eliminar el aviso de beta de primera ejecución

Status: implemented

[English](2026-08-13-remove-first-run-beta-notice.md) | [中文](2026-08-13-remove-first-run-beta-notice.zh.md) | Español

## Problema

Cada primer lanzamiento de la GUI abría con una declaración de prueba interna (内测声明) a pantalla completa: encuadre de beta interna más instrucciones para habilitar la subida del Session Log a través de `DSH_TELEMETRY_MODE`. La telemetría de sesión ya resuelve a `DISABLED` cuando su modo no está fijado ([telemetry default-off](../feature/2026-08-10-telemetry-default-off.es.md)), así que el único contenido de onboarding sobre telemetría era un prompt que explicaba cómo activarla, y el encuadre de prueba interna en sí no debe salir en una build de release.

## Decisión

Esta decisión eliminó el aviso de primera ejecución del producto ensamblado en lugar de reformularlo. `ui-settings-general` no asentó ningún paso `settings.onboarding`; el componente del aviso, el store de reconocimiento, el dueño del copy y las claves de locale se borraron, mientras el Host conservó el namespace `ui-onboarding` para que los documentos almacenados siguieran siendo válidos. El posterior [shared-modal product onboarding](../feature/2026-08-13-shared-modal-product-onboarding.es.md) restaura un aviso nuevo y conciso de etapa de pruebas en `ui-settings-models`, reutilizando ese campo y contrato de backend sin restaurar el layout de takeover eliminado ni las instrucciones de telemetría. El opt-in de telemetría sigue siendo una elección explícita del entorno de despliegue documentada en el [README de referencia de la CLI](../../../../apps/cli/reference/README.md); el aviso restaurado no dice nada sobre habilitarla.

## Alternativas consideradas

**Conservar el aviso y solo quitar su párrafo de telemetría.** Rechazada: el encuadre de prueba interna es lo que un release no debe presentar, y una intersticial obligatoria de primera ejecución sin declaración material restante es fricción pura.

**Pedir consentimiento de subida en su lugar (un paso de consentimiento versionado).** Rechazada para este release: una pregunta de primera ejecución sobre habilitar la subida sigue siendo un prompt de telemetría. Un flujo de consentimiento futuro puede registrarse a través del seam `settings.onboarding` sin cambios y usar un campo versionado nuevo para el reconocimiento.

**Desregistrar también el namespace `ui-onboarding`.** Rechazada: los documentos de ajustes existentes ya llevan la sección, y el seam de ajustes valida los documentos almacenados contra namespaces registrados; mantener el registro mantiene válidos esos documentos sin coste.

## Consecuencias

Esta eliminación suprimió el aviso a pantalla completa y su copy de telemetría. La restauración posterior es intencionadamente una presentación y revisión de copy distintas: un modal compartido precede al diálogo de credenciales inline, el escenario remoto vuelve a cubrir el reconocimiento local al proceso, y el campo `welcomeNoticeVersion` existente registra la nueva versión del copy. El prompt histórico de telemetría sigue ausente.
