# Agent Note: Estandarizar la terminología china de contract en 约定

Status: implemented

[English](2026-08-09-chinese-contract-terminology.md) | [中文](2026-08-09-chinese-contract-terminology.zh.md) | Español

## Problema

La documentación china renderizaba el inglés `contract` de forma inconsistente como `契约` y `约定`, a veces dentro de un mismo archivo o párrafo. La tabla de terminología prescribía `契约`, mientras que la corrección incremental revisada elegía el renderizado de ingeniería más natural, `约定`. Dejar la tabla y el corpus divididos hacía que cualquiera de las dos opciones incumpliera la regla de terminología del repositorio y permitía que traducciones posteriores reintrodujeran el desacuerdo.

El inglés `convention` también se renderiza habitualmente como `约定`. Esa superposición es intencional: la prosa de ingeniería china ordinaria usa `约定` para ambos conceptos, y el contexto suele indicar si una afirmación es práctica descriptiva o una regla de interfaz vinculante. Cuando una oración inglesa contrasta explícitamente una convention con un contract, la oración china debe conservar la distinción mediante redacción como `惯例` frente a `约定`, en lugar de dar mecánicamente a cada `convention` el mismo renderizado.

## Decisión

La fuente de verdad de terminología define `contract` como `约定` y `adapter contract` como `适配器约定（adapter contract）` en la primera mención. Todo par de documentación china activo sigue ese fallo; las Agent Notes archivadas permanecen congeladas. Los activos de calibración bilingües no emparejados y la prosa explicativa del prompt de traducción siguen los mismos términos para no enseñar el renderizado superado.

La migración es mantenimiento semántico de prosa, no un cambio de nombre de identificadores. El código en línea, las rutas de archivo, los enlaces, los nombres de API, los nombres de archivo en inglés que contienen `contract` y los valores legibles por máquina permanecen sin cambios. `convention` no recibe una fila global de terminología ni una reescritura en todo el corpus: los traductores conservan el chino natural y desambiguán explícitamente solo donde la fuente contrasta los dos conceptos. La [decisión de prosa concreta](2026-08-09-concrete-prose-names-actors-and-recorded-facts.es.md) decide por separado cuándo la prosa inglesa debe sustituir un uso vago de `contract` por la regla, API o comportamiento exactos antes de la traducción.

## Alternativas consideradas

**Mantener `contract` como `契约`.** Rechazado porque el corpus revisado prefería de forma consistente `约定` para interfaces técnicas, garantías de ciclo de vida y límites de comportamiento, y mantener el término antiguo habría exigido revertir correcciones aceptadas en muchos documentos.

**Dar a `convention` un renderizado global obligatorio.** Rechazado porque su significado va desde práctica de nomenclatura hasta convención de protocolo. Un único término forzado habría creado una segunda migración amplia sin mejorar la prosa ordinaria; solo los contrastes explícitos de la fuente exigen un renderizado distinto.

**Permitir tanto `契约` como `约定` para `contract`.** Rechazado porque conserva exactamente la inconsistencia que hacía discrepar a familias de paquetes e incluso a párrafos individuales.

## Consecuencias

La documentación china activa tiene un renderizado vinculante para `contract`, y los prompts de traducción futuros reciben esa decisión directamente de la tabla de terminología. Los registros archivados conservan su texto histórico. Una oración fuente que contrasta convention y contract exige redacción semántica local, de modo que las opciones chinas equivalentes del diccionario nunca borran una distinción que la fuente usa realmente.

## Verificación

La migración escanea cada par bilingüe activo, actualiza cada documento chino afectado, vuelve a registrar su sidecar de emparejamiento y deja la prosa activa sin apariciones de `契约`. La puerta de emparejamiento, el `doc-sync` completo, la compilación del sitio web, las pruebas y la instantánea del prompt de traducción, y `git diff --check` verifican el corpus y los activos de la canalización resultantes.
