# Agent Note: El chrome de takeover de onboarding pasa al paso

[English](2026-08-06-onboarding-step-owned-takeover-chrome.md) | Español

Estado: implementado

## Problema

El shell de ajustes montaba el chrome de takeover de onboarding —un overlay portalado al body con una etapa opaca `--dsw-alias-bg-layer-1`, una máscara de desenfoque y `#root` en inert— en el momento en que se registraba un paso de `settings.onboarding` aún no completado localmente. Cada paso decide si realmente necesita mostrarse cargando primero un hecho privado (WelcomeNotice: el bit de reconocimiento a través de su join de ajustes; DeepSeekOnboardingDialog: la preparación de credenciales a través del join de Models) y renderiza `null` mientras ese hecho está en vuelo. Renderizar `null` no podía suprimir el chrome, porque la etapa opaca la pintaba el shell alrededor del outlet del slot, no el paso.

En cada recarga mientras el hero (en blanco o sin sesión) era el actual, la lista de sesiones al pasar a `ready` hacía aparecer una capa opaca a pantalla completa —blanca en la paleta clara— y bloqueaba toda interacción durante exactamente un ida y vuelta de RPC de credenciales/ajustes, tras el cual los pasos ya configurados se auto-completaban y la capa desaparecía. Los usuarios veían la aplicación destellar en blanco en cada recarga en el momento en que llegaban las listas de workspace/sesión.

## Decisión

El chrome de takeover pertenece al paso, no al shell. Una nueva primitiva sin cordis, `OnboardingSurface` (ui-primitives), renderiza el overlay/máscara/etapa portalado al body —nombres de clase CSS y geometría movidos verbatim desde `SettingsRoot.module.css`— y mantiene `#root` inert durante exactamente el tiempo de vida de su propio mount. Ambos componentes de paso envuelven en ella solo su rama **visible**; sus ramas `null` existentes ya no pintan ni bloquean nada por construcción, porque el chrome forma parte de la misma decisión de renderizado.

`SettingsRoot` conserva el coordinador exactamente como estaba (proyección de ledger ordenado, un paso montado, conjunto local de completados, moneda `stepId`/`complete`/`openSection`) pero renderiza el paso elegido desnudo —sin portal, sin etapa, sin efecto inert. El contrato del slot `settings.onboarding` ahora establece que los registrantes son propietarios del envoltorio de superficie y deben renderizar `null` mientras sus hechos privados no estén decididos.

## Alternativas consideradas

**Registrar los pasos condicionalmente (el ledger como señal de tiene-contenido).** Registrar la entrada solo después de que el join privado resuelva a «necesita intervención». Arquitectónicamente limpio (publicar en el punto de confirmación), pero un cambio mayor: la carga del join tendría que pasar de los diálogos al apply de cada plugin, y el registro/disposición se convertiría en plomería reactiva en dos paquetes. Rechazada por desproporcionada para el defecto.

**Convertir `settings.onboarding` en una cadena con un store externalizado del conjunto de completados.** Es el patrón de takeover del compositor; prototipado y revertido. Los selectores solo pueden juzgar props del propietario, así que los hechos privados de preparación aún debían resolverse dentro de los componentes —la cadena compraba una generalidad de enrutamiento que los dos pasos actuales no necesitan, al coste de un cambio de contrato en tres paquetes.

**Detectar la salida vacía del slot en el sitio de renderizado.** `renderSlot` devuelve un elemento outlet incondicionalmente, así que el propietario no puede ramificar sobre el `null` de un paso; sondear la vaciedad del DOM renderizado exige un baile de confirmar-y-retractar cuyas transiciones dinámicas pierden la garantía de antes-del-pintado.

## Consecuencias

Mientras un paso está montado pero indeciso, la aplicación permanece visible e interactiva: `#root` ya no es inert durante la ventana de decisión (antes lo era detrás de una capa opaca). Para un usuario genuinamente no configurado, el takeover aparece ahora un ida y vuelta de join más tarde que antes —pero con su contenido ya presente, en lugar de una etapa vacía que se rellena.

Un paso futuro que se registre sin envolver su contenido visible en `OnboardingSurface` se renderiza desnudo sobre la aplicación sin máscara; el JSDoc del contrato del slot nombra el envoltorio como obligación del registrante.

## Pruebas

`packages/client/ui-primitives/tests/onboarding-surface.client.spec.tsx` fija la primitiva: portal al body alrededor del contenido, presencia de las clases de máscara/etapa, `#root` inert mantenido exactamente durante el tiempo de vida del mount y la composición sin `#root`. `packages/client/ui-settings-general/tests/settings-root.client.spec.tsx` fija el contrato invertido del shell: sin chrome de takeover y sin inert mientras un paso montado no renderiza nada. `apps/web/tests/onboarding-deepseek-config.e2e.ts` gana el pin de regresión ensamblado del defecto: un mundo configurado se recarga mientras cada respuesta de `settings.describe` se mantiene abierta en la frontera de red del navegador —ampliando la ventana de decisión de los pasos de invisible-al-loopback a cientos de milisegundos, que es lo que mantiene las aserciones no vacuas— y un muestreador en página de 8 ms demuestra que el chrome de takeover nunca se monta y que `#root` nunca pasa a inert. Los escenarios existentes del archivo y los specs de los pasos (`ui-settings-general`, `ui-settings-models`) pasan sin cambios —los pines de selector de máscara y geometría sobreviven porque la hoja de estilos se movió verbatim.
