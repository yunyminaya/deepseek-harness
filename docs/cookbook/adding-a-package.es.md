# Recetario: añadir un paquete de workspace

[English](adding-a-package.md) | Español

La lista de verificación archivo por archivo para un nuevo paquete `@deepseek-ai/dsh-<name>`. Esta lista está validada contra los paquetes bash y adapter como plantillas; si se desvía de ellos, corrígela aquí.

## 1. Crea el paquete

```
packages/<group>/<pkg>/
  package.json     # copy from packages/core/tools, adjust name/description/deps
  tsconfig.json    # extends ../../../tsconfig.base.json, rootDir src,
                   # outDir lib/types, references: ../../../vendor/cosmokit,
                   # ../../../vendor/cordis (+ ../../../vendor/schemastery if
                   # you use Config, + ../../<group>/<dep> for each dsh dep)
  src/index.ts     # service default export or plugin (name/inject/apply/Config)
  README.md        # service API, events, extension points, design notes,
                   # + gated Model Experience context blocks or short form
                   # + the gated "Known Limitations and Deferred Work" section
                   # (or a whitelist entry in scripts/verify-package-readme-limitations.ts)
```

Elige un grupo existente cuando uno coincida con el rol del paquete (`core`, `llm`, `bash`, `compact`, `subagent`, `todo`, `session-persistence`, `ui`, `util` o `support`). Un grupo nuevo está permitido, pero es un contenedor puro: sin `package.json`, sin archivos fuente, y los paquetes siguen estando exactamente un nivel por debajo.

Invariantes de package.json (aplicados por `pnpm run constraints` / `scripts/check-workspace-constraints.ts`): `private: true`, un `version` que coincida con el `package.json` raíz, `type: module`, `main: "lib/index.js"`, `types: "lib/types/index.d.ts"`, `exports["."].types: "./lib/types/index.d.ts"`, `exports["."].default: "./lib/index.js"`, `@deepseek-ai/cordis` en TANTO peerDependencies como devDependencies (mismo rango). Refleja cada dependencia peer de dsh en devDependencies. `@deepseek-ai/schemastery` va en `dependencies` (es un validador en tiempo de ejecución), igual que agent-loop. La lista `files` contiene exactamente `lib/index.js`, `lib/invariant.js`, `lib/types/**/*.d.ts` y los artefactos de runtime específicos del paquete que reconoce el gate; un paquete cuya exportación de runtime apunta al árbol emitido también incluye `lib/types/**/*.js`. No publiques `src`, mapas de declaraciones, mapas JS ni archivos de declaración raíz obsoletos. Los paquetes de app CLI con un `bin` de paquete incluyen `lib/bin.js` inmediatamente después de `lib/index.js` en `files`.

Los imports relativos dentro del paquete usan especificadores `.ts` explícitos en el código fuente (por ejemplo, `export * from './types.ts'`). El compilador los reescribe a `.js` en el JS emitido y deja los especificadores `.ts` explícitos en las declaraciones, que los consumidores TypeScript estándar de NodeNext/Node16 resuelven a los archivos `.d.ts` hermanos.

## 2. Regístralo en las configs raíz

| Archivo | Cambio |
|---|---|
| `tsconfig.base.json` | sin edición para un grupo existente; para un grupo nuevo, añade un candidato `./packages/<group>/*/src` al comodín `@deepseek-ai/dsh-*` |
| `tsconfig.host.json` (paquete Host) o `tsconfig.client.json` (paquete Client) | añade `{ "path": "./packages/<group>/<pkg>" }` a `references` — un paquete normal pertenece exactamente a un aggregate, nunca a ambos. `api/remotes` usa una división específica del repositorio porque el Host genera un contrato que el Client consume en una fase posterior; los paquetes nuevos no deben copiarla ([layout](../development.es.md#typescript-project-layout)) |
| `knip.json` | solo si el paquete tiene entrypoints que el descubrimiento del repositorio no cubre ya |

Un paquete `packages/client/*` extiende además `tsconfig.base.client.json` en lugar de `tsconfig.base.json`, y un paquete de plugin de cliente declara `dsh.client` en package.json, exporta `./client` y llama al preset compartido de tsdown (`packages/client/tsdown.client.ts`) — consulta [packages/client/AGENTS.md](../../packages/client/AGENTS.md) para el contrato del lado cliente.

Cubierto automáticamente por globs o por el descubrimiento de manifests de paquetes — sin ediciones necesarias: workspaces del `package.json` raíz, `scripts/publint-all.ts`, `tsdown.config.ts`, `.oxlintrc.json`, `scripts/check-workspace-constraints.ts`.

## 3. Decide la topología del paquete

Para una capacidad intercambiable, separa los roles de Service Definition / Service Provider / Consumer en paquetes cuando evolucionan de forma independiente (consulta docs/architecture.md § «Capability seams» — el trío de shell es la plantilla). Un plugin de un solo propósito sigue siendo un único paquete.

### Nombra el rol que existe

Nombra la responsabilidad estable actual. No nombres la primera implementación, una posible expansión futura ni la clase base de Cordis. Un paquete de interfaz nombra la capacidad. Un paquete de implementación añade el mecanismo, protocolo, entorno o vendor que lo distingue. Usa `local` solo cuando la ejecución en el mismo host sea parte del contrato.

Usa una clave `ctx` en singular para un único engine, runtime, policy, controller, resolver, store o configuración actual. Usa una clave en plural para un registro o un servicio que posee varios miembros con nombre. El rol de la clase y el número de la clave deben concordar. No reutilices una clave de `Context` de Cordis para declaraciones host y client incompatibles. El declaration merging de TypeScript ve ambas caras aunque usen contextos de runtime separados. Añade el sufijo del rol cuando el plural natural ya pertenezca a otra cara.

| Palabra | Úsala cuando | No la uses cuando |
|---|---|---|
| `Controller` | Acepta comandos o intención del usuario y cambia un estado de dominio o de presentación existente. | Ejecuta trabajo arbitrario, posee una flota de providers o solo convierte valores para mostrarlos. |
| `Store` | Posee un único conjunto de datos y ofrece sobre todo operaciones de CRUD, instantánea o suscripción para esos datos. | Valida una máquina de estados, arbitra autoridad, despacha trabajo o posee precedencia de providers. Un map no convierte a una clase en store. |
| `Directory` | Expone entradas y metadatos para descubrimiento o selección. | Los producers registran implementaciones arbitrarias en él, o los llamadores ejecutan trabajo a través de él. |
| `Presenter` | Es una conversión pura de valores de dominio o argumentos de herramienta a intención de renderizado. | Realiza I/O, se suscribe, muta estado o posee ciclo de vida. |
| `Registry` | Posee un conjunto dinámico de registros con nombre, incluidos lookup, reglas de duplicados o precedencia, tiempo de vida y disposición. | Su contrato principal es dispatch, ejecución, cancelación, policy u orquestación. |
| `Runtime` | Ejecuta trabajo en vivo y posee dispatch, cancelación, coordinación de providers o ciclo de vida de operaciones entre llamadas. | Solo almacena registros, devuelve un catálogo, resuelve un valor o guarda configuración. |
| `Resolver` | Calcula o localiza una única respuesta a partir de las entradas suministradas sin poseer el ciclo de vida de esa respuesta. | Posee una colección mutable o una ejecución de larga duración. |
| `Binder` | Adjunta una interfaz declarada a un contexto o ciclo de vida del llamador y devuelve el valor vinculado. | Posee el valor como colección, controla su estado de dominio o solo convierte datos. |
| `Engine` | Implementa un algoritmo de dominio o un modelo de ejecución con estado. | Solo selecciona un provider o reenvía a través de un límite de protocolo. |
| `Policy` | Decide qué está permitido, seleccionado, limitado u observado. | Ejecuta el mecanismo que la decisión permite. |
| `Executor` | Ejecuta una solicitud explícita o una especificación resuelta en una capacidad. | Posee un ciclo de vida de aplicación amplio o un catálogo de providers. |
| `Gateway` | Adapta un límite de proceso, red, RPC o API. | Solo registra servicios del mismo proceso o almacena metadatos. |
| `Provider` | Aporta una implementación de una definición de capacidad. Añade un calificador de mecanismo o vendor cuando puedan existir varias. | Es la definición de capacidad, el registro de providers o el runtime del consumidor. |
| `Backend` | Implementa persistencia, transporte o ejecución de nivel inferior reemplazables detrás de una interfaz definida. | Es un servicio de cara al usuario o una referencia a recurso en vivo devuelta. |
| `Handle` | Se refiere a un recurso en vivo y controla u observa ese recurso. | Crea y gestiona el conjunto completo de recursos. |
| `Config` | Posee un valor de configuración resuelto o un registro acotado de forma estricta y su contrato de actualización. | Almacena una colección general, ejecuta trabajo o expone ajustes no relacionados. |
| `Service` | Posee un servicio de dominio cohesionado que ningún rol más preciso de los anteriores describe con honestidad. | El nombre existe solo porque la clase extiende el `Service` de Cordis. |

Usa `SDK` solo para el protocolo cliente/servidor JSON-RPC que usan los SDK de Python y TypeScript soportados. DeepSeek Harness en sí es un agent harness, no un proyecto SDK. Usa la ortografía canónica del producto, `Typert`, nunca `TypeRT` ni `typeRT`.

## 4. Escribe el README del paquete

Mantén primero la API de servicio, la config, los eventos, los puntos de extensión y las notas de diseño específicos del paquete. La sección de limitaciones registra los huecos durables del consumidor y las restricciones no obvias del mantenedor que posee este paquete; la limpieza ordinaria queda en su TODO de código o en una Agent Note. Una frase indirecta de Model Experience puede nombrar al consumidor que saca a la luz la contribución de este paquete, pero no reexpone la implementación de ese consumidor. Termina el README del paquete con esta secuencia canónica:

````markdown
## Model Experience

### Request context and condition

#### What the model sees

The exact data-dependent fields, an anchored generated-catalog link, or an introduction to the verbatim literal below.

##### Verbatim text for this field, when needed

```markdown
Stable system-prompt prose of any length, or another long non-generated literal, copied exactly from source.
```

#### Token effect

Fixed, conditional, retained, replaced, capped, or zero-direct token effect.

#### KV Cache effect

Append-only, prefix-stable, replacing, or independent behavior, including the exact conditions that may invalidate reuse.

## Known Limitations and Deferred Work

- **Consumer-visible gap** — exact missing operation or case, its consequence, and any maintainer constraint.
````

Rellena Model Experience a partir de la implementación. Usa un H3 por entrada de contexto de modelo directa, condicional, con tope, de ciclo de vida o auxiliar, con los tres campos H4 ordenados que se muestran arriba y un párrafo en prosa bajo cada uno. Cita el texto estable que posee el paquete: la prosa del system prompt va en un H5 con título más un fence `markdown` bajo el campo que la introduce — normalmente `What the model sees` —, otros literales cortos se quedan en línea con placeholders con nombre, y otros literales largos usan la misma forma anidada. Resume solo el texto dependiente de datos o de propiedad del provider. Una entrada de tool schema enlaza su sección anclada en el [catálogo de herramientas](../tool-catalog.es.md) y solo indica los deltas ausentes allí. Mantén separadas las entradas de prompt y de schema cuando el ámbito pueda ocultar una sin la otra. En `KV Cache effect`, distingue el crecimiento solo de append, un prefijo repetido estable, el reemplazo de tokens de solicitud anteriores y una solicitud de modelo independiente, y luego nombra los cambios de propiedad del paquete que pueden invalidar la reutilización. «Does not invalidate» significa que el paquete conserva un prefijo ya reutilizable; la disponibilidad de la caché del provider y su expulsión quedan fuera del contrato del paquete. El [estándar de prosa](../../.agents/skills/dsh-prose-standard/SKILL.md) rige la completitud y la propiedad; el verificador aplica la estructura de secciones requerida.

Un paquete sin efecto de contexto, o con una única ruta de propiedad del consumidor, usa la frase auditada `None, as ` o `Indirectly, through ` de [`SENTENCE_MODEL_EXPERIENCE`](../../scripts/verify-package-readme-model-experience.ts), seguida de un H4 `KV Cache effect` y un párrafo no vacío; un paquete genérico independiente del modelo puede en su lugar sumarse a `NO_MODEL_EXPERIENCE_SECTION`. No expandas ninguno de los dos casos a una descripción del trabajo de otro paquete. La [allowlist](../../scripts/verify-package-readme-limitations.ts) de limitaciones es independiente. La [Agent Note de Model Experience](../../.agents/notes/implemented/process/2026-07-12-package-model-experience-contract.es.md) registra el razonamiento.

## 5. Verifica

```sh
pnpm install        # registers the workspace
pnpm run doc-sync
pnpm run constraints && pnpm run typecheck && pnpm run lint
pnpm run build && pnpm run hygiene
```

Sigue la [política de pruebas del repositorio](../testing.es.md) para los checks específicos de comportamiento y la cobertura que requiere el nuevo paquete.
