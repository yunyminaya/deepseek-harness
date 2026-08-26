# Matriz de soporte

[English](support-matrix.md) | Español

## Soportado

| Paquete de plataforma | Runner de GitHub (builder of record) | Notas |
|---|---|---|
| `@deepseek-ai/node-addon-landlock-run-linux-x64` | `ubuntu-24.04` | musl estático — distribuciones glibc y musl por igual |
| `@deepseek-ai/node-addon-landlock-run-linux-arm64` | `ubuntu-24.04-arm` | musl estático — distribuciones glibc y musl por igual |

La aplicación además requiere un kernel con Landlock habilitado (5.13+). El nivel de ABI negociado decide el veredicto de la sonda: todo acceso que esta compilación sabe amparado → `full`; un ABI más antiguo que ampara un subconjunto → `partial` (aún confinado para todo lo que soporta); Landlock ausente o deshabilitado → `unusable`, y el lanzador se niega a ejecutar comandos por completo. La sonda — no la versión del kernel — es la autoridad: un kernel compilado sin Landlock, o con el LSM deshabilitado, devuelve `unusable` en la sonda independientemente de su versión.

## Deliberadamente no soportado

- **darwin**: los consumidores de macOS suelen confinar a través de `sandbox-exec`/Seatbelt, que viene con el SO — no hay binario que distribuir.
- **win32**: un lanzador de confinamiento para Windows sería un mecanismo distinto en su propio repositorio, no un port de este.
- **Otras arquitecturas Linux** (riscv64, s390x, …): todavía no hay un builder of record nativo en CI. La regla de no usar toolchains cruzadas significa que un paquete de plataforma solo se añade junto con un runner nativo que lo compile y lo pruebe.

Un consumidor en una plataforma no soportada resuelve una ruta de lanzador inexistente, devuelve `unusable` en la sonda y cierra ante los fallos — la degradación documentada, ejercitada por el tramo darwin de CI.
