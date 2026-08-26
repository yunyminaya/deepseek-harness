# Guía de desarrollo

[English](development.md) | Español

El tutorial de configuración lleva a un contribuidor nuevo desde los prerrequisitos hasta un checkout verificado. La referencia para contribuidores que sigue cubre la estructura del repositorio, el flujo de trabajo diario y la organización del CI. La justificación de diseño y los detalles de implementación pertenecen a las Agent Notes y scripts enlazados.

## Tutorial de configuración

### Prerrequisitos

- Node.js admite 22.19+ y 24+. El CI cubre 22.19, 24 y 26; consulta la [Agent Note sobre el mínimo de motor de Node](../.agents/notes/implemented/process/2026-07-06-node-engine-floor.es.md).
- pnpm con Corepack habilitado. El repositorio fija `pnpm@11.7.0` en `package.json`; ejecuta `corepack enable` si `pnpm --version` no se resuelve a través de Corepack.
- Git 2.26 o más reciente; la configuración de hooks habilita la extensión de configuración específica de worktree de Git.
- Opcional: una API key de DeepSeek para los demos de automatización Web, headless y ACP, y para los tests e2e de API real.

### Configuración inicial

Instala las dependencias desde la raíz del repositorio:

```sh
pnpm install
```

La instalación también configura los hooks de Lefthook locales al worktree y el Git merge driver `dsh-translation-pairing` a través de `scripts/install-lefthook.mjs`. La [Agent Note de hooks locales al worktree](../.agents/notes/implemented/process/2026-07-27-worktree-local-lefthook.es.md) es la dueña del contrato de seguridad de las rutas de hooks; la [Agent Note de merges de emparejamiento automático](../.agents/notes/implemented/process/2026-08-08-automatic-translation-pairing-merges.es.md) es la dueña del merge driver.

Si falta alguna de las dos integraciones porque las dependencias se restauraron desde caché o se omitió `postinstall`, instálalas manualmente:

```sh
node scripts/install-lefthook.mjs
```

Si el wrapper rechaza la configuración de Git existente o reporta un lock obsoleto, sigue su diagnóstico y la Agent Note enlazada en lugar de editar los metadatos del worktree de forma especulativa. Después de mover un checkout, vuelve a ejecutar el wrapper para regenerar la ruta de su propiedad.

Ejecuta typecheck una vez tras un clon nuevo:

```sh
pnpm run typecheck
```

La configuración está completa cuando `pnpm run typecheck` termina con éxito.

## Referencia para contribuidores

### Estructura del proyecto TypeScript

El repositorio usa agregados Host y Client aislados. Un paquete ordinario se registra en exactamente un agregado: los paquetes Host en `tsconfig.host.json` y los paquetes Client en `tsconfig.client.json`.

| Archivo | Rol | ¿Forma un programa? |
|---|---|---|
| `tsconfig.json` | Raíz de la solución: `extends` la base, `files: []` y referencias a los dos agregados. Es la entrada de descubrimiento de tsserver y la entrada para ejecutar explícitamente el grafo completo de Project References; a través de los `paths` heredados, también es la config de resolución para tsx al ejecutar `examples/` y `scripts/`. | No |
| `tsconfig.host.json` | Agregado Host: paquetes Host, examples, tests, scripts, website y el proyecto Host excepcional de `api/remotes`. | Sí |
| `tsconfig.client.json` | Agregado Client: los paquetes `packages/client/*` y sus tests, `apps/web` y el proyecto Client excepcional de `api/remotes`. | Sí |
| `tsconfig.base.json` | compilerOptions compartidos y el mapa de `paths` de origen. También es la fachada de resolución a la que las configs de vitest apuntan vite-tsconfig-paths: no tiene `include`, así que sus `paths` se aplican a todo importer. | No |
| `tsconfig.base.client.json` | Ajustes de compilación para navegador (`jsx`, librerías DOM, `types: []`) que extienden el agregado Client y todos los paquetes `packages/client/*`. | No |

Host y Client siguen siendo dos programas agregados porque ambos lados hacen declaration merge de la interfaz `Context` de cordis bajo las mismas claves con servicios distintos; un programa que vea ambos merges reporta una colisión. La colisión solo existe dentro de un `ts.Program` —la resolución de módulos nunca la dispara—, por eso la solución puede referenciar ambos agregados y una misma fachada de paths puede abarcar ambos lados. De aquí se siguen tres disciplinas:

- `tsconfig.base.json` nunca gana `include` ni `files`: se filtrarían a todos los proyectos de paquetes que lo extienden y reducirían el alcance de coincidencia total de la fachada.
- Un script que construye un `ts.Program` de todo el repositorio siembra `tsconfig.host.json` o `tsconfig.client.json` explícitamente —nunca la solución raíz, porque aplanar ambos agregados en un solo programa hace colisionar los merges de `Context`.
- Un paquete nuevo se registra en exactamente un agregado. Tener una entrada de loader de Node y una entrada de navegador no es motivo para dividir un paquete; un plugin Client ordinario produce ambos artefactos de runtime durante la fase de build de Client.

`api/remotes` es el único paquete del repositorio con tsconfigs Host y Client divididos. Su entrada Host debe participar en el grafo Typert de Host, mientras que su entrada Client importa declaraciones `/remote` que el tsdown de Host debe generar primero. Por eso el `tsconfig.json` de la raíz del paquete es solo una solución, y los dos agregados y los consumidores directos referencian `tsconfig.host.json` o `tsconfig.client.json` respectivamente. El gate de `constraints` del workspace recorre el grafo alcanzable de Project References y comprueba la cara de compilación propia de cada proyecto que referencia: un destino de una sola config sigue siendo válido desde cualquiera de las dos caras, mientras que un destino dividido debe nombrar la hoja correspondiente en lugar de su raíz de solución o la hoja opuesta; descubre los paquetes divididos por la presencia de ambas configs hoja, así que una división nueva se une al gate automáticamente. No copies esta estructura a otros paquetes; el [README de `api-remotes`](../packages/api/remotes/README.es.md) explica la división Host/Client y el orden de build.

El build de la raíz sigue el orden de dependencias generado:

```sh
tsc -b tsconfig.host.json
tsdown --env.DSH_BUILD_FACE host
tsc -b tsconfig.client.json
tsdown --env.DSH_BUILD_FACE client
pnpm run build:web
```

Ambas pasadas de tsdown usan el mismo match completo del workspace. No escanean los artefactos de build para descubrir paquetes Client ni mantienen una lista de filtro de paquetes Host/Client. Las configs de tsdown locales a cada paquete seleccionan las entradas de la fase actual a través de `DSH_BUILD_FACE`: un plugin Client ordinario produce tanto su loader de Node como su bundle de navegador durante la fase Client; `api-remotes` usa `hostPhase: true` para producir su entrada Host antes y solo su bundle de navegador durante la fase Client. Tsdown consume únicamente el JavaScript emitido a `lib/types` por la fase tsc anterior.

Typert solo se ejecuta durante el tsdown de Host, sembrado por `tsconfig.host.json`. Analiza los tipos de Host y genera tanto los artefactos de reflexión de Host como la proyección Remote de Host-para-Client; el tsdown de Client no inicia Typert. En consecuencia, `pnpm run typecheck` ejecuta la fase lib completa de Host antes del tsc de Client, mientras que `pnpm run build` continúa por el tsdown de Client y el build de Web. La [nota de build de contrato generado de API Remotes](../.agents/notes/implemented/process/2026-08-08-api-remotes-generated-contract-build.es.md) registra esta decisión de orden.

`pnpm run build` incrusta el entorno `DSH_CLIENT_*` exacto de quien lo llama y no usa valores de client públicos cuando no hay ninguno establecido. `pnpm run build:official` es el equivalente local multiplataforma del build de artefactos del CI y de release. Cada build completo con éxito escribe un registro en gitignore que vincula esos valores a la salida de Vite y a los bundles de client dinámicos; el empaquetado de release y los tests de Web compilados rechazan un registro faltante o artefactos cambiados por un build parcial posterior.

El análisis estático y los tests resuelven los imports del workspace a través del mapa de `paths` base hacia `src` y deben pasar sobre un árbol limpio; los gates que consumen la salida de `lib/` compilada declaran esa dependencia explícitamente. Las declaraciones Remote generadas de Host-para-Client son la excepción deliberada: los comandos públicos `typecheck`, `lint` y `doc-typecheck` las generan primero, mientras que los scripts internos `*:contracts-ready` asumen que un comando público invocador o un gate de scheduler ya depende de la pasada de generación de contratos de Typert o del build completo. Consulta la [nota de la raíz de solución](../.agents/notes/implemented/process/2026-07-22-tsconfig-solution-root-two-aggregates.es.md) para la configuración de dos agregados, la [nota de ts-build-config](../.agents/notes/implemented/process/2026-06-17-ts-build-config.es.md) para la propiedad del emit tsc-primero y la [nota de Typert Remote](../.agents/notes/implemented/architecture/2026-08-02-typert-remote-method-calls.es.md) para el contrato de preparación de gates.

Los servicios de negocio declaran métodos invocables en el Host con `@Remote` o `@RemoteScope`; el build de Host genera los tipos Host-para-Client y las contribuciones de runtime, y la composición `api-remotes` del Client carga esas contribuciones bajo los namespaces `ctx.remote` y `agentCtx.remote` con alcance. Consulta [API Gateway](api-gateway.es.md) para los artefactos generados en ambos lados, sus relaciones de ensamblaje, el fallback de desarrollo SRC y el orden del build de Web.

Si alguna comprobación local relevante consume la salida compilada de un paquete, compila una vez primero:

```sh
pnpm run build
```

`pnpm run hygiene` incluye `publint`, que valida los entrypoints de los paquetes contra los archivos `lib/*.js` compilados, y `verify-node-next-types`, que valida las declaraciones compiladas contra un consumidor NodeNext temporal. Un worktree nuevo no tiene JS empaquetado ni declaraciones hasta que se ejecuta `pnpm run build`; los commits y pushes ordinarios no requieren ese build salvo que sus comprobaciones seleccionadas lo consuman.

### Variables de entorno

El adaptador real de DeepSeek y los demos de agent (agente) respaldados por clave leen las credenciales del entorno o de un `.env` en gitignore en la raíz del repositorio:

```sh
DEEPSEEK_API_KEY=sk-...
DEEPSEEK_BASE_URL=https://... # optional
```

`DEEPSEEK_BASE_URL` es opcional y usa por defecto la API pública. Nunca hagas commit de credenciales reales. Las suites e2e de API real se auto-omiten cuando `DEEPSEEK_API_KEY` no está establecida.

### Integraciones de Git

El merge driver de emparejamiento deriva un registro `.i18n.yaml` en conflicto a partir de los blobs de propietario confirmados (ancestro, actual y otro) cuando ambos archivos de idioma usan la estrategia de texto por defecto de Git y hacen merge limpio. Falla en modo cerrado ante conflictos de propietario, configuración de merge no textual o registros inválidos; después de un merge ya detenido, ejecuta `pnpm run resolve-translation-pairing-conflicts`, que prepara (stages) cada registro de emparejamiento seguro y termina sin éxito si otros conflictos de emparejamiento aún necesitan trabajo manual. Consulta el [contrato de documentación bilingüe](i18n/README.es.md#the-pairing-contract) para los archivos y estados exactos que acepta el driver.

El instalador prueba el entrypoint exacto del driver Node/tsx antes de publicar su configuración de worktree. Si ese runtime deja de estar disponible más tarde, el launcher independiente de Node escribe el resultado de texto ordinario de Git, deja el sidecar sin resolver e imprime la ruta de recuperación; restaura las dependencias y ejecuta `pnpm run resolve-translation-pairing-conflicts`, o ejecuta `git merge --abort`. Si `pre-merge-commit` rechaza un merge que por lo demás está limpio, Git deja el resultado completo preparado (staged) sin commit; repara el fallo y ejecuta `git commit`, o aborta. La [Agent Note de merges de emparejamiento automático](../.agents/notes/implemented/process/2026-08-08-automatic-translation-pairing-merges.es.md#failure-contract) es la dueña de los estados exactos del index y de `MERGE_HEAD`.

lefthook está configurado en `lefthook.yml` como un checkpoint local rápido:

- `pre-commit` verifica los registros de emparejamiento preparados contra los blobs de propietario preparados, valida los archivos preparados con el perfil `.oxlintrc.staged.json` libre de proyecto y aplica las correcciones de Oxlint con un reintento acotado, regenera `THIRD_PARTY_NOTICES.md` cuando un archivo preparado es una de sus entradas, comprueba el diff preparado en busca de errores de espacios en blanco y ejecuta el guard del manifest (manifiesto) del vendor.
- `pre-merge-commit` realiza la misma comprobación de emparejamiento respaldada por el index antes de que Git cree un commit de merge automático.
- `pre-push` ejecuta `pnpm run typecheck`, que completa la fase lib de Host, incluidos los contratos de Typert generados, antes de la comprobación de TypeScript del Client.

El guard del manifest del vendor comprueba que los cambios bajo `vendor/*/src` se preparen junto con la actualización del manifest `vendor/README.md` correspondiente. Consulta `vendor/README.md` antes de editar código vendored.

Aparte de la verificación acotada de registros preparados, los hooks no ejecutan a propósito tests, instantáneas, comprobaciones de documentación, builds ni hygiene. Los contribuidores ejecutan una vez las [comprobaciones relevantes para el comportamiento cambiado](../AGENTS.md#run-relevant-checks-locally); el CI es dueño de la cobertura exhaustiva, de los smokes de artefactos compilados y de la matriz de compatibilidad de Node 22.19, 24 y 26.

Los contribuidores pueden optar por el conjunto completo de gates locales con `pnpm run check:all`. El comando es independiente de los hooks de Git y no es una instrucción de agent.

### Gates del CI

El [flujo de trabajo de CI](../.github/workflows/ci.yml) sin clave agrupa los gates independientes en lanes amplios y ejecuta una señal de compatibilidad más pequeña en las versiones de Node soportadas. Los consumidores de artefactos esperan un build dentro de su lane. El flujo de trabajo separado de API real ejecuta `pnpm run test:e2e` con su worker configurado vinculado. Consulta [scripts/run-gates.ts](../scripts/run-gates.ts) y los archivos de flujos de trabajo para el inventario actual de gates y jobs.

### Comandos diarios

Las [instrucciones para contribuidores](../AGENTS.md#commands) de la raíz resumen los comandos comunes, mientras que [`package.json`](../package.json) y [scripts/run-gates.ts](../scripts/run-gates.ts) son los dueños de los inventarios actuales de scripts y gates. Selecciona las comprobaciones más pequeñas que cubran la superficie cambiada. Los cambios de documentación usan `pnpm run doc-sync`; los cambios de comportamiento público de paquetes también actualizan el README o JSDoc de su propietario, y las comprobaciones de artefactos compilados requieren `pnpm run build` primero.

### Demos

Ejecuta el build del repositorio por separado antes de usar estos demos de checkout desde el código fuente:

```sh
pnpm run build
```

El coding agent Headless de un solo uso necesita `DEEPSEEK_API_KEY` en el entorno o en el `.env` de la raíz del repositorio:

```sh
pnpm dsh --profile headless "summarize this workspace"
```

El demo cordis autorreferencial puede inspeccionar y modificar su runtime de plugins en vivo y necesita las mismas credenciales (`web` por defecto, o `acp`):

```sh
pnpm run demo:cordis
```

El servidor de automatización ACP expone sesiones de agent nuevas a través de JSON-RPC stdio y también necesita `DEEPSEEK_API_KEY`:

```sh
pnpm run demo:acp
```

### Marcadores TODO

Usa una de las tres etiquetas de comentario para marcar problemas conocidos en el código, ordenadas por urgencia:

- `FIXME` — un problema que debería bloquear un release nuevo. Un release no debería publicarse con un `FIXME` abierto salvo que los revisores acepten explícitamente que el cambio puede mergearse de todos modos.
- `TODO` — un problema que debería arreglarse pronto, cuando tengamos los recursos.
- `XXX` — un problema que quizá arreglemos algún día; la prioridad más baja, sin compromiso.

Elige la etiqueta que coincida con la urgencia para que cualquiera que escanee el código distinga un bloqueador de release de un quizá-algún-día.

### Documentar tipos tal cual (`ts type-equiv`)

Las páginas de [subsistemas](subsystems/README.es.md) pegan declaraciones equivalentes al código fuente junto con su JSDoc original para que el lector vea la definición exacta del tipo y el contrato de la fuente. Para evitar que una pegada se desvíe cuando cambia el código fuente, delimítala como ` ```ts type-equiv ` (en lugar de ` ```ts `) y regístrala en `scripts/type-equiv.manifest.json` con el archivo fuente y el símbolo que refleja:

```json
{ "doc": "docs/subsystems/session.md", "symbol": "SessionEvent", "source": "packages/core/session/src/types.ts" }
```

`pnpm run verify-type-equiv` (parte de `doc-sync`) extrae entonces la declaración de ese símbolo y su JSDoc adjunto del código fuente a través del parser de TypeScript y comprueba que el bloque coincide con ambos. Para una clase cuyos cuerpos de implementación no pertenecen al catálogo, usa ` ```ts public-api ` y establece `"projection": "public-api"`; la proyección comprobada conserva los campos públicos, el constructor, los accessors, los métodos y el JSDoc original de la clase y sus miembros, omitiendo los cuerpos y los miembros privados o protegidos. La comparación ignora los espacios en blanco y los comentarios que no son JSDoc, pero exige todos los comentarios JSDoc originales, incluida la documentación de los miembros, para que los lectores vean el contrato de la fuente junto a la definición exacta del tipo. El gate hace cumplir una correspondencia 1:1 por documento, símbolo y proyección entre los bloques primarios y las entradas del manifest; un bloque `.zh.md` emparejado reutiliza la entrada de su hermano sin sufijo solo cuando toda la secuencia de fences rastreada es idéntica byte a byte y está ordenada de forma idéntica. `doc-typecheck` aplica la misma regla derivada a los fences compilables, saltándose ambos tipos de fences de equivalencia con la fuente en la compilación y en su ratio de exclusión. Cuando cambias una declaración documentada o su JSDoc, el gate falla hasta que actualices la pegada; cuando añades o eliminas un bloque primario, actualiza el manifest en el mismo cambio.
