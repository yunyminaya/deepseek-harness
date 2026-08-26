# Reglas de traducción

English | [中文](translation-rules.zh.md) | Español

Cómo traducir la documentación de este repositorio al español. Tanto el inglés como el chino son fuentes de autoridad equivalentes ([README.md](../README.md)); estas reglas rigen la producción del par español. Se aplican por igual a humanos y agentes. Los niveles de obligatoriedad siguen el uso RFC 2119: **MUST** / **MUST NOT** bloquean la revisión si se incumplen; **SHOULD** exige una razón declarada para desviarse; **MAY** es discrecional.

## Fidelidad

- El texto en español *MUST* decir lo mismo que el lado original: sin comportamiento añadido, sin requisitos, advertencias, afirmaciones de versión ni ejemplos nuevos, y sin omitir ninguno de los existentes.
- El texto en español *SHOULD* leerse como redacción técnica natural en español, no como glosa palabra por palabra. Traduce el significado, reestructura las frases donde la gramática del español lo pida y conserva el registro del autor: lo conciso sigue siendo conciso.
- No traduzcas lo intraducible: si una frase solo funciona por un giro idiomático del idioma fuente, traduce la idea, no el giro.

## Voz

- Escribe como un autor técnico nativo que reformula el contenido, no como un traductor que transpone frases, conservando todas las cláusulas de la fuente: nada añadido, nada omitido; la fluidez nunca justifica perder una cláusula.
- Da a las oraciones un sujeto explícito cuando el español lo requiera; evita los pasivos vagos.
- Prefiere la jerga técnica española consolidada sobre los calcos (p. ej. `despliegue` por deployment, `instantánea` por snapshot); localiza las metáforas en lugar de transplantarlas.
- Divide los párrafos largos por unidad semántica — una idea por párrafo. Los límites de párrafo *MAY* diferir de la fuente; la firma estructural no cuenta los párrafos.
- Nombres de categoría: usa el nombre convencional en español (recetario, cookbook) con anotación en inglés en la primera mención cuando el término inglés sea el habitual en el sector. Las referencias literales a directorios o archivos permanecen en inglés con formato de código.

## Preservación de la estructura

El archivo traducido *MUST* corresponder uno a uno con el original en:

- jerarquía de encabezados (mismos niveles, mismo orden — el TEXTO del encabezado se traduce),
- forma y numeración de listas,
- tablas (mismas columnas, mismo orden de filas; las celdas de cabecera se traducen según la terminología),
- bloques de código delimitados — **idénticos byte a byte, incluidos los comentarios**,
- tramos de código en línea (comandos, banderas, claves de configuración, rutas de archivo, nombres de eventos, nombres de API, números de versión) — verbatim, nunca traducidos ni reformateados,
- enlaces y anclas: cada enlace relativo *MUST* mantener el mismo destino semántico y el mismo sufijo de consulta/fragmento exacto. Cuando el destino pertenece al corpus bilingüe, el lado inglés usa su ruta `.md`, el lado chino su ruta `.zh.md` y el lado español su ruta `.es.md`; un destino fuera del corpus conserva la ruta original. Quedan excluidos de la reescritura a `.es.md`: `LICENSE.md`, `THIRD_PARTY_NOTICES.md`, `CLAUDE.md`, cualquier archivo de código (`.ts`, `.py`, `.json`, `.yaml`, …), `docs/i18n/style-samples.md` (fuera del corpus) y los destinos bajo `.agents/notes/archived/`. Las URLs externas, las imágenes y los fragmentos intra-página puros permanecen sin cambios. El TEXTO del enlace se traduce.
- conmutador de idioma: inmediatamente después del H1, la versión española lleva la línea `[English](foo.md) | Español` (con la ruta relativa correcta al `.md` inglés). Si el archivo ya la tiene, no se duplica.

Se aplican al español las mismas convenciones Markdown del repositorio: una línea física por párrafo (`verify-md-wrap`), resolución de enlaces relativos (`verify-md-links`), exactamente un salto de línea final.

## Nomenclatura del archivo traducido

- El nombre del archivo de salida es EXACTAMENTE `F.es.md`, donde `F` es el nombre base del archivo fuente sin la extensión: para `foo.md` → `foo.es.md`, para `README.md` → `README.es.md`. NUNCA se escribe `foo.md.es.md` ni `foo.es.md.es.md`; si por error se creó un archivo con ese patrón, renómbralo.

## Archivos grandes

- Un archivo fuente de más de ~25 KB se traduce por secciones para no desbordar el contexto: se lee con `read_file(path, offset, limit)` en trozos de ~25 KB (avanzando con los offsets de continuación) y cada trozo traducido se escribe en el `.es.md` — el primero con `write_file`, los siguientes AÑADIENDO con: `python3 -c "import sys;open('RUTA.es.md','a').write(sys.stdin.read())" <<'EOF' ... EOF`. Al terminar se verifica que fuente y traducción tienen el mismo número de bloques de código (`grep -c '^```'` en ambos).

## Terminología

- [terminology.es.md](terminology.es.md) es la fuente de verdad para el español. Antes de traducir, cárgala; cada término listado *MUST* seguir su fila y sus prohibiciones de «no traducir como».
- Un término técnico no listado *MAY* usar una traducción consolidada de una fuente importante en español (documentación en español de K8s/Vue/MDN, guías de estilo de Microsoft, documentación de grandes proyectos). Sin ese precedente, *MUST* permanecer en inglés y anotarse como término pendiente.
- Ninguna dirección puede inventar una traducción sobre la marcha; un término decidido entra en [terminology.es.md](terminology.es.md) en el mismo cambio o en uno posterior.

## Tipografía

- El español de este repositorio sigue las convenciones habituales de la redacción técnica en español: puntuación española (`¿`, `¡`, `—`), comillas españolas («») en prosa, y las reglas estándar de mayúsculas (solo inicial mayúscula en títulos, salvo nombres propios).
- Los nombres propios conservan su mayúscula canónica: GitHub, TypeScript, DeepSeek — nunca `github`/`Github` salvo al citar código.
- La segunda persona es tú, no usted (voz directa del repositorio).
- Los marcadores de énfasis (`**negrita**`, `*cursiva*`) permanecen en los mismos tramos que la fuente.
- Los números usan coma decimal y punto de miles según el español solo si la fuente usa cifras con formato localizable; en caso de duda, conserva la forma exacta de la fuente.

## Nivel de calidad

- Un par está terminado cuando un ingeniero hispanohablante que lee cualquiera de los dos archivos obtiene todo lo que obtiene un lector del otro — los mismos datos, las mismas salvedades, el mismo tono — y nada más.

## Referencias

Autoridades citadas por estas reglas:

- [Microsoft Spanish style guide](https://learn.microsoft.com/en-us/globalization/reference/microsoft-style-guides) — convenciones tipográficas y de localización.
- [FundéuRAE](https://www.fundeu.es) — criterio normativo para neologismos técnicos.
- [Kubernetes es docs](https://kubernetes.io/es/docs/contribute/localization_es/) — práctica de localización técnica en español.
