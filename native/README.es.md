# native/

[English](README.md) | Español

Código nativo y paquetes públicos mantenidos con DeepSeek Harness. El [workspace `landlock-run/`](landlock-run/README.es.md) es dueño del lanzador de Landlock que se autolimita y luego ejecuta (self-restrict-then-exec) que consume el harness, incluidos su arquitectura, su familia npm de tres paquetes, el soporte de plataformas, el flujo de trabajo de desarrollo y el [procedimiento de release](landlock-run/docs/release.md).

## Frontera de workspace y release

`landlock-run/` y sus paquetes pertenecen al workspace y lockfile pnpm raíz del repositorio. Los consumidores del harness usan el paquete de entrada actual del workspace durante el desarrollo y la CI, así que un cambio de contrato del lanzador y su actualización de consumidor pueden aterrizar y probarse juntos.

El workflow `Landlock Run` del repositorio principal compila y prueba cada arquitectura soportada. `Landlock Run Release` ensambla esos artefactos nativos, empaqueta y verifica los tres tarballs npm y luego los publica opcionalmente bajo una sola versión de lanzador. El paquete de entrada conserva los paquetes de plataforma como dependencias opcionales de npm, así que npm sigue instalando solo el paquete que coincide con el sistema operativo y la CPU del usuario.
