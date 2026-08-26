# Agent Note: Cargar sesiones persistidas antes de la identidad de mensaje

Status: implemented

[English](2026-07-28-load-pre-identity-session-messages.md) | Español

## Problema

El cambio del mensaje inmutable identificado reemplazó cuatro cargas de evento duraderas por valores de mensaje completos. Las sesiones v0 existentes en JSONL y SQLite aún conservaban las formas inmediatamente anteriores: `content`/`source` directos en los eventos de usuario y de steering, `content`/`provenance` en los eventos de asistente, y `callId`/`content`/`isError` en los resultados de herramienta. Sus cabeceras seguían coincidiendo con `SESSION_FORMAT_VERSION`, pero la validación de forma actual las rechazaba antes de que la reanudación pudiera construir una `Session` activa.

Cambiar la representación del mensaje sin elevar la versión hizo que esos registros resultaran indistinguibles a nivel de cabecera de los registros v0 actuales. El runtime necesita una regla de importación estrecha que restaure datos creados por los backends de primera parte soportados sin debilitar la validación para eventos obsoletos o mal formados no relacionados.

## Decisión

`PersistenceCoordinator` normaliza las cuatro cargas de mensaje pre-identidad exactas tras la decodificación del backend y antes de la validación de mensaje actual. Envuelve sus campos semánticos existentes en la forma de mensaje actual específica del rol y asigna `legacy-message:<session-id>:<event-seq>` como `MessageId` importado determinista. Un reemplazo de contenido de `tool/result` heredado hereda el id importado de su objetivo de reemplazo, preservando el invariante actual de reescritura de solo contenido.

La misma normalización se ejecuta para `load`, `inspect`, un estado cargado sin dueño que reclama su sesión activa, y la adopción de prefijo HMR. Las comparaciones de prefijo comparan por tanto la semilla activa de forma actual con la misma vista almacenada normalizada. Los envoltorios de apariencia actual con campos ausentes o inválidos no se reparan, y el vocabulario de eventos no soportado, las cabeceras de solicitud, las versiones y las relaciones de superficie conservan sus rutas de rechazo existentes.

La actualización es de solo lectura. Los registros heredados almacenados permanecen sin cambios; una sesión reanudada solo añade eventos de forma actual después de ellos. Las identidades deterministas hacen que cargas repetidas y un registro mixto heredado/actual reproduzcan los mismos ids de mensaje sin una transacción de reescritura específica del backend.

## Alternativas consideradas

**Rechazar los registros bajo la postura de compatibilidad pre-lanzamiento.** Es el comportamiento por defecto para el ruido v0 no relacionado, pero deja varadas sesiones reales de primera parte aunque cada campo antiguo se corresponde sin ambigüedad con la representación actual de mensaje.

**Reescribir el registro almacenado completo in situ.** Canonicalizaría el artefacto pero violaría el contrato de almacenamiento de solo adición, exigiría mecanismos de reemplazo atómico separados para JSONL y SQLite, y expandiría una corrección de compatibilidad de lectura hasta convertirlo en un sistema de migración.

**Acuñar ids aleatorios en cada carga.** Los mensajes satisfarían la forma del tipo pero perderían identidad estable entre inspect, resume, reinicio y adiciones mixtas heredado/actual.

## Consecuencias

Las sesiones pre-identidad de JSONL y SQLite se reanudan con su contenido de mensaje original, fuentes, campos provider/model del asistente, correlación de herramientas, errores, metadatos y reemplazos de superficie. Los eventos devueltos son por lo demás indistinguibles de las instantáneas de mensaje importadas actuales y siguen profundamente congelados.

Esta es una excepción de importación explícita de misma versión, no una capa general de compatibilidad v0. Añadir otra excepción requiere otra correspondencia completa y sin ambigüedad en el límite de persistencia; los datos actuales mal formados siguen fallando en lugar de ser adivinados hacia la validez. El contrato compartido del coordinador ejercita la actualización contra los backends de referencia en memoria, JSONL y SQLite, incluida la recarga determinista y la identidad de reemplazo de resultado de herramienta.

## Relacionado

- [Crear cada mensaje como valor inmutable identificado](../architecture/2026-07-28-identified-immutable-message-values.es.md) — posee el contrato actual de identidad e inmutabilidad de mensaje.
- [La persistencia de sesión como servicio abstracto](../architecture/2026-06-14-session-persistence.es.md) — posee el backend de solo adición y el límite de reanudación.
