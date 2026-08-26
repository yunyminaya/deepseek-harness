# Configura los modelos

[English](providers.md) | Español

Esta guía asume que iniciaste la interfaz web a través del [README raíz](../../../README.es.md#run). Los cambios de modelo surten efecto en la siguiente solicitud, sin reiniciar el servidor.

## Configura DeepSeek

Abre **Ajustes → Modelos**. La tarjeta de DeepSeek expone un único campo de clave de API; introduce la clave y guárdala.

![La página de Modelos: la tarjeta de DeepSeek, con Add provider y Add a custom provider debajo](providers-models-page.png)

Las claves son de solo escritura. Tras guardar, la página recibe un descriptor censurado, nunca el secreto literal. La clave se almacena en `$DSH_HOME/.credentials.yaml`, mientras que los ajustes conservan únicamente su referencia de credencial.

## Añade un provider del catálogo

Elige **Add provider**, selecciona un provider como Anthropic u OpenAI, introduce su clave de API y guarda. El catálogo instalado aporta el endpoint, el protocolo y la lista de modelos.

Los providers con autenticación nativa necesitan en su lugar sus credenciales nativas. Bedrock, Vertex, Azure y Codex usan, respectivamente, credenciales de AWS y una región, un proyecto ADC, una `api-version` y OAuth; rellenar solo el campo de clave de API no los configura.

## Añade un provider personalizado

Elige **Add a custom provider** para un gateway de empresa, un servidor autoalojado o un provider ausente del catálogo instalado. Proporciona un ID de provider en minúsculas, la URL base, el protocolo de API, la credencial y al menos un modelo.

![El formulario de provider personalizado: Provider ID, display name, base URL, protocolo de API y API key](providers-custom-form.png)

El ID de provider es permanente porque las solicitudes, las sesiones guardadas, los valores por defecto de modelo y las referencias de credencial lo usan. Para renombrar un provider, añade uno nuevo y elimina el antiguo. El nombre mostrado, la URL base, el protocolo, la credencial y los modelos siguen siendo editables.

En **Catálogo de modelos**, elige **Fetch available models** para consultar la URL base y la credencial que se muestran actualmente en el formulario. Seleccionar candidatos actualiza el borrador; el provider no se guarda hasta que guardes. Los providers del catálogo usan su catálogo instalado sin ninguna solicitud de red.

### Entrada de imágenes

Un modelo que introduces a mano se trata como solo texto hasta que diga lo contrario, porque nada puede preguntar a un endpoint qué modalidades acepta. Adjuntar una imagen a ese modelo se rechaza antes de enviarla, nombrando el modelo.

Por eso un modelo de visión en un provider personalizado necesita una línea. El formulario no tiene campo para ello; añade `input` al modelo en `$DSH_HOME/settings.yaml`:

```yaml
llm-pi-ai:
  providers:
    my-gateway:
      apiKeyEnv: GATEWAY_API_KEY
      api: openai-completions
      baseURL: https://gateway.example/v1
      models:
        - id: legacy-chat
        - id: vision-preview
          input: [text, image]
```

`input` acepta `text` e `image` y se aplica solo a ese modelo, de modo que una ruta puede servir ambos tipos. Omitirlo —o escribir una lista vacía, que significa lo mismo— conserva lo que el catálogo instalado registra para ese modelo y recurre al `defaultInput` de la ruta para un modelo que el catálogo no describe.

Si todos los modelos que introdujiste a mano aceptan imágenes, fija el respaldo una sola vez en la ruta en lugar de en cada uno:

```yaml
llm-pi-ai:
  providers:
    vision-gateway:
      apiKeyEnv: GATEWAY_API_KEY
      api: openai-completions
      baseURL: https://vision.example/v1
      defaultInput: [text, image]
      models:
        - id: first-model
        - id: second-model
```

`defaultInput` es un respaldo, no una anulación, y su valor por defecto es `[text]`: en un provider del catálogo solo responde por los modelos que el catálogo no describe, así que nunca elimina las imágenes de un modelo del catálogo que las tenga. Restringe uno de esos con el `input` propio de ese modelo. Un provider del catálogo no tiene lista `models` donde ponerlo, así que escríbelo en `modelOverrides`, con la clave por id de modelo:

```yaml
llm-pi-ai:
  providers:
    anthropic:
      modelOverrides:
        claude-sonnet-4-5:
          input: [text]
```

Toda lista debe nombrar al menos una modalidad, salvo la propia de un modelo, donde una lista vacía significa lo mismo que omitirla. Una modalidad desconocida se rechaza allí donde se escriba.

Ambos campos declaran una afirmación sobre tu endpoint en lugar de comprobarlo. Un modelo que declara imágenes que tu endpoint no sirve no se detecta aquí; en su lugar, el provider rechaza la solicitud.

### Compatibilidad de solicitudes

Un gateway puede tener una clave válida en una dirección accesible y aun así rechazar todas las solicitudes. pi-ai decide la forma de una solicitud —qué rol lleva el system prompt, qué campo limita la salida, cómo viaja un nivel de razonamiento— a partir de la URL del endpoint, y una dirección que no reconoce se trata como si fuera el propio OpenAI. La mayoría de los gateways compatibles con OpenAI rechazan al menos una cosa que OpenAI acepta.

Dos explican la mayor parte de los casos. A un modelo que declara razonamiento se le envía su system prompt como `role: "developer"`, algo que muchos gateways rechazan de plano, y el límite de salida se envía como `max_completion_tokens`, algo que un servidor que solo conoce `max_tokens` rechaza. El formulario no tiene campo para ninguno de los dos; corrígelos en la ruta de `$DSH_HOME/settings.yaml`:

```yaml
llm-pi-ai:
  providers:
    my-gateway:
      apiKeyEnv: GATEWAY_API_KEY
      api: openai-completions
      baseURL: https://gateway.example/v1
      compat:
        supportsDeveloperRole: false
        maxTokensField: max_tokens
      models:
        - id: my-model
```

El `compat` de una ruta es el valor por defecto de sus modelos, y el propio del modelo gana campo a campo, de modo que se puede corregir un modelo sin repetir la ruta:

```yaml
      models:
        - id: my-model
        - id: my-reasoner
          compat:
            thinkingFormat: deepseek
```

Lo que ninguno de los dos fija conserva el valor del catálogo instalado para ese modelo, y lo que el catálogo no describe recae en la detección de pi-ai. Dale un valor a cada conmutador que nombres: una clave dejada vacía (`supportsDeveloperRole:`) se rechaza en lugar de ignorarse, porque un valor vacío borraría lo que el catálogo sabe sin decir nada en su lugar. También se rechaza un nombre que ningún protocolo acepta, y el mensaje enumera los disponibles.

Cada conmutador pertenece a los protocolos que lo declaran, de modo que un conmutador válido en un `api` puede rechazarse en otro —el mensaje indica lo que ese protocolo sí ofrece—. Como `input` más arriba, un conmutador declara una afirmación sobre tu endpoint en lugar de comprobarla: fijar uno que tu gateway no necesita realmente solo envía una solicitud distinta.

Todos los conmutadores, sus valores aceptados y los protocolos que los admiten se enumeran bajo `PiAiCompatProfile` en la [referencia de configuración generada de `dsh-llm-pi-ai`](../../config-catalog.es.md#deepseek-aidsh-llm-pi-ai) —que se deriva del código fuente, así que no puede quedarse atrás de lo que el adaptador acepta—.

## Selecciona un modelo

Los providers configurados aparecen en el selector de modelos. Seleccionar un modelo también lo convierte en el valor por defecto de las sesiones nuevas. Una sesión que ya ha enviado una solicitud conserva el modelo registrado en su propio log.

Si un valor por defecto guardado nombra a un provider eliminado, el compositor muestra **Select model** y bloquea la entrada hasta que se seleccione otro modelo.

## Solución de problemas

- **`MISSING_CREDENTIAL`** — Guarda la clave del provider a través de la página de Modelos o proporciona la variable de entorno referenciada.
- **`UNKNOWN_MODEL`** — Selecciona un modelo configurado o añade el modelo que falta al provider personalizado.
- **Obtener los modelos disponibles devuelve 401** — Comprueba la clave. El descubrimiento de modelos llama al endpoint compatible con OpenAI `GET /models`; introduce los modelos manualmente en los endpoints que no lo ofrecen.
- **El gateway rechaza todas las solicitudes aunque la clave y la URL sean correctas** — La forma de su solicitud difiere de la de OpenAI. Empieza con `compat.supportsDeveloperRole: false` y `compat.maxTokensField: max_tokens` en la ruta.
- **Solo fallan los modelos de razonamiento** — pi-ai envía su system prompt como rol `developer`, algo que el gateway rechaza. Fija `compat.supportsDeveloperRole: false`.
- **Un conmutador de compat se rechaza por no tener valor** — Una clave escrita sin nada después de los dos puntos. Dale un valor, o elimina la clave para conservar el del catálogo instalado.
- **Una imagen se rechaza antes de enviarse** — El modelo no declara la modalidad de imagen. Dale a un modelo de provider personalizado `input: [text, image]`; la ruta chat-completions propia de DeepSeek es solo texto y no se puede configurar de otro modo.
- **El provider rechaza una solicitud que lleva una imagen** — El modelo declara imágenes que el endpoint no sirve realmente. Elimina `image` de la lista que se la concedió —el `input` del modelo o el `defaultInput` de la ruta— y luego inicia una sesión nueva: la imagen adjunta permanece en el log de la sesión, así que la misma solicitud se repite hasta que la sesión avance.

## Configuración avanzada

El [catálogo de configuración de plugins](../../config-catalog.es.md) generado enumera todos los campos y valores por defecto admitidos de cada plugin; [`dsh-llm-pi-ai`](../../config-catalog.es.md#deepseek-aidsh-llm-pi-ai) es la sección de provider que configura esta página. Las referencias [`dsh-llm-pi-ai`](../../../packages/llm/llm-pi-ai/README.es.md) y [`dsh-llm-deepseek`](../../../packages/llm/llm-deepseek/README.es.md) cubren la configuración directa de `settings.yaml`, la resolución del catálogo, los controles de razonamiento, las credenciales y los errores del adaptador.
