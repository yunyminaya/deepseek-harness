# Post-mortem 0002: las herramientas de instantánea del sistema de archivos quedaron permanentemente deshabilitadas

[English](0002-js-expression-disabled-filesystem-tools.md) | Español

Estado: resuelto

## Resumen ejecutivo

El ejemplo de ACP intentaba habilitar los plugins de sistema de archivos de forma condicional con `disabled: !!js ...`, pero Cordis solo evalúa las expresiones JavaScript dentro del `config` del plugin. El objeto de expresión crudo era truthy, así que la pila de sistema de archivos quedó siempre deshabilitada. La actualización de instantáneas aceptó entonces los resultados `UNKNOWN_TOOL` como nuevas salidas esperadas. La corrección usa un overlay de sistema de archivos explícito y añade guardas de configuración estática y de resultados de instantánea.

## Resumen

La composición ACP predeterminada es intencionadamente solo-bash porque su sandbox no puede confinar los providers de sistema de archivos en proceso. Los escenarios de instantánea del sistema de archivos siguen necesitando `read`, `write` y `edit`, así que sus plugins se colocaron en el `cordis.yml` predeterminado con una expresión `disabled` pensada para habilitarlos solo en los lanzamientos de acceso total y en las instantáneas.

Cordis Include analizaba cada escalar `!!js` como un objeto de expresión. El Loader interpolaba recursivamente el `config` del plugin, pero consumía directamente metadatos de la entrada como `disabled`. Por tanto, cada entrada de sistema de archivos veía un objeto truthy y permanecía deshabilitada en todos los modos.

## Impacto

Siete escenarios de sistema de archivos y el escenario mixto de edición de workspace llamaban a herramientas ausentes del registro. Sus registros de sesión estructurados contenían `ToolNotFoundError` con el código `UNKNOWN_TOOL`, mientras que stdout renderizaba tarjetas genéricas de herramienta fallida. La suite de instantáneas pasó porque ambas salidas coincidían con los fixtures actualizados; demostró la reproducción determinista de la regresión, no un comportamiento correcto del sistema de archivos.

El modo predeterminado confinado en vivo no obtuvo acceso no deseado al sistema de archivos. Una corrección ingenua de la interpolación habría creado ese riesgo: los preajustes de permiso actualizan en tiempo de ejecución el sandbox de bash y el estado de aprobación, pero no pueden montar, desmontar ni confinar la pila de sistema de archivos.

## Cronología

- El PR #261 consolidó las composiciones de ACP y actualizó las instantáneas del sistema de archivos al tiempo que introducía entradas de sistema de archivos condicionales.
- Todas las comprobaciones de unidades, cobertura, instantáneas, documentación, compilación e higiene pasaron.
- La revisión de las salidas esperadas actualizadas del sistema de archivos encontró tarjetas genéricas de fallo y resultados estructurados `UNKNOWN_TOOL`.
- Un arranque real del Loader confirmó que cada valor `disabled` seguía siendo un objeto de expresión y que cada fiber de sistema de archivos estaba ausente.

## Causa raíz

La implementación asumía que `!!js` se aplicaba a una entrada completa del Loader. Solo se aplica a `entry.options.config`: `Entry._resolveConfig()` interpola ese campo, mientras que `Entry.disabled` comprueba `entry.options.disabled` sin interpolación. La etiqueta YAML era sintácticamente válida, así que la carga no produjo ningún diagnóstico.

El marco de instantáneas trataba cualquier transcript determinista como comportamiento válido. Los pines de cabecera verificaban los schemas de herramientas compuestos, pero los escenarios de sistema de archivos compartían un pin de la composición predeterminada y, por tanto, no demostraban de forma independiente que sus herramientas requeridas estuvieran registradas. La actualización reescribía el stdout esperado y los registros de sesión antes de que ninguna aserción semántica rechazara las herramientas ausentes.

## Protecciones añadidas

- Los escenarios de sistema de archivos arrancan `fs.cordis.yml`, un overlay de acceso total fijo y explícito con una config de reproducción emparejada y su propia clase de cabecera de solicitud.
- [`AGENTS.md`](../../AGENTS.md) y el [primer de Cordis](../cordis-primer.es.md#loader-configuration) establecen que `!!js` solo es válido bajo el `config` del plugin y que la composición condicional usa overlays.
- `verify-cordis-config` analiza el YAML de Cordis del repositorio y rechaza los nodos de expresión en los metadatos de entrada del Loader, incluidos los patches include y las entradas insertadas.
- `dsh-acp-snapshot` rechaza los resultados estructurados `UNKNOWN_TOOL` en ejecuciones nuevas y en fixtures de sesión ya confirmados, antes de que puedan confirmarse como salidas esperadas.

## Lecciones

- Un valor de configuración aceptado sintácticamente no se evalúa necesariamente en ese lugar; documenta y verifica exactamente qué campos se interpolan.
- Una actualización de instantáneas es producción de fixtures, no revisión de corrección. Las imposibilidades semánticas, como la ausencia de una herramienta registrada, necesitan aserciones independientes de la salida esperada.
- Los controles de permiso deben describir solo las capacidades que realmente gobiernan. El acceso al sistema de archivos en tiempo de composición no puede seguir de forma segura un preajuste solo-bash de tiempo de ejecución.
