# @deepseek-ai/dsh-session-projection-cache

[English](README.md) | [中文](README.zh.md) | Español

La caché de proyección persistida (`ctx.sessionProjectionCache`): checkpoints duraderos del estado de cada unidad de proyección, un registro por sesión en la forma de datos de dominio (`session_projcache` — el backend json incluido lo deposita junto a `workspace.json` bajo la raíz de almacenamiento configurada). Autoridad de diseño: el [RFC de proyección de sesión](../../../.agents/notes/proposed/architecture/2026-07-27-session-projection-and-command-log.es.md) (sección de caché de proyección persistida).

Una fila almacenada `(key → {ver, seq, val})` es un atajo de pliegue, nunca una autoridad: posiblemente desactualizada (`seq` dice exactamente cuán desactualizada está) pero nunca incorrecta. Consecuencias a las que la implementación se compromete:

- **Cada escritura en segundo plano es fail-soft.** Una escritura duradera fallida registra una advertencia y mantiene la caché desactualizada; la siguiente escritura o lectura en frío se autocura. Un crash entre escrituras cuesta una reproducción de cola más larga, nunca un valor incorrecto.
- **Un desajuste de `ver` contra el `stateVersion` de la unidad en vivo descarta, nunca migra.** Una subida de versión de una unidad invalida sus filas en el momento de la lectura; la clave se vuelve a plegar desde el log.
- **Una fila debe pasar el `stateSchema` de la unidad en vivo.** Una fila malformada se omite de la vista sin E/S y restore la rechaza, de modo que la escalera de lectura en frío la vuelve a plegar desde el log.
- **Escrituras de registro completo.** Cada escritura reemplaza el checkpoint completo de la sesión (el corte del registro siempre es completo), instantáneado a través del límite JSON sin pérdida — un estado de unidad que viole el contrato plain-JSON falla ruidosamente.
- **Los registros están ligados a un ciclo de vida de log, no solo a un id.** Cada registro almacena la identidad de cabecera (`createdAt`, `cwd`) desde la que se plegó; cada lectura la valida (la cabecera en vivo o almacenada es el testigo) antes de aceptar una fila, de modo que un id borrado y recreado, o un almacén de persistencia sustituido bajo una caché superviviente, descartan el registro ajeno en lugar de sembrar valores fantasma.
- **El log lidera; la caché sigue.** Un checkpoint en vivo vuelca de forma duradera los eventos en búfer de la sesión ANTES de que aterrice la fila de caché, de modo que un crash puede dejar la caché detrás del log (una reproducción de cola más larga) pero nunca por delante.

## Política de escritura

Dos puntos obligatorios, regulados en medio:

| Disparador | Naturaleza |
|---|---|
| `turn/end` | Obligatorio — el valor final de turno es lo que quieren las lecturas en frío. |
| Disposición de sesión (detach) | Obligatorio — el momento de vivo-a-frío; después de él, la escalera en frío sirve esta sesión. |
| Eventos confirmados de `writeEveryEvents` | Regulación de configuración (recuento). |
| `writeIntervalMs` desde el primer evento sucio | Regulación de configuración (intervalo). |

Ambos campos de `Config` son obligatorios (sin valores por defecto): la cadencia de volcado es una decisión de despliegue sin un valor universalmente correcto, declarada en cordis.yml.

## Lectura de listado (`cachedSnapshot(meta)`)

El peldaño sin E/S: los valores de cliente vistos directamente desde el registro almacenado que coincide en identidad (solo claves que coinciden en versión y schema de estado), devueltos como un corte `{asOfSeq, values}` — `asOfSeq` es el watermark más bajo de las filas servidas, de modo que un cliente que siembre su almacén de valores por sesión bajo la regla gana-el-seq-más-alto nunca permitirá que una lista desactualizada bloquee la sobrescritura de una trama push más nueva. Las filas solo de host nunca se devuelven. `undefined` cuando no existe ninguna fila de cliente utilizable (id desconocido, ciclo de vida ajeno o ninguna fila utilizable); el carrier de listado del api-proxy convierte eso en una columna ausente.

## Lectura en frío (`coldSnapshot(id, signal?)`)

La escalera de lectura, con cero cargas de log completo en el camino feliz: filas en caché → `sessionProjections.restoreFloor` (anclado un evento por debajo del watermark utilizable más bajo) → `readFrom(id, floor)` de persistencia → `sessionProjections.restore` → reescritura fail-soft de las filas refrescadas. El ancla hace demostrable un log encogido (truncamiento por reparación de crash): una fila que se excede dispara exactamente una relectura completa desde el seq 0 en lugar de servir un valor fantasma. Sin unidades registradas se sirve `{asOfSeq: -1, values: {}}` sin tocar la persistencia; una sesión sin log persistido rechaza con el `not found` del seam.

`write(session)` es el checkpoint de corte síncrono que usan ambos puntos obligatorios; los carriers pueden llamarlo directamente (no es fail-soft — los envoltorios fail-soft son dueños del confinamiento).

## Composición

```yaml
- id: session-projection-cache
  name: '@deepseek-ai/dsh-session-projection-cache'
  config:
    writeEveryEvents: 200
    writeIntervalMs: 5000
```

Inyecta `storageDomain`, `sessionProjections`, `sessionPersistence`, `sessions`. Sin esta fila, el sistema de proyección funciona solo en vivo (caché de watermark; las lecturas en frío recurren a cargas completas de log allí donde un carrier las implemente).

## Experiencia del modelo

Ninguna, ya que la caché solo persiste y restaura modelos de lectura del lado del host del estado de sesión ya registrado y no toca ningún prompt, mensaje, schema, stream ni resultado de herramienta.

#### Efecto de KV Cache

Ninguno; la caché nunca ensambla ni envía solicitudes de provider.

## Limitaciones conocidas y trabajo diferido

- **Sin superficie de expulsión ni retención** — los registros se acumulan por sesión; podar los checkpoints almacenados es mantenimiento fuera de banda, la misma postura que la propia persistencia de sesión.
- **La regulación por intervalo es gruesa por sesión** — el temporizador se arma en el primer evento sucio tras una escritura limpia; un goteo constante por debajo del umbral escribe una vez por intervalo, no en ventana deslizante.
- **Las lecturas de `coldSnapshot` no se deduplican** — dos lecturas en frío concurrentes de una sesión ejecutan cada una la escalera; gana la última reescritura (las filas son equivalentes), aceptable para tasas de llamada a escala de listado.
