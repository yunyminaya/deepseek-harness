# @deepseek-ai/dsh-code-runtime

[English](README.md) | Español

La **`CodeRuntime`** (`ctx.codeRuntime`) define QUÉ hace un runtime de código — ejecutar un programa escrito por el modelo contra un conjunto de bindings asíncronos proporcionados por el host e informar `{ value, logs, error? }` — sin decir CÓMO.

Este paquete posee el rol de Service Definition de la capacidad (el trío de bash es la plantilla — ver [capability seams](../../../.agents/notes/implemented/architecture/2026-06-13-capability-seams.es.md)): los providers subclasifican `CodeRuntime` y registran el servicio; el Consumer es el Code Mode del registro de herramientas, que genera el SDK orientado al modelo y puentea el despacho de herramientas — ambos especificados en la [Agent Note de Code Mode](../../../.agents/notes/implemented/feature/2026-06-15-code-mode.es.md), cuyo primer provider es un backend de hilo de trabajo de Node. El runtime no sabe nada de herramientas ni de sesiones: recibe funciones asíncronas nombradas y una cadena de programa, y todo lo que tiene forma de herramienta queda en el Consumer.

## API del servicio (`ctx.codeRuntime`)

| Miembro | Semántica |
|---|---|
| `run(request)` | Ejecuta un programa contra los bindings de la solicitud. **Se resuelve con un campo de error PARA cada resultado del programa** — fallo de parseo/transformación, excepción lanzada, finalización inválida, desbordamiento de salida, caducidad de presupuesto, abort o muerte del sustrato (la taxonomía ortogonal `kind` de `CodeRunFailure`); solo rechaza por mal uso del contrato de Service Definition por parte del llamador (p. ej. una ejecución enviada después de la disposición). El programa se ejecuta como cuerpo de una función asíncrona: el `await`/`return` de nivel superior funcionan, y una finalización JSON sin pérdidas se convierte en `result.value`. |
| `language` | Descriptor de solo lectura: el lenguaje fuente que espera `run`. `'typescript'` y `'python'` son los valores conocidos — los que presenta `dsh-tools`; solo `'typescript'` tiene un backend publicado. Informativo, no restrictivo — un consumer que genera presentación específica de lenguaje conmuta según él y falla de forma ruidosa ante un lenguaje que no puede presentar. |
| `isolation` | Descriptor de solo lectura: el sustrato de ejecución (`'worker-thread'`, `'process'`, `'container'`). Una etiqueta para despliegues y diagnósticos, **no una afirmación de seguridad**. |

Semántica que toda implementación debe respetar (detalles del contrato en el JSDoc de la clase): las llamadas de binding puentean argumentos y resoluciones JSON completos sin pérdidas sin tope de bytes a nivel de seam; el programa se trata como un peer hostil (los nombres arbitrarios de binding son propiedades propias, el tráfico malformado nunca tumba al host); ningún estado sobrevive entre ejecuciones; la disposición termina las ejecuciones en curso Y espera su salida antes de completarse.

## Vocabulario

`CodeRunRequest` (`program`, `bindings`, `signal?`) transporta todo aquello sobre lo que actúa el runtime — el valor predeterminado (presupuestos de tiempo y tope de salida exterior) es la configuración validada del provider, nunca un `??` oculto dentro de `run()`. `bindings` es una lista de `CodeBindingNamespace`s (`global` + `functions` + `errorClass` opcional), cada uno expuesto al programa como un objeto global de callables asíncronos que devuelven `CodeJsonValue`, el equivalente estructural local al servicio del `JsonValue` canónico que mantiene este paquete de Service Definition independiente de las sesiones. Un descriptor `errorClass` nombra un constructor global real del programa y la propiedad propia que recibe el nombre del miembro rechazado; los runtimes permanecen independientes de términos del Consumer como `ToolCallError`. `CodeRunResult` informa de la finalización JSON sin pérdidas `value?`, los `logs: string[]` ordenados y el `error?` (`CodeRunFailure`: `kind` + `message` alimentable al modelo). Ver `src/types.ts` para los contratos completos.

Los nombres de binding-global y de clase de error son **portables entre lenguajes**: deben encajar en el subconjunto de identificadores `[A-Za-z_][A-Za-z0-9_]*` (sin el `$` solo de JS) y superar los conjuntos de exclusión exportados por el seam, de modo que una lista `bindings` sea válida contra cualquier backend sin importar su `language`. El paquete exporta el contrato que todo backend aplica — `PORTABLE_RESERVED_WORDS` (palabras reservadas de ECMAScript ∪ Python), `RESERVED_BINDING_GLOBALS` (globales propiedad del backend como `console`), `RESERVED_ERROR_MEMBERS` y `DUNDER_MEMBER` (exclusiones de miembros de error) — de modo que un nombre como `$tools`, `lambda` o `__dsh_main__` hace que `run()` rechace como mal uso del seam en cualquier backend, no solo en algunos. Ver `src/index.ts` para los conjuntos exactos y la justificación.

## Experiencia del modelo

Indirectamente, a través de Code Mode en `dsh-tools`, que expone `run_code` y devuelve los logs, valores o fallos del programa como tokens retenidos de resultado de herramienta.

#### Efecto en la caché KV

Sin invalidación directa; el Consumer nombrado es dueño de cualquier cambio en el prefijo de solicitud.

## Limitaciones conocidas y trabajo pendiente

- **`run()` es de un solo uso** — los `logs` llegan solo en el `CodeRunResult` resuelto; el seam no expone una API de logs en streaming ni de progreso para la salida de un programa en vivo.
- **Un kernel persistente estilo REPL está registrado como trabajo futuro** — el contrato de sin-estado-entre-ejecuciones se mantiene hasta que un backend de kernel persistente traiga su propia historia de registro ([Agent Note de Code Mode](../../../.agents/notes/implemented/feature/2026-06-15-code-mode.es.md)).
- **Solo se distribuye el backend de hilo de trabajo** — `'process'`/`'container'` se declaran como valores `isolation` conocidos sin implementación; un límite de seguridad duro espera a un backend de contenedor.
- **Los valores de binding intermedios no tienen tope de bytes** — las implementaciones siguen sujetas al coste del clonado estructurado y a la memoria del proceso, mientras que un provider o executor ya puede haber impuesto su propio límite de adquisición.
