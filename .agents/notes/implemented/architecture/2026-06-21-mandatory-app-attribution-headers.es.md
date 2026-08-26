# Agent Note: Atribución obligatoria de `User-Agent` para las solicitudes a providers

Status: implemented

[English](2026-06-21-mandatory-app-attribution-headers.md) | [中文](2026-06-21-mandatory-app-attribution-headers.zh.md) | Español

## Problema

Las solicitudes a providers de LLM (modelo de lenguaje de gran tamaño) deberían identificar el producto que las realiza. Eso es útil para el soporte del lado del provider, la investigación de abusos, la depuración de compatibilidad y la analítica de tráfico. Antes de esta Agent Note el harness solo lo hacía parcialmente: el adaptador DeepSeek hecho a mano enviaba una constante `User-Agent` copiada a mano (`packages/llm/llm-deepseek/src/adapter.ts`), mientras que su gemelo basado en pi-ai no enviaba ninguna cabecera propiedad del harness (`packages/llm/llm-pi-ai/src/adapter.ts`). Los adaptadores nuevos podían por tanto omitir la atribución en silencio, y un adaptador respaldado por una librería podía desviarse del adaptador hecho a mano aunque [la Agent Note de los adaptadores gemelos](2026-06-13-twin-llm-adapters.es.md) exista para mantener honesto el contrato de provider en ambas implementaciones.

El desencadenante inmediato vino de la documentación de [Atribución de la aplicación](https://openrouter.ai/docs/app-attribution) de OpenRouter. OpenRouter crea páginas y rankings de aplicaciones a partir de `HTTP-Referer` más las cabeceras de visualización/categoría. Eso es valioso, pero no es el estándar HTTP de identidad de aplicación. El riesgo es adoptar el conjunto exacto de cabeceras de OpenRouter como si fuera universal y después filtrar cabeceras específicas del provider hacia las solicitudes directas a DeepSeek, los adaptadores futuros de OpenAI/Anthropic/Vertex, los servidores de prueba o los proxies que registran campos desconocidos indefinidamente.

## Investigación

- **El mecanismo de OpenRouter es específico del provider.** Su documentación actual dice que la atribución de la aplicación se rastrea mediante `HTTP-Referer` (obligatorio), `X-OpenRouter-Title` y `X-OpenRouter-Categories`; `X-Title` solo se acepta por retrocompatibilidad. Su referencia de API llama a estas cabeceras opcionales y dice que hacen la aplicación descubrible en OpenRouter. Es un contrato concreto de OpenRouter, no un estándar IETF ni de API compatible con OpenAI.
- **En las herramientas de agentes, `HTTP-Referer` es una convención propia de OpenRouter, no una convención general de agentes.** Es lo bastante común como para que los SDK de OpenRouter y los ejemplos de OpenRouter la expongan directamente, y los frameworks orientados a OpenRouter suelen necesitar una forma de pasarla. Pero los protocolos de agentes como ACP negocian nombres, versiones y capacidades en sus propios mensajes de initialize, mientras que las solicitudes a providers de modelo siguen necesitando identidad a nivel HTTP. «Aceptado en el mundo de los agentes» significa por tanto «reconocido por las integraciones de OpenRouter», no «portable entre runtimes de agentes o providers».
- **Los coding agents identifican el producto y la versión en `User-Agent`.** Las implementaciones públicas varían en el detalle del entorno y en las cabeceras laterales específicas del provider, pero la identidad del producto es el contrato común; no hay un formato exacto universal.
- **La cabecera estándar en vía de estandarización para la identidad general del cliente es `User-Agent`.** La sección 10.1.5 del RFC 9110 define `User-Agent` como la identidad del software user agent, dice que se usa para informes de interoperabilidad y analítica, y dice que un user agent SHOULD enviarla en cada solicitud salvo que esté configurado para no hacerlo. Es la única cabecera estándar que coincide directamente con «qué producto está haciendo esta solicitud HTTP».
- **`Referer` es estándar, pero el `HTTP-Referer` de OpenRouter no es el campo estándar.** La sección 10.1.3 del RFC 9110 define `Referer` como el URI del que se obtuvo el URI de destino y dedica bastante texto a las restricciones de privacidad. OpenRouter pide en cambio `HTTP-Referer` y lo usa como identificador de URL de la aplicación. Ese nombre y ese significado son específicos de OpenRouter aunque se parezca a la forma de variable de entorno CGI de la cabecera estándar `Referer`.
- **`From` es estándar, pero no es adecuada como valor por defecto obligatorio.** La sección 10.1.2 del RFC 9110 define `From` como una dirección de correo de la persona responsable de un user agent. Los agentes robóticos SHOULD enviarla para que los servidores puedan ponerse en contacto con un operador, pero los agentes no robóticos no deberían enviarla sin configuración explícita del usuario por preocupaciones de privacidad y política de seguridad. El harness puede admitir un contacto de operador más adelante, pero no debe inventar uno ni exigirlo globalmente.
- **Los campos `user` o `metadata` del cuerpo de la solicitud no son atribución de la aplicación.** Algunas APIs de modelo exponen un identificador estable de usuario final, metadatos de solicitud, etiquetas o cabeceras de proyecto/cuenta. Eso es útil para la supervisión de abusos, la facturación interna, los paneles o la correlación de trazas, pero identifican al usuario final en lugar del producto, son schema de cuerpo específico del provider o no se garantiza que los reenvíen los gateways compatibles con OpenAI. No sustituyen a una cabecera estática de identidad de aplicación.
- **Las cabeceras de telemetría de los SDK identifican al SDK, no a la aplicación.** Los SDK oficiales y de terceros suelen enviar cabeceras de librería/versión. Eso ayuda al mantenedor del SDK a depurar su cliente, pero no identifica al harness como la aplicación salvo que la aplicación suministre explícitamente una capa de atribución de producto.
- **pi-ai tiene un hook de cabeceras de primera clase.** `StreamOptions.headers` de `@earendil-works/pi-ai` fusiona las cabeceras del llamador en último lugar, por encima de los valores por defecto del provider, así que un adaptador respaldado por una librería puede cumplir el mismo contrato de wire que el hecho a mano sin envolturas ni trabajo aguas arriba. Las suites de mock-server comprueban la llegada al wire en ambos adaptadores.

## Decisión

La atribución de aplicación neutra respecto al provider es obligatoria en el límite de los adaptadores LLM, usando solo la cabecera estándar `User-Agent`. La regla: cada adaptador LLM de producto envía una identidad de aplicación estática y no secreta en cada solicitud HTTP al provider, y cada adaptador tiene pruebas de que `User-Agent` llega al wire (un mock server que comprueba las cabeceras recibidas; para un adaptador respaldado por una librería, el hook de cabeceras de la librería alimenta la misma comprobación del mock server). Esta regla rige la atribución de la aplicación, no la identidad de solicitud específica del provider: [la decisión de identidad de solicitud de DeepSeek](../feature/2026-08-11-deepseek-request-user-id-header.es.md) es dueña por separado de sus cabeceras de usuario y sesión.

La atribución de aplicación de OpenRouter se deja deliberadamente sin implementar. `HTTP-Referer`, `X-OpenRouter-Title`, `X-Title` y `X-OpenRouter-Categories` son cabeceras de superficie de producto específicas de OpenRouter, no atribución de solicitud de modelo neutra respecto al provider. Pueden proponerse más adelante mediante un adaptador de OpenRouter o un modo OpenRouter explícito, con su propia decisión de producto/privacidad, pruebas y documentación. Hasta entonces, incluso las solicitudes dirigidas a OpenRouter envían solo la atribución `User-Agent` compartida de esta decisión.

La identidad neutra respecto al provider es propiedad de `dsh-llm` (`packages/llm/llm/src/attribution.ts`), no de adaptadores individuales. `AppIdentity` contiene solo los datos públicos de producto necesarios para construir `User-Agent`, y los valores por defecto de `APP_IDENTITY`:

- token de producto para `User-Agent`: `deepseek-harness` (continuidad con el valor de wire anterior a la Agent Note y con la identidad del repo/org)
- versión: leída del manifest (manifiesto) del paquete propietario mediante `createRequire`, nunca una constante copiada a mano
- URL de la aplicación: `https://github.com/deepseek-ai/deepseek-harness` — la página de inicio del repositorio

El valor por defecto es obligatorio y no vacío. Los despliegues de marca blanca pasan su propio `AppIdentity` a `attributionHeaders(identity)` — el hook de anulación es el parámetro de la función, sin plomería de configuración de despliegue hasta que un consumidor la necesite — y omitirlo recae en el valor por defecto del harness en lugar de suprimir la atribución. No hay ninguna API por solicitud para que el modelo, el prompt del usuario, el id de sesión, el cwd, el correo del usuario, el propietario de la clave API o la identidad de la máquina local influyan en estos campos.

Mapeo en el wire (`attributionHeaders`; los nombres de cabecera van en minúsculas en el código — los nombres de campo HTTP no distinguen mayúsculas en el wire):

| Destino | Mapeo |
|---|---|
| Todos los adaptadores basados en HTTP | `User-Agent: {product}/{version} (+{url})` — el comentario `+url` entre paréntesis se mantiene dentro de la sintaxis conservadora de producto/comentario del RFC 9110. |
| Endpoint directo de DeepSeek | `User-Agent` para la atribución de la aplicación; `x-deepseek-harness-user-id` y el condicional `x-deepseek-harness-session-id` son identidad de solicitud separada bajo la decisión específica de DeepSeek. No envíes cabeceras solo de OpenRouter salvo que DeepSeek documente un contrato equivalente. |
| Endpoints de OpenRouter | `User-Agent` solo por ahora. No envíes `HTTP-Referer`, `X-OpenRouter-Title`, `X-Title` ni `X-OpenRouter-Categories` bajo esta decisión. |
| Providers futuros | Solo `User-Agent` salvo que una Agent Note específica de provider acepte cabeceras adicionales. No reutilices `HTTP-Referer` por analogía. |

La detección de endpoints no forma parte de esta Agent Note porque aquí no se acepta ningún mapeo específico de endpoint. Si el soporte de OpenRouter llega más adelante, la detección debe ser explícita: un paquete de provider dedicado de OpenRouter o una configuración explícita `provider: 'openrouter'` / `attributionTarget: 'openrouter'`, no fragmentos de ruta arbitrarios ni nombres de modelo.

## Verificación

El contrato implementado:

- `dsh-llm` documenta el contrato obligatorio de atribución `User-Agent` para los autores de `LlmAdapter` (JSDoc de `LlmAdapter`, README del paquete y la sección de contrato de adaptadores de `docs/subsystems/llm-streaming.md`).
- Un helper compartido (`attributionHeaders` / `userAgent`) construye la identidad de la aplicación y el valor estándar de `User-Agent` a partir de los metadatos del paquete, para que los adaptadores no copien a mano constantes de versión.
- `dsh-llm-deepseek` envía el `User-Agent` compartido en cada solicitud y su suite de mock-server comprueba el valor exacto.
- `dsh-llm-pi-ai` envía el mismo `User-Agent` a través del hook `StreamOptions.headers` de pi-ai y su suite de mock-server comprueba el valor exacto.
- Ningún adaptador envía cabeceras de atribución específicas de OpenRouter (`HTTP-Referer`, `X-OpenRouter-Title`, `X-Title`, `X-OpenRouter-Categories`) como parte de esta decisión.
- Ningún campo de atribución de la aplicación lleva secretos, rutas locales, ids de sesión, texto de prompt, salida del modelo, correo del usuario ni identificadores estables por usuario.
- Los README de los adaptadores declaran la política de atribución `User-Agent` y evitan explícitamente documentar la atribución de aplicación de OpenRouter como comportamiento implementado.

## Alternativas consideradas

**Atribución de aplicación de OpenRouter ya.** Rechazada para esta decisión. Enviar `HTTP-Referer` más `X-OpenRouter-Title` satisfaría los rankings de OpenRouter, pero esas cabeceras son una funcionalidad de producto específica del provider, no la atribución de solicitud de modelo neutra respecto al provider que esta decisión estandariza. Darles soporte debería ser una decisión explícita de adaptador/modo de OpenRouter más adelante, no algo escondido dentro del primer helper compartido de atribución.

**Cabeceras de OpenRouter en todas partes.** Rechazada. Trataría un contrato personalizado de OpenRouter como estándar universal y enviaría campos con semántica engañosa a providers que no los pidieron. También arriesga usar `HTTP-Referer` como campo genérico de URL de aplicación aunque el HTTP estándar ya tiene `User-Agent` para la identidad de producto y `Referer` para un concepto distinto de contexto de navegación.

**Solo identidad de cuenta/proyecto del provider.** Rechazada. Las cabeceras de organización/proyecto, las claves API, las cuentas en la nube y los proyectos de facturación identifican quién paga o posee la solicitud, no qué aplicación envía el tráfico. Además no exponen ningún título/categoría público de la aplicación y no ayudan a gateways como OpenRouter a construir rankings de aplicaciones.

**Campos `user`/`metadata` del usuario final.** Rechazados para esta Agent Note. Son valiosos para la supervisión de abusos y el soporte al cliente, pero describen a la persona o al inquilino detrás de una solicitud. La atribución de la aplicación debe ser identidad de producto estática y segura de enviar en cada solicitud.

**Atribución opcional solo por configuración.** Rechazada. Un ajuste desactivado por defecto es exactamente cómo los adaptadores siguen desviándose. La política es atribución obligatoria por defecto con valores públicos anulables, no atribución opcional.

**Token con nombre de SDK (`deepseek-harness-sdk`).** Considerado para el token de `User-Agent` porque la pila de cliente del runtime compatible usa el nombre del SDK. Ganó `deepseek-harness` porque nombra al producto DeepSeek Harness, coincide con la identidad de org/repo y el ámbito del paquete, y mantiene estable la atribución de wire sin llamar SDK al producto completo.

## Consecuencias

**Los providers ven que el tráfico viene del harness.** Ese es el punto, pero significa que los despliegues que antes se fundían con el tráfico genérico de SDK se vuelven identificables. Mitigación: enviar solo datos públicos estáticos de producto y dejar que los forks y los despliegues de marca blanca pasen su propio `AppIdentity`.

**El soporte de cabeceras difiere según la librería cliente.** El adaptador hecho a mano fija las cabeceras directamente; el adaptador respaldado por pi-ai depende de que pi-ai siga honrando `StreamOptions.headers` (fusionadas en último lugar, por encima de los valores por defecto del provider). Las pruebas de mock-server a nivel de wire son la salvaguarda: si una actualización de pi-ai deja de entregar la cabecera, la suite se pone en rojo. Es una presión útil sobre la abstracción: un adaptador de provider que no puede fijar cabeceras obligatorias no puede implementar por completo el contrato LLM del harness.

**Los rankings de OpenRouter aún no se benefician.** `User-Agent` es la línea de base correcta para la identidad HTTP neutra respecto al provider, pero no creará páginas ni rankings de aplicaciones en OpenRouter porque OpenRouter exige `HTTP-Referer` para esa funcionalidad de producto. Es deliberado: la participación en un marketplace público de aplicaciones es una decisión de producto separada, no un requisito previo de la atribución obligatoria de solicitudes.
