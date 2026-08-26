# @deepseek-ai/dsh-goal

[English](README.md) | Español

Estado de objetivo de misma sesión basado en eventos (event-sourced). El servicio conserva un objetivo de finalización actual en la sesión existente de un agent mientras mantiene el permiso para continuar como activación local del proceso. La [Agent Note del dominio de objetivo](../../../.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.es.md) es la dueña del razonamiento de diseño; el [catálogo de tipos de objetivo](../../../docs/subsystems/goal.es.md) registra las formas de datos literales.

## Configuración

```yaml
- id: goal
  name: '@deepseek-ai/dsh-goal'
  config:
    defaultMaxGoalRounds: 256
```

`defaultMaxGoalRounds` debe ser un entero seguro positivo. `create()` materializa este valor por defecto de despliegue internamente antes de confirmar un objetivo; un valor a nivel de petición lo anula.

## Contrato del servicio

`ctx.goals` solo acepta la instancia `Agent` en vivo exacta registrada bajo su id. `get()` devuelve una `GoalView` desacoplada; las mutaciones usan una barrera de comparar-y-establecer `GoalRef { id, revision }` y rechazan las refs obsoletas. El servicio expone los verbos create, edit, pause, resume, complete, block y clear a través de la región generada de [goal.md](../../../docs/subsystems/goal.es.md#cordis-surface). La resolución del valor por defecto de creación es interna. `disarm()` es la única excepción de solo ciclo de vida: elimina la autoridad de continuación local del proceso sin escribir una revisión ni emitir una mutación.

Como máximo un objetivo es el actual. La creación produce un objetivo activo de revisión uno y lo arma. Un objetivo no completado debe editarse, transicionarse o limpiarse; un objetivo completado puede reemplazarse por un id globalmente nuevo. Las ediciones conservan la fase, el motivo del bloqueador y la activación. La pausa, la finalización, el bloqueo y el clear desarman la activación. Un bloque registra un código en kebab-case en minúsculas de propiedad de la política más una explicación libre normalizada; los límites de provider, los presupuestos configurados, los errores de ejecución y las peticiones de entrada humana usan todos esta única fase duradera en lugar de multiplicar los estados del ciclo de vida. Resume acepta una fase detenida o un objetivo activo desarmado solo mientras el tope de rondas configurado tiene capacidad restante; limpia cualquier motivo de bloqueador anterior. Un objetivo activo y armado rechaza la operación redundante.

Cada mutación añade un evento duradero `goal/change` que lleva la instantánea completa posterior a la mutación; clear usa una tumba con revisión. Por tanto, el estado de objetivo no depende de la colocación, la reclamación, la admisión ni el descarte en la bandeja de entrada. El log de sesión es la única autoridad duradera.

La reproducción estricta deriva las mutaciones del ciclo de vida solo de `goal/change` y rechaza formas malformadas, revisiones discontinuas, transiciones de ciclo de vida ilegales, marcas de tiempo no monótonas por objetivo y rondas de objetivo admitidas no secuenciales. Las rondas positivas solo avanzan con eventos `user/message` admitidos originados por el objetivo. Las marcas de tiempo de mutación se recortan contra la actualización de objetivo precedente cuando el tiempo de pared retrocede. La reproducción incremental conserva su cursor en el primer evento corrupto, y `goal/changed` se dispara después de que el evento duradero se confirme, con los fallos de listener contenidos.

La activación nunca se persiste. Una cache nueva y cada borde `agent/session-start` la desarman incluso cuando la reproducción encuentra una fase duradera activa. Un driver de continuación también llama a `disarm()` antes de la descarga o tras una incertidumbre de durabilidad. Por tanto, el resume y el fork de sesión y el reemplazo de driver conservan el objetivo, la fase, las revisiones y el recuento de rondas admitidas sin iniciar trabajo; una mutación de resume explícita posterior debe armar la continuación.

El acompañante `./invariant`, publicado por separado, mantiene un plegado independiente de cada sesión adjunta. Rechaza los cambios de objetivo malformados, las revisiones discontinuas, las transiciones de ciclo de vida ilegales, las regresiones de marca de tiempo y las rondas admitidas no secuenciales antes de que el evento candidato entre en el log duradero.

## Puntos de extensión

Los plugins de política llaman a los verbos del servicio y reaccionan al evento con ámbito `goal/changed`. Un Consumer de continuación admite rondas como eventos `user/message` con `GoalMessageSource`; los turnos humanos ordinarios nunca incrementan `roundsStarted`. Los Consumers usan la interfaz `Agent` y los eventos en lugar de importar `dsh-agent-loop`.

## Experiencia de modelo

### Mutación de estado de objetivo

#### Lo que ve el modelo

Las mutaciones de objetivo no inyectan contexto al modelo. Herramientas como `get_goal` devuelven el estado actual, y un Consumer de continuación puede mostrar el objetivo y el estado de ronda cuando programa trabajo del modelo. Un contexto de objetivo siempre visible futuro pertenece a un plugin de contexto independiente, no a la ruta de persistencia.

#### Efecto en tokens

Los eventos de mutación de objetivo no añaden tokens de modelo por sí mismos. Los resultados de herramienta y los prompts de continuación programados contabilizan su propio estado visible.

#### Efecto en la KV cache

No hay efecto en la KV cache hasta que otro componente expone el estado de objetivo en la entrada visible para el modelo.

## Limitaciones conocidas y trabajo pendiente

- **Estado, no planificación** — este paquete no decide cuándo continúa un objetivo armado, no reintenta fallos anómalos ni cancela un turno activo; esas políticas pertenecen a los Consumers del seam de agent.
- **Solo presupuesto de recuento de rondas** — `maxGoalRounds` no mide tokens, moneda, tiempo de pared ni cuotas de provider.
- **Sin evaluador independiente** — el llamante que registra la finalización o el bloqueo es autoritativo; la certificación respaldada por un evaluador queda diferida a una capa de política independiente.
- **Un solo objetivo actual** — los objetivos paralelos y una base de datos de objetivos independiente están ausentes a propósito; el historial sigue disponible en el log de sesión después del reemplazo o del clear.
- **Productores de confianza en el proceso** — un plugin con acceso directo a `Session` puede añadir datos falsificados de `goal/change`. La reproducción estricta detecta los registros malformados o incoherentes y deja el acceso a objetivo en fallo en ese registro hasta que se repara el log; esto es detección de integridad, no aislamiento de plugins.
