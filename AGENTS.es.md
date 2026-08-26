# AGENTS.md

[English](AGENTS.md) | Español

DeepSeek Harness es un agent harness (marco de trabajo para agentes) basado en plugins sobre Cordis vendored: **todo es un plugin**. Lee [docs/architecture.md](docs/architecture.es.md) antes de cambiar `packages/`; sigue [docs/AGENTS.md](docs/AGENTS.es.md) para la documentación.

## Postura previa al lanzamiento: los cimientos antes que el radio de impacto

**Elimina esta sección en el primer release etiquetado.** Sin consumidores externos, prefiere la base correcta antes que los parches de compatibilidad: renombra o reempaqueta libremente y actualiza todas las referencias a la vez. Los backends rechazan los formatos antiguos en disco. SQLite usa una `SCHEMA_VERSION` monótona; `dsh-session` mantiene `SESSION_FORMAT_VERSION` en `0` sin promesa de compatibilidad.

## Estructura del repositorio

```
vendor/      Vendored Cordis source — manifest + sync procedure in vendor/README.md
packages/    @deepseek-ai/dsh-<pkg> workspaces at packages/<group>/<pkg>/
  core/        product API spine: session, system-prompt, tools, agent, agent-loop
  api/         Remote BFF assembly and Typert RPC gateway
  typert/      type graph generator, loader, and runtime registry
  llm/         LLM capability: Service Definition/Consumer + DeepSeek providers
  e2b/         E2B POC: sandbox + FS/subprocess adapters
  shell/        bash capability: Service Definition + local/pwsh providers + shell Consumers
  subprocess/  subprocess capability + local process-tree provider
  terminal/         persistent sessions
  fs/          filesystem capability + policy
  lsp/         language-server capability
  skill/       skill provider registry + local impl + catalog/loader tool
  web/         web capability: Service Definition + search/fetch providers + tool Consumer
  compaction/     compaction capability + basic provider
  context/     request-context plugins
  subagent/    subagent capability: Service Definition + providers + delegation Consumers
  bundle/      installable dsh --profile patch-layer bundles
  workflow/    workflow capability + worker-thread provider + tool Consumer
  todo/        todo_write tool
  plan/        plan mode as logged state
  preset/      per-session agent composition from preset cordis.yml files
  guard/       loop-hygiene + tool-timeout plugins
  self-modification/  the agent inspects/mounts its own plugins
  hooks/       Claude Code/Codex hook bridges + wire-protocol library
  session/     durable session data: persistence, projection, titles, telemetry
  identity/    anonymous identity
  settings/    user-settings capability + file provider
  credentials/ credential/authorization capabilities + env/.env provider
  acp/         automation-only Agent Client Protocol server
  interaction/ approval/interaction capabilities, permission, commands, ask-user
  boot/        shared app-bin glue
  sdk/         JSON-RPC protocol, server, and TypeScript client
  examples/    demo bundles (agent-spine + CLI/ACP/JSON-RPC bins)
  experimental/ private prototypes excluded from official releases
  support/     dev/test infrastructure
  util/        zero-dependency utilities
python/      Python SDK and bundled runtime (see python/README.md)
native/      @deepseek-ai/node-addon-landlock-run source of record (see native/README.md)
examples/    Runnable cordis.yml leaves over packages/examples bundles (see examples/AGENTS.md)
.agents/     Agent workflows and Agent Notes (`notes/`)
docs/        architecture, generated catalogs, postmortems, cookbook (see docs/AGENTS.md)
scripts/     repo gates and generators
website/     VitePress projection of selected bilingual docs/ sources
```

Grupos de paquetes: [packages/README.md](packages/README.es.md).

## Comandos

```sh
pnpm install            # pnpm workspaces, node ^22.19 || >=24
pnpm run clean           # remove build outputs and safe residue from deleted packages
pnpm run test           # vitest unit tests
pnpm run test:coverage  # CI coverage gate: per-file 100% on packages/*/*/src
pnpm run test:e2e       # real-API tests; self-skip without DEEPSEEK_API_KEY
pnpm run test:snapshot  # keyless ACP/headless replay vs expected outputs; filter: -t <name>
pnpm run test:snapshot:record  # re-record expected outputs (needs key)
pnpm run typecheck
pnpm run lint
pnpm run duplication    # cross-file TypeScript clone detection
pnpm run build          # tsc emits lib/types, tsdown bundles runtime
pnpm run hygiene        # knip + publint + workspace constraints + NodeNext consumer check
pnpm run check:windows-wine  # ONLY when diagnosing a known Windows failure (needs wine); CI owns this signal
pnpm run doc-sync       # all documentation gates; leaf list in scripts/run-gates.ts
pnpm run website:build  # VitePress build (doubles as dead-link check)
pnpm dsh --profile headless "task"  # run one task from source (needs DEEPSEEK_API_KEY)
pnpm run demo:cordis    # the agent modifies its own runtime (needs key)
pnpm run demo:acp       # ACP automation server (needs DEEPSEEK_API_KEY)
```

### Fallos del sandbox del host

Cuando los comandos necesarios `gh`, `pnpm`, de build, de test o generadores fallan porque el sandbox del agent bloquea credenciales, red, IPC, la observación de archivos o `sandbox-exec` anidado, reinténtalos sin cambios con la escalada de host más estrecha antes de diagnosticar un fallo de autenticación o del proyecto. Exige evidencia del sandbox; nunca eludas fallos reales de test ni el sandbox del producto en pruebas.

### Ejecuta las comprobaciones pertinentes localmente

Ejecuta las comprobaciones antes de los pushes mediante [dsh-pre-push-checks](.agents/skills/dsh-pre-push-checks/SKILL.es.md); reporta solo los comandos ejecutados. Después de `gh stack sync`, valida de inmediato; no hagas merge antes de que las comprobaciones pasen.

- Adecúa la evidencia a la superficie: tests enfocados para el comportamiento, instantáneas para la salida del modelo o del usuario, `doc-sync` para la documentación, build/hygiene y smokes de lo construido para las rutas publicadas, y e2e de API real para el comportamiento de los providers.
- Nunca recurras por defecto a la suite completa ni repitas una comprobación que ya pasa para commit o push. CI es dueña de la cobertura exhaustiva y de la matriz de plataformas; ensaya todo localmente solo por petición explícita, para diagnosticar CI o para un cambio irreduciblemente global del repositorio.
- `test:coverage`, no `test`, es el gate de cobertura de CI ([por qué](docs/testing.es.md)).

## Secretos / .env

Los tests de API real y las demos leen `DEEPSEEK_API_KEY`, la opcional `DEEPSEEK_BASE_URL` y el `.env` de la raíz. cordis.yml permite `!!js` (nunca `!js`) en `config` del plugin y en la entrada `disabled`; el resto de los metadatos permanece literal, por lo que la composición condicional también usa overlays ([primer](docs/cordis-primer.es.md#loader-configuration)). Nunca hagas commit de credenciales. El e2e de CI se salta sin clave; [testing.md](docs/testing.es.md) es dueño de la política de claves.

## Convenciones

- Cada paquete npm es `@deepseek-ai/dsh-<name>`; los paquetes vendored se re-escopean ([mapeo](docs/rescope.es.md)) y son `private: true`. `@deepseek-ai/cordis` es un peerDependency (+ dev) de cada paquete del harness.
- ESM en todas partes (`"type": "module"`). Usa nombres de paquete entre paquetes y `.ts` en los imports relativos locales. Los subprocesos de config ejecutan `lib/` compilado bajo Node normal; las regresiones de fuente usan su launcher declarado ([política de tests](docs/testing.es.md#test-subprocess-launch-modes)). El arranque de la CLI `dsh` desde la fuente pasa por el hook solo-ESM de tsx (`node --import tsx/esm`); los módulos a los que llega deben seguir siendo ESM (sin exports solo-CJS) — los modos nativos de TypeScript de Node no están disponibles en todo el rango de engines ([contrato de arranque desde fuente](.agents/notes/implemented/architecture/2026-07-29-dsh-source-launch-tsx-esm.es.md)). Los plugins simples de `cordis.yml` Raw/Web deben aparecer en `dependencies` del manifest de su resolver; `verify-cordis-config` lo exige.
- **Los registros son efectos**: toda contribución pasa por `ctx.effect()` / `ctx.on()`; el `register()` de un registro devuelve el disposer.
- **Los invariantes de runtime afirman relaciones de propiedad.** Comprueba los streams de eventos autoritativos o los datos mutables, no la presencia de servicios o métodos, los metadatos o efectos de plugins, ni los ejemplos puros fijos. Sin una relación plausible, un compañero vacío explicado es lo correcto ([reglas de invariantes de paquetes](packages/AGENTS.es.md)).
- **Los eventos tipados usan declaración por merging** y mapas extensibles por merging. El JSDoc de eventos necesita `@mode` y `@param` del payload; las claves con alcance ausentes de los payloads necesitan `@dshScopeScan unsupported`. Los métodos públicos de servicios documentan parámetros y retornos no void. Un miembro de `SessionEventMap` se exige al leerlo por defecto — los builds que no conocen su tipo rechazan el log salvo que el evento lleve `ignorable: true` del envelope; solo los cambios estructurales de formato suben `SESSION_FORMAT_VERSION` ([mecanismo](.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.es.md)).
- **Cambia según los tags discriminantes.** Las uniones cerradas terminan en `assertNever`; las uniones extensibles por merging pasan por un default documentado.
- **Los listeners de waterfall DEBEN llamar a `next()`** para delegar; retornar sin hacerlo corta la cadena ([semántica](docs/cordis-primer.es.md#cordis-waterfall-semantics)).
- **Visible para el modelo ⟺ registrado**: cualquier cosa que llegue a una solicitud del modelo debe poder reconstruirse desde el log de sesión; una entrada nueva visible para el modelo requiere un evento de sesión.
- **Plugins, no cambios del loop**: el comportamiento nuevo va en puntos de extensión documentados; cambiar `agent-loop` exige actualizar docs/architecture.md.
- **Un seam de capacidad comprende los roles de Service Definition / Service Provider / Consumer.** Está completo, nunca es un solo rol; se divide solo cuando los roles evolucionan de forma independiente ([glosario](docs/glossary.es.md#capability-seam)).
- **Prefiere dependencias mantenidas antes que implementar a mano** cuando de verdad eliminan código y tests propios ([política](.agents/notes/implemented/process/2026-07-26-dependencies-over-hand-rolling.es.md)).
- **Lo explícito > lo implícito en los límites de paquetes**: el valor por defecto es un paso explícito `resolve(request): Spec` en la implementación propietaria, nunca un `?? default` oculto dentro de `run()` (la división request/spec de `dsh-shell` es la plantilla).
- **Sin parámetros ajustables hardcodeados en los plugins**: las elecciones que varían según el despliegue son campos `Config` validados y modificables desde cordis.yml; una constante `DEFAULT_*` o un test hook no es configurabilidad. Las constantes de protocolo, las especificaciones externas y los invariantes de seguridad permanecen fijos.
- **La mala configuración falla ruidosamente** en la carga cuando es autocontenida; si no, en el punto resoluble más temprano; nunca omitas silenciosamente un referente ausente.
- **Los ids opacos entre límites están marcados** (`Branded<B>` de `dsh-brand`), nunca un `string` desnudo.
- **Confía en TypeScript en los límites tipados del mismo proceso.** No añadas validación en runtime, comportamiento de respaldo ni tests de entradas hostiles solo por valores que la interfaz estática exige; valida en los límites parser/config, en cola, JSON de modelo/herramientas, durable/archivo, worker, proceso y wire.
- **Plano de fuente vs plano de artefactos, nunca mezclados.** Los gates y tests estáticos resuelven los imports del workspace a través de `paths` de tsconfig hasta `src` y pasan en un árbol limpio; los gates que consumen `lib/` compilado declaran esa dependencia ([disposición](docs/development.es.md#typescript-project-layout)).
- **Mantén explícitas las faces del compilador.** Cada paquete usa un aggregate, excepto `api/remotes`; los programas de todo el repo siembran una config de face, nunca la solution raíz ([disposición](docs/development.es.md#typescript-project-layout)).
- **Un `catch` vacío nombra lo que engulle** y por qué nada más puede llegar a él; limita el `try` a una sola sentencia.
- No comentes hechos obvios a partir del código.
- **Prefiere la simetría para valores paralelos**; una asimetría sin explicación suele indicar una extracción perdida.
- **Los tests describen comportamiento, no corrección.** Cambia el comportamiento obsoleto junto con sus tests; explica el porqué en el PR.
- **Los cambios no triviales DEBEN incluir un Agent Note en el mismo PR;** solo las ediciones mecánicas o locales están exentas ([alcance](.agents/notes/README.es.md#when-to-write-one)). Las notas archivadas están congeladas: nunca las edites ni las trates como autoridad vigente ([política de archivado](.agents/notes/README.es.md#archiving-and-deletion)).
- **Política de tests** — [docs/testing.md](docs/testing.es.md). Todo cambio no trivial de comportamiento visible para el modelo o para el usuario del producto añade o actualiza una instantánea sin clave mediante un ejemplo real ejecutable en el mismo PR; los tests de paquete, las aserciones solo-e2e y los fixtures solo-mock no sustituyen al transcript (transcripción) de la aplicación ensamblada. Los fixtures deben poder reproducirse en macOS/Linux; corrige los fixtures, no los normalizadores.
- **La intención de renderizado en la UI de una herramienta es parte de su diseño**, decidida de antemano (`generic`/`terminal`/`diff`, `locations`); los métodos de presentación son funciones puras de `args` ([recetario](docs/cookbook/adding-a-tool.es.md)).
- **Planifica la cobertura unit, e2e y de instantáneas** para los seams de capacidad, las rutas del ciclo de vida y la salida de transcripts; incluye en el mismo cambio el soporte de snapshot-harness que falte.
- **Ambos SDK proyectan el loop.** Los cambios de agent-loop, session-lifecycle y `SessionEventMap` actualizan las salidas esperadas de los SDK de TypeScript y Python en el mismo PR; `pnpm run test` no cubre ninguna de las dos ([superficies](docs/testing.es.md#when-a-snapshot-test-is-required)).
- **Elige la historia del PR deliberadamente.** Divide los cambios independientes; corrige el PR que introdujo el problema antes de propagar. Los PR independientes y las pilas oficiales pueden hacer merge-forward o rebase tras la revisión. Las reescrituras usan `--force-with-lease`, abortan si el remoto se movió, nunca `--force` crudo; un merge-forward en curso conserva su checkpoint antes de tomar una base más nueva ([justificación](.agents/notes/implemented/process/2026-08-02-native-github-stacks-and-optional-rebases.es.md)).
- **Etiquetas:** un PR `kind/*`, todos los `area/*` pertinentes y el Issue Type nativo ([taxonomía](.agents/notes/implemented/process/2026-08-08-unified-github-label-taxonomy.es.md)).
- Marcadores TODO: `FIXME`/`TODO`/`XXX` según urgencia ([semántica](docs/development.es.md)).
- Los archivos terminan con exactamente un salto de línea final; `git diff --cached --check` (pre-commit) lo comprueba.

## Patrones defensivos

Lee [docs/defensive-patterns.md](docs/defensive-patterns.es.md) antes de trabajar con ciclo de vida, concurrencia, subprocess o teardown.

## Seguridad de tipos y documentación

Todo compila bajo `strict: true` con `noImplicitAny`; cada `any` restante explica por qué el estrechamiento es inviable. Cada módulo y export tiene un JSDoc conciso para su contrato no obvio; los exports similares a funciones incluyen `@param`/`@returns`, como exige `verify-export-jsdoc`. Los miembros declarados por herencia, los slots del protocolo de plugins y los constructores mantienen su documentación en la Service Definition, el protocolo o la clase que los declara.

Los comentarios y la documentación declaran contratos y contexto completos, no transcript (transcripción) de razonamientos. Usa términos directos y concretos. No uses metáforas. Antes de escribir `contract`, `boundary` o `shape`, pregúntate si un término más exacto nombra el sujeto: escribe `response fields`, `JSON validation` o `ESM exports` en lugar de `response shape`, `validation boundary` o `module shape`. Reserva `contract` para precondiciones, postcondiciones, invariantes, promesas de compatibilidad y demás obligaciones de las que dependen callers, callees, implementers, providers, producers o consumers. Mantén un boundary literal de proceso, wire, seguridad, transacción o ciclo de vida. No narres el flujo de control ni los tests, no conserves historia de revisión ni reformules código. Conserva los hechos de comportamiento, fallo, temporización, propiedad y uso seguro; enlaza la justificación. Usa [dsh-prose-standard](.agents/skills/dsh-prose-standard/SKILL.es.md) para las decisiones. Conecta los invariantes comprobables mecánicamente a un gate de nivel superior ejecutado y demuestra que cada ruta de aceptación modificada rechaza un caso inválido. Usa excepciones estrechas y justificadas en lugar de desactivar una regla globalmente.

La documentación acompaña a cada cambio de código: actualiza juntos los README y los contratos JSDoc afectados. El trabajo bilingüe de rutina sigue [docs/AGENTS.md](docs/AGENTS.es.md); solo la invocación explícita del usuario puede ejecutar `dsh-translate-docs`. La prosa de estado actual, una línea física por párrafo, un hogar por hecho y los presupuestos de palabras viven allí.

## Editar estas instrucciones

`CLAUDE.md` es un symlink de `AGENTS.md` en la raíz, en `packages/` y en `examples/`; edita el archivo real. Mantén cada regla autocontenida y a la vez enlaza los documentos de alto nivel. Condensa cuando la claridad lo permita; sube el techo de `verify-doc-budgets` cuando el contenido requerido necesite de verdad más espacio.

## Política de vendoring

Los paquetes de `vendor/` son copias de fuente fijadas (manifest con los SHA upstream en [vendor/README.md](vendor/README.es.md)). Actualízalos mediante el procedimiento de sync que allí se indica; vuelve a aplicar o retira las modificaciones locales registradas; vuelve a ejecutar `pnpm run test && pnpm run build`.
