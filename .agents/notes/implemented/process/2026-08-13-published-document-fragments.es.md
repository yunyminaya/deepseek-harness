# Agent Note: Validar fragmentos de documentos publicados

Status: implemented

[English](2026-08-13-published-document-fragments.md) | [中文](2026-08-13-published-document-fragments.zh.md) | Español

## Problema

`verify-md-links` valida fragmentos con los ids de encabezado Markdown de GitHub, mientras que el sitio web de documentación renderiza los encabezados con VitePress. Los encabezados cargados de puntuación y los encabezados traducidos pueden por tanto pasar la validación de fuente pero producir enlaces a ids ausentes del HTML publicado. Un build exitoso de VitePress valida las páginas de destino, no los ids de fragmento.

## Decisión

`docs:build` y su variante MPA corren `verify-doc-site-fragments` después de que VitePress emita `website/.dist`. El verificador procesa cada página HTML emitida, resuelve cada enlace interno de fragmento contra las clean URLs de VitePress, y falla cuando la salida está ausente, las rutas son ambiguas, un href está malformado, o falta la página de destino o el id solicitado. Las pruebas unitarias cubren esos fallos más las clean URLs, los alias `.html`, los enlaces dentro de la misma página, los ids codificados y literales, y la exclusión de enlaces externos.

Cualquier encabezado de destino de fragmento cuyo id GitHub difiera de su id VitePress lleva un alias explícito compatible con GitHub. Las páginas inglesas autoradas y las traducidas colocan el alias antes del encabezado; las páginas traducidas usan el id inglés compartido por el par bilingüe. Los catálogos generados de config, tools y persistencia emiten el alias desde su generador propietario. La validación del Markdown fuente permanece independiente y sigue rechazando enlaces que no resuelvan bajo el renderizado del repositorio.

La ruta del archivo sigue el locale del documento fuente cuando existe un destino emparejado, como define la [decisión de enlaces bilingües localizados](2026-08-18-localized-bilingual-links.es.md); el sufijo de query y fragmento permanece idéntico en el par.

## Alternativas consideradas

**Usar fragmentos específicos del locale.** Los pares bilingües preservan intencionadamente un sufijo de fragmento idéntico aunque la ruta del archivo emparejado siga el locale de la fuente. Los fragmentos específicos del locale exigirían que cada productor de enlaces conociera el encabezado traducido del locale de destino.

**Confiar en los ids de encabezado de VitePress.** Esos ids dependen de la puntuación renderizada y del texto localizado del encabezado. No preservan los ids GitHub ya usados por los enlaces del repositorio y las referencias generadas.

**Verificar solo el Markdown fuente.** Esto deja el artefacto publicado sin verificar y no puede detectar diferencias entre los algoritmos de slug de GitHub y VitePress.

## Consecuencias

Cada build de documentación de producción lee una vez su HTML emitido, añadiendo una comprobación post-build acotada al build del sitio existente. Los enlaces de fragmento entre páginas ahora requieren un id que sobreviva a la publicación. Los alias explícitos pasan a formar parte de la referencia publicada y permiten que los encabezados cambien de idioma o puntuación sin invalidar fragmentos establecidos.
