# Agent Note: Plegar la interfaz de persistencia en dsh-session

Status: rejected — el paquete separado de Service Definition de persistencia es la división de roles modular intencionada para el seam de capacidad de persistencia durable. Plegarlo en `dsh-session` reduciría el conteo de paquetes a costa de una frontera de backend más limpia.

[English](2026-06-20-fold-session-persistence-interface.md) | [中文](2026-06-20-fold-session-persistence-interface.zh.md) | Español

## Problema

`dsh-session-persistence` es un paquete de Service Definition cuyos conceptos principales ya son propiedad de `dsh-session`: `SessionHeader`, `SessionEvent`, `SessionId`, `session/event` y `session/flush`. El paquete añade el servicio abstracto `SessionPersistence`, el coordinador de escritura compartido y helpers de contrato. Los paquetes de provider dependen de él, y `agent-loop` tiene que encontrar opcionalmente un servicio hermano para el resume.

La división por seams de capacidad tenía sentido cuando la persistencia era un diseño nuevo de backend intercambiable. Tras eliminar el resumen mutable, el paquete de Service Definition mayormente envuelve la propia preocupación de almacenamiento del registro de sesión. Mantenerlo separado puede ser más ceremonia que claridad.

## Propuesta

Mover el servicio abstracto `SessionPersistence`, el coordinador y los helpers de contrato de persistencia a `dsh-session`. Mantener JSONL y SQLite como paquetes de backend separados que registran el servicio propiedad de la sesión. Esto preserva la intercambiabilidad del backend mientras borra un paquete de soporte y una frontera entre paquetes.

El PR implementador debería actualizar la guía de [capability seams](../../implemented/architecture/2026-06-13-capability-seams.es.md) con la excepción: la persistencia no es como bash o LLM porque su vocabulario y eventos de ciclo de vida ya son el dominio central del paquete de sesión.

## Criterios de aceptación

- `@deepseek-ai/dsh-session-persistence` se elimina como paquete.
- `dsh-session` exporta el tipo de servicio de persistencia, el coordinador y los helpers de contrato.
- Los paquetes de backend JSONL y SQLite dependen de `dsh-session` directamente.
- El resume de `agent-loop` usa la clave de servicio propiedad de la sesión.
- [Session persistence](../../implemented/architecture/2026-06-14-session-persistence.es.md), [shared persistence write coordinator](../../implemented/architecture/2026-06-18-shared-persistence-write-coordinator.es.md) y los [docs del paquete](../../../../packages/session/session-persistence/README.md) explican por qué las implementaciones de backend permanecen separadas.

## Qué se pierde

`dsh-session` se vuelve más pesado: posee tanto el registro en memoria como la Service Definition de persistencia. Ese es el trade. Si los backends de persistencia de terceros fueran ya un ecosistema público, el paquete separado de Service Definition sería una frontera SDK más limpia; pre-release, el paquete extra parece abstracción antes de que exista un Consumer externo.
