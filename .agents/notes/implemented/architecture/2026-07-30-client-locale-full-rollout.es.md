# Agent Note: Despliegue completo del copy del cliente sobre el seat de locale tipado, y la frontera de no traducción

Status: implemented

[English](2026-07-30-client-locale-full-rollout.md) | [中文](2026-07-30-client-locale-full-rollout.zh.md) | Español

## Problema

Después de aterrizar el seat estándar tipado de locale (`locale:` en register → `t` tipado inyectado por el framework), solo cuatro adoptantes tempranos lo montaron; todos los demás paquetes de cliente seguían distribuyendo literales codificados y en idiomas mezclados. Migrar el resto exigía mecanismos y decisiones de frontera que los adoptantes tempranos nunca tocaron: cómo se actualiza el texto en tiempo de registro (filas de nav, etiquetas de pestañas de vista) al cambiar de idioma; cómo reciben el copy los átomos ui-primitives con cero cordis; y qué cadenas permanecen deliberadamente sin traducir — una frontera no registrada invita a un agente futuro a «completar» la localización.

## Decisión

**El texto en tiempo de registro viaja en un thunk de etiqueta.** El `label` de un registro de lista acepta `SlotLabel = string | (() => string)`; los propietarios que proyectan filas del ledger resuelven a través de `resolveSlotLabel` (nunca leen `options.label` crudo) y hacen que el punto de lectura siga la revisión de locale (los outlets se suscriben ellos mismos a la revisión; las proyecciones fuera del ledger, como la nav de ui-settings, incorporan la revisión a su clave de caché y se suscriben a ambas fuentes). Los thunks se evalúan por lectura, por lo que un cambio de idioma causa cero agitación del ledger — sin re-registro, las versiones permanecen en su sitio, y se elimina todo el cableado de re-registro de `locale/change`.

**El copy de los componentes viaja en el seat estándar `t`; los hijos profundos reciben `t` como una prop simple** tipada `XxxProps['t']`. El canon del diccionario no cambia: `zh satisfies Record<string, string>` es la fuente de claves y `en satisfies Record<XxxKey, string>` fija el equilibrio bilingüe.

**Los átomos con cero cordis (ui-primitives) reciben el copy como props**: `copyLabel`/`copiedLabel` en `HoverCard`, `labels` en `TerminalBlock`/`JsonTree`, `copyLabel`/`copiedLabel` en `CodeBlock`, `codeLabels` en `MarkdownText`, `truncatedLabel` en `JsonBlock`, `label` en `ConnectionBanner`, `closeLabel` en `Modal` — los valores predeterminados son las cadenas codificadas anteriores, por lo que un consumidor que no pasa nada renderiza una salida byte-idéntica. Los plugins localizados pasan etiquetas guiadas por diccionario desde su propio seat `t`; los puntos de llamada que pasan props de objeto las memorizan sobre la identidad de `t` (`MarkdownText` guarda en caché su tabla de componentes sobre la identidad de `codeLabels`).

**La frontera de no traducción (decisiones deliberadas, no deuda):**

- **Las cadenas de error/fallo permanecen en inglés**: los respaldos escritos por el cliente (`command failed`, fallos de alternancia de plan), los mensajes de RpcError y los passthroughs wire `error.message (code)` se renderizan tal cual.
- **Los literales de diseño quedan fuera de los diccionarios**: los títulos de variante de la fila de herramientas (Think/Bash/…), las insignias de tipo estilo SYSTEM/USER, el wordmark del chip Plan, toda la StatsLine — idénticos en ambos idiomas.
- **ui-trajectory queda diferido por completo** (una superficie de inspección para desarrolladores, densa en terminología, regulada por separado).
- **El copy de arranque sigue codificado** (la página de arranque sin framework se ejecuta antes de que exista el servicio de locale).

**Las capas de derivación permanecen puras; la localización ocurre en el render.** El `relativeTime` de ui-workspace devuelve el `{unit, n}` estructurado, compuesto con plantillas de diccionario por el renderizador; las sesiones en blanco y el cubo Ungrouped conservan sus títulos almacenados, con el renderizador sustituyendo el copy localizado a partir del flag `blank` / el `workspaceId` ausente; **las filas en blanco se excluyen por completo de la búsqueda** (un título de visualización bilingüe no puede coincidir de forma estable con una consulta en un solo idioma). Las fechas no usan Intl: las plantillas de formato viven en los diccionarios (reloj de mensajes `clock.md`/`clock.ymd`, hover del workspace `date.ymd`) y los formateadores reciben `t` como parámetro, permaneciendo puros.

**Doctrina de pruebas y e2e**: `makeTranslate(...dicts)` (dsh-client-test-runtime) refleja la cadena de búsqueda del servicio (gana el primer diccionario, respaldo por clave, interpolación `{name}`); las specs de componentes hacen stub del seat `t` con ella, tipadas contra los seats reales de props. El e2e web abre uniformemente a través de `newEnglishPage` (un navegador `en-US`) y la instantánea de arranque compilado fija el mismo idioma de navigator — los goldens son inmunes a las migraciones de localización; el escenario de cambio de idioma de ajustes evita el helper y abre un navegador `zh-CN`, ya que el locale provisional sigue a `navigator` antes de que llegue una preferencia explícita de Host ([locale inicial derivado del navegador](../feature/2026-07-31-browser-derived-initial-locale.es.md)).

El mecanismo «la capa de aplicación se suscribe a `locale/change` y se re-registra para etiquetas frescas» de la [nota de capas de ajustes/locale/tema](../../proposed/architecture/2026-07-25-client-settings-locale-theme.es.md) queda superado por esta decisión (thunk + ciclo de vida de revisión).

## Alternativas consideradas

- **Conservar las etiquetas como cadenas y re-registrarse al cambiar** (la forma original de los adoptantes tempranos): el arranque ya registra una vez por paquete, y los listeners de `locale/change` que se re-registran se amplifican hasta convertirse en una tormenta; la agitación de versiones del ledger también invalida toda caché de proyección indexada por versión. Los thunks trasladan el costo de actualización a puntos de lectura que ya siguen la revisión.
- **Un canal de contexto/inyección de locale para ui-primitives**: rompe la frontera de cero cordis (los átomos dependerían del runtime) y arrastra a los consumidores no localizados (ui-trajectory). Las props permiten que cada consumidor decida de forma independiente.
- **Cadenas de error en los diccionarios**: la superficie de error es una superficie de depuración — el inglés verbatim es lo que se busca y se compara en los informes; los passthroughs wire no son traducibles de todos modos, y la traducción a medias fabrica texto en idiomas mezclados.
- **`toLocaleString()`/Intl para las fechas**: sigue el idioma del navegador/SO, no el locale de la app, lo que garantiza texto mezclado tras un cambio; las plantillas de diccionario son diminutas e isomorfas al reloj de mensajes.
- **Filas en blanco que coinciden con la búsqueda (contra títulos localizados o almacenados)**: cualquiera de las dos opciones produce «visible pero no encontrable» en un idioma; las filas de marcador de posición no llevan información, por lo que la exclusión de fila completa es la semántica estable.

## Consecuencias

- Un cambio de idioma actualiza toda la UI al instante con cero re-registro; adoptar un paquete nuevo son tres pasos (diccionario + declare-merge + `locale: NS`), sin pegamento escrito a mano.
- Costo: los consumidores de etiquetas de lista deben conocer `resolveSlotLabel` (una lectura cruda de `options.label` ahora puede contener una función); el tipo `SlotLabel` detecta estáticamente la mayor parte del uso indebido.
- Los valores predeterminados en chino de ui-primitives siguen renderizando chino bajo el locale inglés **hasta que un consumidor pasa etiquetas** — el consumidor no migrado de JsonTree (ui-trajectory), que muestra sus valores predeterminados en inglés, coincide por casualidad con el statu quo totalmente en inglés de ese paquete.
- Fijar el e2e al inglés significa que la superficie de copy zh queda cubierta principalmente por las specs de componentes a nivel de paquete y el escenario de cambio de idioma de ajustes; el e2e de navegador ya no comprueba el copy zh. El locale inicial/respaldo (un navegador que no nombra ningún idioma publicado, o una ejecución sin navegador) es `en`, no `zh` — véase [locale inicial derivado del navegador](../feature/2026-07-31-browser-derived-initial-locale.es.md).
