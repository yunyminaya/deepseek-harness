# guard/ — familia de guardas de higiene del loop

[English](README.md) | Español

Los plugins de guarda de comportamiento vigilan el agent loop en busca de patrones improductivos y aplican presupuestos por llamada. Una guarda es un Consumer autocontenido de servicios centrales y puntos de extensión, no una capacidad intercambiable.

| Paquete | Rol | Clave ctx |
|---|---|---|
| [`repeat-tool-reminder/`](repeat-tool-reminder/README.es.md) | Recordatorios de asesoramiento para llamadas de herramienta repetidas | escucha eventos de herramienta y de agent |
| [`timeout-policy/`](timeout-policy/README.es.md) | Arma plazos de herramienta por llamada como política de despliegue | registra un listener de `tools/execute` |

Los recordatorios viajan como `additionalContexts` en la decisión de `tools/post-execute` y se añaden como eventos `user/message` registrados con origen en el plugin ([tools](../../docs/subsystems/tools.es.md)); la división del timeout entre `dsh-timeout`, la terminación de capacidades y esta capa de política se registra en la [Agent Note de la librería de timeout](../../.agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.es.md).
