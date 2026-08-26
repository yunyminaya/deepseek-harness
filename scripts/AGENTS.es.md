# AGENTS.md — Scripts del repositorio

[English](AGENTS.md) | Español

Los scripts de gate invocan pnpm sin shell, normalizan las rutas glob relativas al repositorio a `/` al ingerirlas y mantienen la adaptación de plataforma en el gate que la necesita, en lugar de en una capa de plataforma compartida.
