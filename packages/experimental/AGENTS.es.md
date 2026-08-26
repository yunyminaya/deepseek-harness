# AGENTS.md — Paquetes experimentales

[English](AGENTS.md) | Español

Estas reglas complementan las [reglas de paquetes](../AGENTS.md). La [decisión de los paquetes experimentales de Agent Teams](../../.agents/notes/implemented/architecture/2026-08-18-experimental-agent-teams-packages.md) es dueña de la justificación.

- Un paquete pertenece aquí solo cuando todo su contrato público es experimental o solo interno. Una opción experimental dentro de un paquete de release permanece con su rol de producto propietario.
- Cada paquete de aquí usa el prefijo npm `@deepseek-ai/dsh-experimental-*`, define `private: true` y omite `publishConfig`; la compuerta de restricciones del workspace impone estas declaraciones y la familia de releases de dsh excluye este directorio.
- Los paquetes y aplicaciones de release no deben nombrar paquetes de aquí en `dependencies`, `optionalDependencies` ni `peerDependencies`. Los paquetes experimentales pueden depender de paquetes de release y entre sí. Los tests pueden usar paquetes experimentales a través de `devDependencies`; los ejemplos pueden cargarlos explícitamente.
- El estado experimental no relaja los requisitos de ingeniería, seguridad, documentación, ciclo de vida, tests, invariants ni instantáneas.
- La promoción mueve un paquete a su grupo de rol de producto y elimina `experimental-` de su nombre npm. Actualiza cada import y cada fila de configuración de forma atómica, y luego revisa su contrato público, sus limitaciones, la evidencia de tests, el payload de release, los dependientes de runtime y el propietario estable nombrado.
