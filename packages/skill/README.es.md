# skill/ — familia de capacidades de skill

[English](README.md) | Español

Esta familia descubre instrucciones de agent (agente) reutilizables y las expone al modelo a través de un catálogo y un loader neutrales respecto al provider.

| Paquete | Rol | Clave de ctx |
|---|---|---|
| [`skill/`](skill/README.es.md) | Define el registro y la búsqueda de providers de skill | `ctx.skills` |
| [`skill-badge/`](skill-badge/README.es.md) | Aporta el skill de insignia dsh empaquetado opcional | se registra en `ctx.skills` |
| [`skill-filesystem/`](skill-filesystem/README.es.md) | Descubre skills en sistemas de archivos locales | se registra en `ctx.skills` |
| [`tool-skill/`](tool-skill/README.es.md) | Publica el catálogo de skills y el loader orientado al modelo | se registra en `ctx.tools` |

Esta capacidad permanece fuera del tronco de control del núcleo y puede usar providers locales, integrados o remotos sin cambiar el contrato orientado al modelo.

La referencia del subsistema — prioridad de descubrimiento, instantáneas del catálogo, el loader `skill` — está en [docs/subsystems/skills.md](../../docs/subsystems/skills.es.md).
