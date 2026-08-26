# AGENTS.md — Paquetes web

[English](AGENTS.md) | Español

Estas reglas complementan las convenciones de paquetes de [packages/AGENTS.md](../AGENTS.es.md).

- **Rechaza las redirecciones en las solicitudes de provider que llevan credenciales.** Configura el cliente HTTP para que falle antes de seguir cualquier respuesta de redirección. La cobertura de regresión debe demostrar que no se contacta con el destino de la redirección y que todos los providers con credenciales optan por la política. El endpoint configurado recibe necesariamente la solicitud inicial; esto evita el reenvío automático de credenciales o datos de solicitud a otro origen, no el compromiso del endpoint configurado.
