# Ejemplos destilados de prosa

[English](examples.md) | Español

Usa estos ejemplos para identificar el principio rector, no como plantillas de texto.
“Balanced” preserva cada proposición que soporta carga con la menor explicación necesaria en esa ubicación.

## Conserva cada cláusula factual

**Original:** “The coordinator carefully serializes writes per session, flushes buffered events before disposal resolves, and reports backend failures to the caller.”

**Recortado en exceso:** “The coordinator serializes persistence.”

**Balanced:** “The coordinator serializes writes per session, flushes buffered events before disposal resolves, and reports backend failures to the caller.”

Elimina decoración y repetición, no proposiciones.
El actor, el alcance por sesión, el orden de disposal y la visibilidad de fallos son hechos separados.

## El alcance explícito del skill es funcional

**Recortado en exceso:** “Read the sources and use judgment.”

**Balanced:** “This skill is guidance, not a complete checklist. Use judgment beyond the named checks; documented requirements still apply.”

**Demasiado detallado:** varios párrafos defendiendo por qué las listas no pueden sustituir el razonamiento independiente.

Conserva la limitación explícita porque cambia cómo un agent aplica el flujo.
Recorta persuasión repetida, no el guardarraíl.

## Un cookbook conserva acción y verificación

**Recortado en exceso:** “Add tests for the tool.”

**Balanced:** “Test registration and disposal at unit level, exercise the tool through the real loader path, and add a snapshot when its rendered output changes. Verify the assertion observes the external result rather than the model's report.”

**Demasiado detallado:** un walkthrough de cada archivo de fixture y assertion que ya se ve en el código de ejemplo.

Conserva los tiers de pruebas, la acción requerida, la ruta real de entrada y la verificación observable.
Elimina la narración de fixtures.

## Conserva propiedad y timing

**Recortado en exceso:** “Provider work is cancelled during teardown.”

**Balanced:** “The runtime requests provider cancellation before releasing the child scope; the provider remains responsible for joining its workers before disposal resolves.”

**Demasiado detallado:** un relato cronológico de cada promise y callback usados para implementar el teardown.

El actor, el orden, el punto donde cambia la propiedad y la garantía de finalización son cláusulas factuales separadas.

## El JSDoc de eventos conserva el timing del límite

**Recortado en exceso:** “Composes and caches the session prefix.”

**Balanced:** “Composes the session prefix once before the first pre-step and model request. Listener appends join the current request, and pre-step pressure accounting receives the composed prefix.”

**Demasiado detallado:** un walkthrough de los helpers del loop, los campos de caché y los callbacks de promises que implementan el orden.

El orden del evento y su consecuencia sobre la solicitud actual son comportamiento visible para el caller, no narración de implementación.

## Orienta código complicado sin narrarlo

**Recortado en exceso:** “Worker realm support.”

**Balanced:** “Owns the worker realm and its host bridge. Realm initialization is single-shot; disposal terminates the worker and rejects later calls. See the worker-isolation Agent Note for the protocol rationale.”

**Demasiado detallado:** un avance párrafo por párrafo de las clases y funciones helper de más abajo.

Conserva el rol del módulo, las dependencias, las responsabilidades y el comportamiento no obvio del ciclo de vida.
Enlaza la justificación de arquitectura y deja que el código muestre el flujo de control local.

## El JSDoc público incluye fallos

**Recortado en exceso:** “Returns the realm global.”

**Balanced:** “Returns the initialized realm global. Throws if initialization has not completed or the realm has already been disposed.”

**Demasiado detallado:** las ramas internas de la state machine y las llamadas helper exactas que llevan a cada throw.

Los throws y las precondiciones de estado son hechos del contrato visibles para el caller.

## Conserva un mapeo de implementación conciso

**Recortado en exceso:** “Search provider backed by an external API.”

**Balanced:** “Maps each provider result to the shared search-result fields, preserving the title, URL, and text while omitting provider-only ranking metadata.”

**Demasiado detallado:** una reformulación campo por campo del código de mapping, incluidos campos con nombres idénticos y asignaciones obvias.

Conserva los detalles del mapping que explican dónde un adaptador elimina o cambia información.

## Enlaza la justificación conservando el contrato local

**Recortado en exceso:** “Disposal is documented in the lifecycle Agent Note.”

**Balanced:** “Disposal aborts the run and waits for provider quiescence. See the lifecycle Agent Note for ownership and race handling.”

**Demasiado detallado:** repetir junto a cada disposer la coreografía de promises y los modelos de propiedad rechazados de la Agent Note.

Conserva el comportamiento y la garantía de finalización donde los callers los necesitan.
Enlaza agresivamente para el algoritmo y la justificación; un enlace no puede sustituir el contrato local.

## Las Agent Notes implementadas conservan contratos de verificación

**Recortado en exceso:** borrar toda la sección Testing porque la Agent Note ya se envió.

**Balanced:** “Unit tests cover cancellation before and after publication, disposal quiescence, and provider reload. A built-entry smoke covers the real loader path; snapshot coverage is deferred because the transport is process-specific.”

**Demasiado detallado:** un walkthrough archivo por archivo de fixtures y assertions sin distinción adicional de comportamiento.

Elimina tareas de migración y narración de tests.
Conserva los tiers, los comportamientos que fijan, la ruta real de entrada y las lagunas de cobertura con nombre.

## Un límite de seguridad puede necesitar un ejemplo concreto

**Recortado en exceso:** “Mounted plugins share the host's authority.”

**Balanced:** “Mounted plugins share the host's authority; for example, access to `ctx.shell` permits commands with the host executor's privileges.”

**Demasiado detallado:** una lista de cada servicio del que un plugin podría abusar y cada exploit hipotético.

Conserva un ejemplo cuando haga operativamente claro un límite de seguridad que, de otro modo, sería abstracto.

## Elimina por completo los reasoning transcripts

**Demasiado detallado:** “First the loop checks whether the value is absent. If it is absent, the next branch returns early. Otherwise it continues, which is why the final assertion is safe.”

**Balanced:** ningún comentario cuando el código ya expresa esas ramas.
Si el early return protege un invariant no obvio, declara solo ese invariant.

No comprimas un reasoning transcript en una narración más corta; elimínalo.

## Los comentarios de configuración explican lo que el árbol no puede

**Demasiado detallado:** “This entry loads the local filesystem provider, followed by the policy plugin, followed by the read, write, and edit tools,” cuando las entradas adyacentes ya muestran ese orden.

**Balanced:** “Load policy before the model-facing tools so their write and edit calls pass through the read-before-mutation gate.”

Conserva la consecuencia del orden, una regla de alcance sorprendente o un límite de seguridad.
Deja que la configuración muestre su propio inventario.

## No recortes solo por el recuento de palabras

**Actual:** “The adapter converts provider errors into the shared error type so callers can handle authentication, rate-limit, and transient failures uniformly.”

**Más corto pero peor:** “The adapter normalizes provider errors.”

**Decisión balanced:** conserva la frase actual salvo que un enlace o el contrato circundante ya enumeren las categorías de fallos.
La versión más corta pierde la consecuencia y las distinciones sin mejorar la estructura.

## El texto visible para el modelo sigue a su propietario

**Recortado en exceso:** “The tool returns errors when a call fails.”

**Demasiado detallado:** copiar en el README de este backend las cadenas de schema y renderer de otro paquete.

**Balanced:** cita el texto estable de prompt, resultado y error cuyo propietario sea este paquete.
Enlaza el catálogo de herramientas generado para schemas y el README del consumer para texto cuyo propietario sea otro paquete; declara localmente solo las condiciones o deltas de este paquete.

El redactado que llega a un modelo es comportamiento, pero la duplicación también deriva.
La exactitud pertenece al propietario.

## Los resúmenes generados deben sostenerse solos

**Recortado en exceso:** “Approval request and policy service.”
El propietario explica el orden de política y el logging de auditoría después, pero el catálogo exporta solo su primera frase.

**Demasiado detallado:** mover el ciclo de vida completo del servicio y el comportamiento de prompt-notice a la frase extraída.

**Balanced:** “Approval service that applies session policy before answerers and logs every ask/outcome pair to the requesting session.”
Conserva el detalle no propio de catálogo en frases posteriores.

Debes saber qué extrae el generador.
Ese fragmento tiene que preservar el contrato necesario en su salida generada.

## Las limitaciones son contratos, no inventarios de deuda

**Recortado en exceso:** omitir una caché de vida-de-proceso que hace que los cambios de configuración requieran recargar el plugin.

**Demasiado detallado:** listar limpieza de helpers privados y accessors solo de test sin consecuencia para callers o maintainers.

**Balanced:** “Provider selection is cached for the plugin lifetime; installing or repairing a provider requires reload.”
Deja la limpieza ordinaria en su TODO o Agent Note.

Conserva lagunas y restricciones no obvias que afecten el uso o el mantenimiento seguro.
Un README de paquete no es un volcado de backlog.
