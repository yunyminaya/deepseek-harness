# @deepseek-ai/dsh-client-test-runtime

[English](README.md) | Español

Runtime de pruebas de slot jsdom para specs de funcionalidad de cliente: un `Context` real de Cordis, el `SlotRegistry` de producción y el renderizador de UI, ensamblados alrededor de dobles tipados de sesión/workspace. Las suites de funcionalidad ejercitan la declaración, el registro, el ámbito, el store, la inyección, el renderizado, las actualizaciones y la disposición sin construir la maquinaria a mano por suite — y sin una segunda implementación de ninguna lógica de producción.

Los dobles implementan las mismas caras externas que las funcionalidades reciben a través de ctx (`TestSessions implements ISessions`, `TestWorkspaces implements IWorkspaces`; cada sesión de fixture es un `FixtureSession implements SessionFace`; `stubSettingsScope` es un `SettingsScope` con publicaciones dirigidas por tests y un espía de escritura), así que un cambio en una cara de producción rompe el banco de pruebas en tiempo de compilación en lugar de desviarse en silencio. La materialización del provide-bundle ejecuta el `SessionProvideChannel` de producción — la única implementación compartida con `SessionRuntime`. Los fixtures alimentan datos planos: filas de lista, instantáneas de conversación (parcheadas con immer vía `updateSnapshot`), valores de proyección y stubs de comportamiento tipados como `ISession` que fallan de forma ruidosa cuando una spec llama a un verbo sin stub. El `provide()` tipado restringe los fakes de los nombres de servicio declarados al `Partial` de la cara externa de ese servicio.

Instantáneas DOM locales: `declare(children)` registra un marco automático cuyos envoltorios `<div data-slot>` por clave son raíces de instantánea; `renderSlot(key, owner)` devuelve la vista local al slot (contenedor, consultas de Testing Library con ámbito, `update(owner)` in situ); un serializador de instantáneas registrado dobla los hashes de clase de los módulos CSS (`_frame_a1b2c3` → `frame`) para mantener los archivos `.snap` estructurales y colapsa los internos de `<svg>` a una huella `data-content`. Las suites que necesitan un marco de página personalizado usan `root.declare(children, Frame)` en su lugar; `mount(plugin)` ejecuta un fiber real con comprobaciones previas de servicio que fallan de forma ruidosa, y `dispose()` desmonta las vistas, los fibers de funcionalidad, los ámbitos acuñados y el estado persistido del store en un solo eje.

No forma parte del grafo de plugins de producto (sin `dsh.client`); los paquetes de funcionalidad dependen de él solo en `devDependencies`.

## Model Experience

Ninguna, ya que este paquete es infraestructura de pruebas del lado del navegador; nada de aquí llega a una petición de modelo.

#### Efecto de caché KV

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo diferido

- **Solo se consume a través de alias de fuente del repositorio.** Las specs resuelven el paquete mediante `paths` de tsconfig a `src`; el artefacto `lib/` compilado reexporta `@deepseek-ai/dsh-client-runtime/client`, cuyo bundle es un script de carga de navegador sin exports ESM de Node, así que `lib/index.js` no es importable bajo Node plano. Cada consumidor es una suite de Vitest dentro del repositorio; no hay un entry de runtime compatible con Node.
- **Las instantáneas de conversación son datos de fixture, no historial reproducido.** `updateSnapshot` escribe el store de instantáneas directamente; el cálculo de línea a instantánea sigue cubierto por los tests propios del paquete de runtime y por el e2e de replay. Un fixture puede por tanto expresar estados que la proyección de producción nunca produciría.
