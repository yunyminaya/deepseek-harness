# Agent Note: Registros de credenciales y flujos de autorización

Status: implemented

[English](2026-08-13-credential-records-and-authorization-flows.md) | [中文](2026-08-13-credential-records-and-authorization-flows.zh.md) | Español

## Problema

El plano de credenciales del harness solo podía expresar un tipo de secreto: un valor detrás de un nombre de variable de entorno. `CredentialRef` es un identificador POSIX, la resolución superpone el entorno del proceso sobre un archivo gestionado y los fallbacks de `.env`, y todo consumidor lo lee por operación. Eso cubre una clave de API exactamente y nada más.

Algunas credenciales no son valores que se pueda decir a un despliegue que almacene. Se obtienen — con una conversación con un humano que abre una página, aprueba una cuenta y pega de vuelta un código — y lo que sale es un documento de token con una mitad de refresh que rota a espaldas del usuario. pi-ai modela esto directamente (`Credential = ApiKeyCredential | OAuthCredential`, un `CredentialStore` propiedad de la app, `Models.login()`), y el harness no tenía dónde poner nada de eso. `PiAiAdapter` construía su colección con `createModels()` y sin opciones, de modo que el store era el default en memoria de pi-ai: vacío en cada boot, descartado en cada cambio de configuración. `openai-codex`, cuyo único método es OAuth, fallaba por tanto toda request con `Provider is not configured` — [retirado del directorio](../bug-fix/2026-08-13-oauth-only-providers-withheld.es.md) como arreglo de release, lo que eliminó la oferta rota sin añadir la capacidad.

De la misma ausencia de plano seguían otras dos brechas. El descubrimiento ambiental propio de un provider se ejecutaba contra el entorno de proceso crudo, de modo que una clave que tuviera el seam de credenciales era invisible para él y nunca se buscaba un archivo de credenciales local. Y un login no tenía superficie desde la que ejecutarse, porque nada en el harness podía hacer una pregunta a un humano en nombre de un plugin.

## Decisión

Tres seams, cada uno dueño de una pregunta, y todo concepto de pi-ai detrás de un adaptador dentro de `llm-pi-ai`.

**`dsh-credentials` gana un segundo espacio de claves.** Un `CredentialRef` responde *qué hay detrás de este nombre de variable de entorno*; una `CredentialKey` responde *qué credencial tiene este plugin para este id*. La unión de registros es `{ kind: 'api-key', key?, env? } | { kind: 'grant', payload }` — la mitad api-key estructural porque el seam puede describirla, la mitad grant opaca porque una librería dueña de un formato de token sigue siendo su dueña. La única restricción sobre un payload es que sobreviva a un round trip de JSON, aplicada a la entrada y a la salida.

La clave es `<scope>/<id>` donde el scope es el **nombre registrado del plugin dueño**, no el del provider. Un usuario conoce `openai-codex`; qué familia de adaptadores responde por los bytes dentro de ese registro es exactamente lo que un nombre de provider desnudo pierde. Dos plugins que sirvieran el mismo nombre de provider leerían el payload del otro, y un registro dejado por un plugin desinstalado no podría distinguirse de uno vivo. La `/` mantiene además disjuntas las dos gramáticas, de modo que los espacios de claves no pueden colisionar. Esto asume que un adaptador registra una ruta de provider dada, que el registro de LLM ya aplica.

Los registros no se superponen. No hay entorno del que pueda leerse una concesión de autorización, de modo que la presencia del registro es todo el hecho, y la regla de valor vacío que gobierna las referencias no se aplica: un registro `api-key` que no lleva ni clave ni env declara que su dueño confirmó autenticación ambiental, que es configuración.

**`dsh-authorization` es dueño de la conversación, nunca del protocolo.** Un plugin que sabe cómo obtener su propia credencial registra un flujo bajo la `CredentialKey` que ese flujo escribe. El seam ejecuta un intento por clave, enruta un vocabulario neutral de avisos y prompts, y se asienta. Un segundo protocolo de autorización llega como otro flujo en lugar de como otro seam, y una superficie que renderiza un flujo los renderiza todos.

Dos elecciones cargan el peso:

- **El flujo es dueño de la escritura.** Que `run()` resuelva significa que el registro ya está confirmado a través de `ctx.credentials`; el seam confirma una confirmación que observó durante el intento — la presencia sola dejaría que una re-autorización hiciera pasar un registro obsoleto por fresco — y rechaza un flujo que resolvió sin una. Esto es lo que permite que `Models.login()` — que persiste a través del adaptador de store como parte de iniciar sesión — siga siendo el único escritor, en lugar de que la credencial se copie de vuelta y se escriba una segunda vez.
- **La interacción viaja con la request, no con un registro.** Quien inicia una autorización es quien puede hablar del tema con el humano, de modo que los prompts llegan exactamente a la página que preguntó, un llamador headless suministra una interacción que declina, y no hay provider ambiental que pueda estar ausente o ser ambiguo entre dos pestañas abiertas.

**`llm-pi-ai` sostiene las tres traducciones.** `credentialStoreFrom` mapea el `CredentialStore` de pi-ai sobre registros; `authContextFrom` responde las preguntas ambientales de pi-ai desde el seam de credenciales y luego desde el entorno de lanzamiento, con la existencia de archivos comprobada contra el sistema de archivos del proceso host; `registerPiAiFlows` reformula el `AuthEvent`/`AuthPrompt` de pi-ai en el vocabulario neutral y ejecuta `Models.login()`. Toda colección se construye con las dos primeras, que es lo que mantiene a un provider con sesión iniciada a través del rebuild de colección que provoca un cambio de configuración. Con una postura que funciona, el directorio deja de retener las rutas solo-OAuth y `openai-codex` se ofrece de nuevo.

El plano de credenciales sigue siendo opcional, como ya lo era para la resolución de referencias. Las lecturas responden «nada almacenado» sin servicio de credenciales, porque tal composición genuinamente no tiene credencial; las escrituras se rechazan por nombre, porque un login cuya concesión se evaporó informaría éxito y luego fallaría toda request. El registro de flujos está acotado al seam de autorización a través de `ctx.inject`, de modo que una composición headless o ACP se monta sin inicio de sesión y nada más cambia.

### Dos mecanismos que los seams necesitaron debajo

`withFileLock` toma un límite de espera por llamada. pi-ai ejecuta un refresh de OAuth *dentro* de `credentials.modify()`, de modo que la ruta de escritura del registro sostiene el lock a través de un round trip de red; el default de 2s se eligió para un render-y-rename y fallaría a todo otro escritor del documento. La cadencia de reintentos sigue fija — es una constante de protocolo — mientras que la espera se dimensiona por el poseedor más largo que un contendiente pueda encontrarse: las refs y los registros comparten un archivo y un lock, de modo que todo escritor del documento (`DOCUMENT_LOCK_WAIT_MS`, escrituras de referencia y borrados de registro incluidos) espera a que salga un refresh de OAuth, no solo la mutación que ejecuta uno.

Los bordes del seam reciben la misma disciplina que su ruta de escritura. Un decline de prompt es un resultado, no una ruptura — una interacción rechaza con `AuthorizationDeclinedError` y el intento se asienta `cancelled` — mientras que un aviso que una superficie no puede renderizar se registra y se pierde en lugar de fallar el flujo, y `authorization/settled` hace fanout con fallos de listener contenidos en los términos del seam de credenciales. En el lado del store, un registro api-key se admite antes de renderizarse (lo que `parseRecord` rechaza en el siguiente boot se rechaza en la escritura), y `llm-pi-ai` pregunta `isCredentialKeySegment` antes de dirigirse a un registro, de modo que una clave de ruta arbitraria declarada a mano se lee como «nada almacenado» en lugar de lanzar a mitad de resolución.

La retirada asienta un intento haya o no reaccionado su flujo a la señal. Se supone que un flujo se detiene cuando su señal dispara, pero uno que no lo hiciera sostendría su clave durante la vida del proceso, y desde fuera una clave acuñada es indistinguible de una ocupada. La ejecución huérfana se deja terminar por su cuenta.

## Alternativas consideradas

- **Poner la forma del `CredentialStore` de pi-ai en el propio seam.** Es la forma que funciona y ya está diseñada. También nombra `api_key`/`oauth` como las dos clases de credencial del mundo y clavea por id de provider, que es la pérdida de propiedad de arriba; una segunda familia de adaptadores tendría que fingir ser pi-ai para participar. La unión de registros es deliberadamente un paso más abstracta en exactamente dos lugares — la clave, y la opacidad de una concesión.
- **Un seam de interacción de login dedicado junto a `user-questions`.** Los prompts de autorización parecen preguntas, y reutilizar `ctx.userQuestions` era tentador. Pero ese seam está construido para que la llamada de herramienta de un modelo se pause en nombre de un agent: valida al agent llamador, rechaza a un llamador delegado y tiene un provider de UI ambiental único. Un prompt de autorización no tiene agent, debe llegar a la página de configuración que lo inició y puede retirarse por prompt mediante un callback de navegador que gane una carrera. Los vocabularios se superponen; los ciclos de vida no.
- **Leer `~/.codex/auth.json` en un store.** Hace que Codex funcione sin nada de esto, y pi-ai sería dueño del refresh. También ata el harness al formato de archivo privado de otra herramienta por un provider, y deja sin construir todo otro login.
- **Unir un segundo `begin()` al intento ya en ejecución.** Más amable que rechazar, hasta que dos humanos están respondiendo las preguntas del mismo flujo. El rechazo con `inFlight` en la entry permite que una superficie deshabilite el botón en lugar de descubrir el estado por error.
- **Mantener la retención solo-OAuth como red de seguridad.** Ahora ocultaría un provider que funciona. El predicado se elimina en lugar de dejarlo inerte; `docs/subsystems/credentials.md` y los README de paquete llevan lo que lo reemplazó.

## Consecuencias

`.credentials.yaml` gana una versión y dos secciones. Un boot actualiza en su lugar el layout plano reconocido previo al release — un mapeo plano todo-cadenas se anida verbatim bajo `refs:` bajo el lock del escritor — porque una clave almacenada a través de la página de Models por un build interno anterior debe sobrevivir al cambio de layout sin edición a mano y sin que falleen sus requests de modelo. Cualquier forma plana que el reconocedor no pueda demostrar que entiende conserva el rechazo por nombre con la migración manual declarada en el mensaje; el parser sigue leyendo exactamente un layout, y el paso de migración se retira con la postura previa al release en el primer release etiquetado. Todo fixture del repo que escribía el documento plano se reescribió; los fixtures de las suites de llm se les escaparon al propio cambio de registros y se arreglaron aquí.

`openai-codex` vuelve al selector de providers y al directorio de la página de Models. El inicio de sesión se ofrece para todo provider instalado que publique un login, que hoy son los 38 — 31 recogen una clave a través del prompt propio de pi-ai, seis ofrecen eso junto a un login de suscripción, y Codex ofrece solo el login de suscripción.

Lo que esto aún no incluye es la superficie: el contrato wire que lleva avisos y prompts al navegador, y el control de la página de Models que inicia un login. Hasta que eso llegue, los flujos solo son alcanzables en proceso, y un despliegue sigue configurando una clave tecleándola en el formulario de ajustes.

Dos límites se registran en los README de paquete en lugar de arreglarse. Un intento no es durable, de modo que recargar la página a mitad de login lo abandona. Y cerrar sesión es `deleteRecord`, que olvida el registro localmente sin decírselo al emisor; un provider que necesite una revocación del lado del servidor no tiene dónde declararlo.

## Pruebas

La suite del seam fija el ciclo de vida que es dueño: rechazo y liberación single-flight, retirada antes de que el flujo arranque y durante él, un flujo que ignora su señal, la confirmación de confirmación y el evento de asentamiento incluido el caso `failed` que un llamador ve como error lanzado. El compañero invariante fija que una clave asentada es una clave libre, porque una acuñada es invisible de otro modo.

`llm-pi-ai` cubre las tres traducciones contra un documento real de `$DSH_HOME` — una credencial api-key campo por campo, una credencial OAuth verbatim incluida su mitad de refresh, el registro de un plugin extraño saltado por scope y el rechazo de escritura sin servicio de credenciales — más todo miembro de `AuthEvent` y `AuthPrompt` reformulado, con `Models.login()` mockeado en el límite de colección ya que uno real abre un navegador. Dos pruebas de composición real arrancan el plugin con y sin el seam de autorización.

Los goldens de e2e web de `models-settings` y `onboarding-usable-provider` recuperan exactamente la línea de opción `openai-codex` que perdieron cuando se retiró — toda la diferencia de aplicación ensamblada que este cambio hace hoy, porque la página de Models aún no tiene control de login que registrar.
