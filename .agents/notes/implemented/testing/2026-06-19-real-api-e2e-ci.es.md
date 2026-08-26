# Agent Note: e2e de API real en CI contra la API externa de DeepSeek

Status: implemented

[English](2026-06-19-real-api-e2e-ci.md) | Español

## Problema

El harness se apoya con fuerza en las pruebas de API real por política: [docs/testing.md](../../../../docs/testing.md) argumenta que una suite sin clave demuestra la fontanería pero no el producto, y el [post-mortem de inyección ACP](../../../../docs/postmortem/0001-acp-default-export-drops-inject.md) es la prueba permanente — 178 pruebas sin clave siguieron en verde mientras una sesión real de cliente ACP se estrellaba al instante. La suite e2e de API real (`pnpm run test:e2e`, los archivos `*.e2e.ts`) existe precisamente para cerrar esa brecha: conduce al agent contra la API en vivo de DeepSeek — llamadas de modelo reales, herramientas bash reales, multi-turno, resumen (resume), ACP sobre stdio.

La puerta por defecto ([.github/workflows/ci.yml](../../../../.github/workflows/ci.yml)) es deliberadamente sin clave: no lleva ningún secreto y corre para forks. `test:e2e` se auto-omite sin clave (`describe.skipIf(!process.env.DEEPSEEK_API_KEY)`), así que añadirla allí reportaría verde sin ejercitar la suite real. Se requiere un flujo separado con secreto para convertir la cobertura de API real en una señal de merge.

## Decisión

Un flujo dedicado, [.github/workflows/e2e.yml](../../../../.github/workflows/e2e.yml), separado de ci.yml, corre solo `pnpm run test:e2e` contra la API externa usando un secreto del repo, en eventos de confianza, con un preflight que convierte un secreto faltante en un fallo ruidoso en lugar de un verde falso. El flujo sin clave permanece separado para que las puertas de calidad forkables y las puertas de API real que consumen secretos mantengan políticas de disparador y credenciales distintas.

### Un flujo separado, no un job en ci.yml

El valor de ci.yml es que es sin clave, forkable y siempre verde: cualquier contribuidor (incluido un fork externo) obtiene una señal completa sin clave sin ningún secreto en el radio de explosión. Añadir allí un job que consuma secretos acoplaría esa puerta siempre verde a la disponibilidad de credenciales y a una política de disparador distinta. Mantener el trabajo con secreto en su propio archivo aísla el secreto, el disparador y la política de concurrencia, y preserva la propiedad de ci.yml para los forks. Ciclos de vida distintos → archivos distintos.

### El costo no es la restricción; la fiabilidad lo es

El costo de inferencia interna no es la restricción limitante, así que el flujo optimiza por cobertura y señal. Corre cada archivo `*.e2e.ts` que coincida en múltiples disparadores y en cada PR de confianza, implementando la política con clave de [docs/testing.md](../../../../docs/testing.md).

### Disparadores: solo eventos de confianza

`workflow_dispatch` + `push` a `main`/`master` + `schedule` nocturno (`17 0 * * *`, 08:17 Asia/Shanghai) + `pull_request`. Push da una señal post-merge; el schedule detecta la deriva de la API externa; dispatch es la puerta de escape manual; y los pull requests de confianza obtienen una puerta pre-merge. Esa señal pre-merge acepta deliberadamente la mayor superficie de exposición de la clave descrita en § Seguridad.

### La puerta del PR no confiable

GitHub retiene los secretos del repo en dos clases de PR: los de **forks** y los PR de **Dependabot** (rama del mismo repo, así que `head.repo.fork == false`, pero los secretos se retienen igualmente). Un `if:` a nivel de job omite todo el job para ambas:

```
github.event_name != 'pull_request'
  || !(github.event.pull_request.head.repo.fork || github.event.pull_request.user.login == 'dependabot[bot]')
```

La cláusula de Dependabot se fija en el **autor** del PR (`pull_request.user.login`), no en `github.actor` (el disparador de la ejecución): un mantenedor que reabra o re-ejecute un PR de Dependabot haría que `github.actor` fuera un humano mientras el PR sigue sin clave, y una prueba basada en el autor sigue siendo correcta a través de eso. Un job omitido por un `if:` **a nivel de job** se reporta como una comprobación *exitosa* (a diferencia de una omisión a nivel de flujo/disparador, que permanece pendiente), así que este flujo es seguro de marcar como comprobación de estado requerida si se desea — la comprobación omitida-pero-verde de un PR de fork/Dependabot no bloquea el merge.

La puerta es una *conveniencia de omisión limpia*, no el límite de seguridad del secreto (véase § Seguridad — el límite es la retención de secretos de fork propia de GitHub bajo `pull_request`). Sin la puerta, los forks tampoco podrían leer la clave; solo se toparían con un fallo duro de preflight confuso y malgastarían cómputo.

### Preflight: falla en voz alta, nunca verde falso

Como el job solo corre en eventos de confianza donde se espera el secreto, el preflight es una comprobación de presencia incondicional: clave vacía → `exit 1` con una anotación `::error::` que nombra el secreto a configurar. Este es el quid que hace que una suite auto-omitente sea segura como puerta. Sin él, un secreto eliminado/renombrado/mal configurado haría que `test:e2e` omitiera todas las suites reales y reportara todo verde — una regresión silenciosa de toda la red de seguridad. La guardia convierte «secreto faltante» de un pase falso invisible en un fallo visible. (Su corrección se verificó en vivo: la ejecución anterior a la existencia del secreto falló exactamente en este paso.)

### Mapeo e higiene del secreto

El secreto del repo se llama `DEEPSEEK_API_KEY_EXTERNAL`; se mapea a la variable de entorno `DEEPSEEK_API_KEY` que leen los adaptadores y las pruebas (`process.env.DEEPSEEK_API_KEY`). El nombre de secreto distinto documenta la intención (esta es la clave *externa* de la API pública, no una clave de endpoint interno) y permite que una clave de endpoint interno coexista después sin colisión. Elecciones de higiene, cada una defensiva:

- **Secreto con ámbito de paso.** `DEEPSEEK_API_KEY` se fija en el `env:` solo de los pasos de preflight y e2e, nunca a nivel de job — así checkout/setup-node/install nunca lo ven. Un script de ciclo de vida de instalación comprometido en una dependencia no puede leer un secreto que no está en su entorno.
- **`permissions: contents: read`.** El job solo lee el repo para correr pruebas; no necesita ámbitos de escritura (sin comentarios de PR, sin escrituras de estado), así que el `GITHUB_TOKEN` se reduce al mínimo privilegio.
- **`DEEPSEEK_BASE_URL` fijado** a `https://api.deepseek.com` en el paso e2e. El adaptador tomaría ese valor por defecto cuando no está fijado ([packages/llm/llm-deepseek/src/index.ts](../../../../packages/llm/llm-deepseek/src/index.ts) `PUBLIC_BASE_URL`), pero fijarlo es auto-documentado y hermético — un `.env` suelto en la raíz del repo (que `vitest.e2e.config.ts` carga si existe) no puede redirigir silenciosamente la ejecución a otro endpoint.
- **Ningún eco del secreto.** El preflight imprime solo `DEEPSEEK_API_KEY present.` — no el valor ni su longitud.

### Alcance, forma del runtime

El job corre solo `test:e2e` en Node 24; las puertas sin clave y la compatibilidad de versiones pertenecen al flujo principal de CI. Las pruebas corren sin compilar a través del mapa de rutas del workspace con un pool de workers acotado y configurable, reintentos por prueba y un timeout de job. Las ejecuciones de PR superados se cancelan, mientras que las de push y programadas se completan para la señal post-merge.

La sonda nativa `web_search` de DeepSeek está registrada pero se omite. El endpoint compatible con Anthropic en vivo puede devolver una respuesta exitosa sin bloques de fuentes estructurados, así que su aserción de fuente positiva no es una señal de merge fiable; la cobertura unitaria sigue fijando el parseo de respuestas, pero CI no demuestra la forma de cable del bloque de fuentes en vivo.

## Seguridad

El primer secreto de CI del repositorio requiere un modelo de amenazas registrado porque el acceso difiere entre pull requests del mismo repo, de forks y de Dependabot, y cambia cuando el repositorio se vuelve público.

### Quién puede alcanzar el secreto hoy (repo privado)

- **Sin acceso de escritura (PR de fork): no puede.** Dos hechos independientes lo bloquean. Primero, el flujo usa `pull_request`, **no** `pull_request_target` — GitHub no pasa secretos del repo a las ejecuciones de PR de fork de `pull_request`, así que `secrets.DEEPSEEK_API_KEY_EXTERNAL` se resuelve a vacío en un runner de fork. Segundo, la puerta `if:` omite los PR de fork por completo. La retención es el límite real; la puerta es defensa en profundidad y UX.
- **Acceso de escritura (push): puede.** Un PR de rama del mismo repo recibe secretos, así que un autor con acceso de escritura podría modificar el código de pruebas (o un script de ciclo de vida de instalación, o el YAML del flujo en su rama) para exfiltrar la clave. Esto es **inherente a GitHub Actions, no introducido aquí**: cualquiera con acceso de push a cualquier repo ya puede exfiltrar cualquiera de sus secretos de Actions escribiendo un flujo. Acceso de escritura ⇒ acceso al secreto, siempre. La mitigación vive en quién recibe escritura y en la protección de ramas, no en este archivo.

Así que «cualquiera que pudiera abrir un PR puede robarla» es falso: solo el conjunto con acceso de escritura puede, y ese conjunto ya podría robar cualquier secreto que el repo tenga.

### La exposición residual que añade el disparador `pull_request`

Como las ejecuciones de PR están habilitadas, la clave se entrega a **el código en la rama del PR de un autor con acceso de escritura** antes del merge. Esto es una superficie mayor que `push` + `schedule` + `workflow_dispatch`, aceptada a cambio de una señal pre-merge dentro del conjunto de escritura de confianza. Si ese cálculo cambia, elimina el disparador `pull_request` conservando la cobertura post-merge, nocturna y bajo demanda.

### Qué cambia cuando el repo se vuelve público

El secreto permanece protegido del público **a través de este flujo**: `pull_request` se comporta de forma idéntica en un repo público — los PR de fork (ahora abribles por cualquiera) siguen sin recibir secreto, y en los repos públicos GitHub además pone las ejecuciones de PR de fork tras la aprobación del mantenedor, donde incluso una ejecución aprobada no recibe secreto (aprobar la ejecución no es lo mismo que entregar la clave). El conjunto con acceso de escritura no cambia con la visibilidad, así que la realidad del interno tampoco cambia.

Lo que empeora es el modelo *circundante*, y estas son las cosas a abordar antes de cambiar la visibilidad:

- **Los logs se vuelven legibles para el mundo.** Un eco descuidado de secreto que hoy se filtra a los miembros de la organización se filtraría a todo el internet y sería raspado en minutos. La disciplina de manejo de secretos (sin ecos de valor/longitud — ya hecho) importa mucho más.
- **La trampa `pull_request_target` se vuelve catastrófica.** Si alguien alguna vez «arregla» las ejecuciones de PR cambiando el disparador a `pull_request_target`, el flujo ejecutaría código de fork no confiable en el contexto del repo base **con** secretos — un vector completo de fuga de clave. Esto es casi benigno en un repo privado y desastroso en uno público. Un comentario `SECURITY —` sobre el disparador en e2e.yml prohíbe el cambio y apunta aquí.
- **Rota al cambiar.** La clave vivió en el CI de un repo privado; trata el ir-público como «asume expuesta» y rota `DEEPSEEK_API_KEY_EXTERNAL` en ese momento.
- **Acota el secreto tras controles.** Confirma que Settings → Actions → *«Send secrets to workflows from fork pull requests»* sigue **desactivado** (el único ajuste que realmente rompería el límite del fork), y considera mover la clave a un **Environment** de GitHub con revisores requeridos para que incluso el código fusionado la use solo bajo condiciones controladas y la rotación tenga un único hogar.

Nada de esto requiere cambiar el flujo para salir a público; son pasos operativos más el comentario de guardia `pull_request_target` ya añadido.

## Alternativas consideradas

- **Un job que consuma secretos dentro de ci.yml** — rechazado: acoplaría la puerta sin clave, forkable y siempre verde a la disponibilidad de credenciales y a una política de disparador/concurrencia distinta; ciclos de vida distintos, archivos distintos.
- **Omitir el disparador `pull_request`** (la superficie de exposición de clave menor) — rechazado por la señal pre-merge; la sección de Seguridad lleva el análisis de exposición aceptado.

## Consecuencias

Un segundo flujo de CI y el primer secreto del repo que mantener. La suite de API real ahora bloquea merges (pre-merge en PR de confianza, post-merge en la rama principal) y corre de noche, así que una rotura real en la interacción del agent con la API externa aflora en CI en lugar de solo en la ejecución local de un desarrollador — al costo de llamadas de API reales (pero internamente gratuitas) en cada PR de confianza y merge. El preflight hace que la mala configuración del secreto se auto-anuncie en lugar de desactivar silenciosamente la red.

El diseño lleva una superficie de restricciones documentada: el tradeoff de exposición de clave del disparador `pull_request` (elimínalo para endurecer), la dependencia de la puerta `if:` de la prueba de Dependabot basada en el autor, y la prohibición dura de `pull_request_target`. La lista de ir-público anterior es el compañero operativo — este Agent Note es el lugar que un futuro mantenedor debería releer antes de cambiar el conjunto de disparadores o la visibilidad del repo, en lugar de re-derivar el modelo fork/secreto desde cero.

El disparador programado se auto-desactiva tras 60 días de inactividad del repo (un comportamiento de GitHub); push/PR/dispatch son respaldos, y un monorepo activo no lo alcanzará. Se asume el egress del runner a `https://api.deepseek.com` — `ubuntu-latest` alojado por GitHub lo tiene; un runner auto-alojado con restricción de egress necesitaría conectividad confirmada antes de confiar en el nocturno.
