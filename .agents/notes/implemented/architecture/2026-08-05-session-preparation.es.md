# Agent Note: Preparación de Session reutilizable antes de la publicación

Status: implemented

[English](2026-08-05-session-preparation.md) | Español

## Problema

La inspección en frío del historial y la reanudación del Agent (agente) materializaban por separado el mismo registro de sesión persistido. Para un registro comprimido grande, cada operación repetía la lectura completa, la descompresión, el análisis, la validación, la congelación y la construcción de Session. La paginación podía por tanto volver a pagar el coste de la lectura en frío, mientras que hacer que una consulta de historial activara un Agent acoplaría un ciclo de vida de lectura a un Agent en vivo sin un punto natural de retirada.

La creación nueva y la reanudación persistida llegaban también a la misma frontera de publicación a través de flujos de construcción diferentes. Esto ocultaba el invariante de que la configuración debe terminar sobre una única Session sin publicar antes de que esa Session exacta y su Agent se vuelvan visibles juntos.

## Decisión

`SessionPreparation` es propietaria de una única `Session` exacta sin publicar hasta la publicación o el rollback. Es un objeto del ciclo de vida de Session, no un objeto del ciclo de vida de Agent ni de activación. La creación nueva envuelve el resultado de `SessionStore.prepare()`; la reanudación persistida obtiene una preparación de `SessionPersistence.prepare()`.

El Agent loop consume ambas formas a través de un único pipeline de configuración y publicación: adquiere la preparación, construye el contexto privado del Agent en torno a `preparation.session`, espera la configuración opcional, publica esa Session y ese Agent exactos y libera la preparación en cada salida. La publicación transfiere el ciclo de vida en vivo a los almacenes de Session y Agent existentes; `SessionPreparation` en sí no posee ningún comportamiento de Agent.

Esto refina la frontera de publicación de la [decisión de ciclo de vida y propiedad del Agent](2026-06-18-agent-lifecycle-and-ownership-contracts.es.md) sin reemplazar su modelo de propiedad.

## Ciclo de vida de la preparación persistida

Una implementación de persistencia respaldada por un coordinador carga una fuente en frío en una Session preparada. El backend transfiere metadatos y eventos nuevos y sin alias mutuos junto con la revisión calificada por la fuente que identifica esos valores exactos; la ruta de restauración de Session valida y congela los grafos in situ en lugar de clonarlos. El coordinador calcula los closers de turnos interrumpidos y construye una única vez la Session exacta sin publicar. Su cabecera inmutable y su registro de eventos lógico equilibrado forman la `SessionInspection` que toman en préstamo los lectores, mientras que la revisión permanece interna a la persistencia.

`inspect(id, signal?)` no muta el almacenamiento. Los closers sintéticos existen solo en la vista preparada en memoria, y una cola física interrumpida permanece intacta. Los llamadores del mismo id comparten una lectura en frío en curso. Una vez lista, la preparación puede permanecer en una LRU por coordinador cuya capacidad por defecto es de cinco y que los backends de primera parte pueden configurar. Antes de reutilizar una fuente retenida, el coordinador lee la revisión actual de ese id; una discrepancia desaloja la fuente lista y repite la materialización en frío. Una fuente que ya se está confirmando o reservada para la reanudación sigue siendo de propiedad exclusiva, por lo que la inspección concurrente toma prestada esa vista inmutable hasta la publicación o la liberación.

`prepare(id, signal?)` reserva en exclusiva la Session preparada. Confirma la revisión retenida antes de confirmar cualquier reparación de cola interrumpida y de turnos interrumpidos, establece el cursor durable y devuelve después una preparación desechable. Una fuente obsoleta se descarta y se vuelve a cargar en lugar de repararse o publicarse. Una reparación con éxito descarta también la fuente previa a la reparación y vuelve a materializar el registro confirmado antes de la reserva, de modo que una revisión más nueva nunca se asocia a un grafo de eventos más antiguo. Otra preparación del mismo id espera hasta que la reserva se publique o se libere. La publicación acepta solo la Session exacta reservada y fija el cursor confirmado sin reconstruir su historial. Una configuración fallida o una cancelación devuelve a la LRU una Session sin publicar sin cambios; la mutación o la fijación consume la reserva.

La API heredada `load(id)` usa la misma maquinaria de preparación y reparación, descarta después su reserva y devuelve la vista lógica inmutable. Sigue siendo una API de compatibilidad, no la ruta de reutilización de historial a reanudación. Este ciclo de vida extiende el [coordinador de persistencia compartido](2026-06-18-shared-persistence-write-coordinator.es.md) y preserva las reglas de almacenamiento y recuperación que pertenecen a la [decisión de persistencia de sesión](2026-06-14-session-persistence.es.md).

## Reutilización de historial y reanudación

Las lecturas del historial usan `inspect()`, de modo que las páginas repetidas toman prestado el mismo estado preparado inmutable sin activar un Agent. Una reanudación posterior usa `prepare()` y recibe la Session exacta retenida por la inspección; no vuelve a leer, descomprimir, analizar, clonar, validar ni congelar el registro completo.

Si el registro durable cambia después de la inspección, su revisión cambia. La siguiente lectura del historial o reanudación descarta una Session lista retenida y materializa el registro nuevo, de modo que un grafo de eventos antiguo no puede asociarse a una revisión de instantánea más nueva. Una fuente ya reclamada por una reanudación en curso no se desaloja: su propietario exclusivo la conserva hasta la publicación o la liberación, y el historial concurrente puede tomar prestada la misma vista inmutable.

El acceso en frío a subagentes (subagent) continuables sigue el mismo camino. La autorización de descriptor inspecciona primero al hijo y luego `ctx.agents.resume()` reserva y publica la Session retenida. Esto preserva las reglas de ciclo de vida y autorización de la [decisión de conversaciones de subagente continuable](../feature/2026-07-28-continuable-subagent-conversations.es.md) y elimina su lectura en frío duplicada.

## Fronteras

- `readFrom()` sigue siendo una API de sufijo físico desacoplada. No crea ni consume una preparación, no sintetiza closers lógicos ni se incorpora a la LRU.
- La adopción de HMR (hot module replacement) mantiene la Session en vivo como autoridad y lee directamente el prefijo almacenado. Puede truncar un fragmento físico interrumpido, pero nunca cierra como interrumpido el turno abierto en vivo.
- La caché pertenece a un único coordinador de persistencia, no a un mapa de Session global al proceso. Las Sessions en vivo pertenecen a los almacenes existentes y nunca ocupan capacidad de preparación.
- Una creación nueva nunca reclama una preparación persistida en frío con el mismo id. Las colisiones de persistencia siguen rechazándose.
- Las implementaciones de persistencia de terceros conservan el fallback abstracto de `prepare()` a través de `load()`. Reciben la misma interfaz de publicación, pero solo obtienen la reutilización de objetos exactos cuando anulan la preparación.
- La validación de revisiones establece la frescura en los puntos de reutilización y de confirmación de reparación; no añade exclusión de escritores entre procesos a un backend. Los reintentos convergen cuando el registro durable permanece sin cambios durante un ciclo de ida y vuelta de lectura/comprobación, de modo que los escritores externos continuos pueden retrasar la preparación.

## Verificación

El contrato de persistencia compartido fija la inspección en frío equilibrada y no mutante y la reparación posterior. `persistence.spec.ts` y `preparations.spec.ts` fijan el uso compartido en curso del mismo id, la reutilización exacta de Session entre inspect y prepare, la actualización disparada por revisión antes del historial y de la reanudación, la confirmación única de reparación, la reserva exclusiva, la liberación tras una configuración fallida, el desalojo de entradas listas de la LRU, el rechazo de append durante la reserva y la publicación únicamente de la Session reservada. Las pruebas de backend fijan que las lecturas completas y ligeras usan la misma identidad de revisión. Las pruebas del Agent loop y de subagente continuable fijan el pipeline de publicación común y el camino de inspección a reanudación a través de la cancelación y el teardown.

## Alternativas consideradas

**Activar un Agent para las lecturas del historial.** Rechazada porque la paginación mantendría en vivo a los Agents de solo consulta y trasladaría la retirada de la caché al ciclo de vida del Agent.

**Almacenar en caché solo `{ meta, events }`.** Rechazada porque la reanudación seguiría reconstruyendo, validando, congelando y copiando una Session a partir de los valores en caché. La Session exacta sin publicar es la unidad reutilizable.

**Mantener un mapa de Session global al proceso.** Rechazada porque cruzaría las fronteras de propiedad de backend y runtime, retendría identidades sin límite y duplicaría el almacén de Session en vivo.

**Añadir una transacción de restauración o un coordinador al Agent loop.** Rechazada porque la lectura en frío, la reparación, la reserva y la fijación del cursor son responsabilidades de persistencia y de Session. El Agent loop solo necesita la frontera de propiedad uniforme `SessionPreparation`.

**Convertir `readFrom()` en preparación lógica.** Rechazada porque los consumidores de watermark necesitan un sufijo físico desacoplado y, en backends con capacidad de búsqueda (seek), una lectura acotada. El equilibrio de recuperación y la reutilización de la Session completa tienen semánticas diferentes.

## Consecuencias

Una sola materialización en frío puede servir a la paginación del historial, a la inspección de descriptores de subagentes y a una reanudación posterior. La transferencia de propiedad elimina los clones de restauración redundantes, mientras que la LRU acotada por coordinador limita la memoria y evita crear Agents en vivo para las consultas. La creación y la reanudación comparten un único protocolo de publicación sin fusionar las responsabilidades de Agent y Session.

La primera inspección en frío asume ahora el coste completo de validación y construcción de Session y puede retener esa Session sin publicar hasta su desalojo. La persistencia debe coordinar la reserva, el append, la reparación y la publicación, y los llamadores deben tratar los valores de inspección como estado prestado inmutable. Los backends que dependen de `prepare()` por defecto siguen siendo correctos, pero no reciben la optimización de reutilización.
