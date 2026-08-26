# @deepseek-ai/dsh-client-locale

[English](README.md) | Español

Plugin de locale: LocaleRuntime — la preferencia `zh`/`en` almacenada como `locale.preference` en `$DSH_HOME/settings.yaml`; cuando ese valor explícito del Host está ausente, un navegador nuevo arranca provisionalmente en el idioma que `navigator` pide (emparejamiento de subetiqueta primaria, con `en` cuando no pide ningún idioma que esta app distribuya). La lectura del Host se ejecuta después de la activación del plugin para que un servicio de ajustes no disponible no pueda bloquear la página; su resultado reemplaza el valor provisional del navegador en vivo. Los navegadores remotos conservan solo una selección local al proceso porque la API de ajustes es solo de loopback. `locale/change` se dispara en los cambios, y el plugin apunta `<html lang>` al locale activo (`zh-CN`/`en`) en la activación y en cada cambio. El servicio también es dueño del registro de diccionarios ns×locale (tipado `register(ns, {zh, en})` comprobado contra `LocaleNamespaceMap`, `bind(ns)`→`TranslateNS<ns>`; cadena de búsqueda ns → common → en → key), implementa el `LocaleFace` del sistema de slots y se instala a sí mismo a través de `ctx.slots.installLocale`, respaldando el asiento estándar `t` inyectado por el framework (`Translate`/`TranslateNS` son tipos de ui-slots; impórtalos desde allí — este paquete solo los re-exporta por conveniencia de los dueños de diccionarios). La [decisión de preferencias respaldadas por el Host](../../../.agents/notes/implemented/bug-fix/2026-08-06-host-backed-web-preferences.es.md) es dueña del límite de persistencia.

## Model Experience

Ninguna, ya que el registro de locale sirve el texto de la UI del navegador; nada de esto llega a una petición de modelo.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo diferido

- **Algunas superficies conservan texto en línea** — las filas de Ajustes, la barra lateral, el compositor de preguntas y el selector de modelos usan asientos de locale; otros paquetes siguen siendo dueños directos de texto estático.
- **El texto en manos del registro lee su traducción una vez** — el texto capturado en el momento del registro fuera de la ruta de render del slot (p. ej. la descripción del comando `/model` en el registro de comandos) conserva el idioma con el que se registró hasta el re-registro; el texto renderizado por slot sigue los cambios en vivo.
