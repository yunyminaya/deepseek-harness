# Post-mortem 0004: el aviso de aplicación parcial de Landlock clasificó mal los fallos de los hijos

[English](0004-landlock-partial-notice-misclassified-child-failures.md) | Español

Estado: resuelto

## Resumen ejecutivo

En kernels con un ABI de Landlock más antiguo, el lanzador imprime un aviso benigno de aplicación parcial antes de ejecutar cada hijo. El harness trataba ese prefijo compartido `landlock-run:` junto con cualquier salida no cero del hijo como un fallo del lanzador, así que resultados ordinarios como el exit 1 de ripgrep sin coincidencias aparecían como `SANDBOX_UNAVAILABLE`; la búsqueda de sistema de archivos, entonces respaldada por bash, ocultaba además ese error estructurado detrás de `SEARCH_FAILED`. Unas reglas de firma demasiado amplias y la falta de cobertura de composición de ABI parcial dejaron pasar el defecto. La clasificación del runner ahora exige evidencia fatal limitada por el estado tras exclusiones informativas exactas, y un escenario ensamblado sin clave fija la ruta bash superviviente. La búsqueda de sistema de archivos usa ripgrep empaquetado a través del seam de subprocess y no cruza el bash con sandbox.

## Resumen

El contrato del lanzador nativo distingue dos tipos de líneas de stderr. Un kernel de aplicación parcial imprime exactamente `landlock-run: partial enforcement (older Landlock ABI)` y continúa hacia el hijo. Un fallo del lanzador imprime otra línea `landlock-run:` y sale con 125 sin ejecutar al hijo.

El harness representaba ambos casos con una única subcadena `landlock-run: ` insensible a mayúsculas. Su consumer clasificaba como fallo del runner cualquier salida no cero que contuviera esa subcadena. Así, el estado del hijo se atribuía a la línea informativa del lanzador: `false`, el exit 1 de ripgrep sin coincidencias, el exit 2 de patrón inválido e incluso un exit 125 elegido por el hijo podían achacarse al sandbox pese a un confinamiento y una ejecución exitosos.

En el momento del incidente, la búsqueda de sistema de archivos añadía un segundo error de atribución. Su `runRipgrep()` respaldado por bash capturaba toda ejecución de bash rechazada que no se hubiera abortado y la reemplazaba por un `SEARCH_FAILED` genérico de inicio de cwd/shell, incluido el `SandboxUnavailableError` estructurado producido por el ejecutor de sandbox.

## Impacto

En hosts de Landlock con ABI parcial, resultados legítimos no cero de los hijos podían aparecer como fallo de infraestructura del sandbox. `glob` y `grep` eran especialmente visibles porque ripgrep usa el exit 1 como búsqueda vacía exitosa. Cuando sí ocurría un fallo real de sandbox a través de la búsqueda de sistema de archivos, los llamadores perdían su código `SANDBOX_UNAVAILABLE` y recibían un diagnóstico de arranque incorrecto.

El defecto no debilitaba el confinamiento ni ejecutaba un comando sin confinar. Su efecto de seguridad fue la disponibilidad y la integridad diagnóstica: un resultado confinado válido se rechazaba o se etiquetaba mal.

## Cronología

- El contrato del lanzador nativo definía el exit 125 para los fallos del lanzador, una línea fatal `landlock-run:` para cada uno de esos fallos y el aviso exacto de aplicación parcial para la ejecución exitosa del hijo.
- El provider de sandbox redujo ese contrato a `runnerFailureSignatures: ['landlock-run: ']`; el consumer de bash combinaba el prefijo con cualquier salida no cero e informaba de la primera línea de stderr.
- Las pruebas unitarias cubrían el éxito limpio, los diagnósticos de denegación y los prefijos fatales del runner. Las pruebas de runner real se auto-omitían sin un kernel utilizable y no forzaban una aplicación parcial seguida de un hijo con salida no cero.
- Un wrapper POSIX mínimo que imprime el aviso y hace `exec` de su payload reprodujo el fallo con `false` y con ripgrep sin coincidencias.
- Reglas estructuradas más una clasificación compartida de primer plano/fondo y cobertura de reproducción ensamblada cerraron la brecha de atribución de sandbox superviviente. La búsqueda de sistema de archivos usa ripgrep empaquetado a través de `ctx.subprocess`; la corrección deja esa ruta fuera del bash con sandbox.

## Causa raíz

El tipo de resultado público del sandbox solo podía expresar un saco de subcadenas. No podía afirmar que un fallo de Landlock exige el exit 125, que la evidencia debe aparecer dentro de una única línea fatal, o que una línea exacta bajo el mismo prefijo es informativa. En consecuencia, el consumer booleano unía hechos no relacionados de procesos distintos y elegía la primera línea de stderr como detalle incluso cuando la evidencia fatal estaba en una línea posterior.

La matriz de pruebas reflejaba esa representación. Los providers falsos emitían o ninguna línea de runner o un prefijo inequívocamente fatal; nunca emitían una línea benigna de runner antes de una salida no cero controlada por el hijo. La cobertura real de Landlock dependía del ABI del host, así que los hosts de ABI completo no podían ejercitar el aviso. En la implementación de búsqueda de la era del incidente, las pruebas de búsqueda de sistema de archivos modelaban errores de spawn crudos, pero no un error estructurado lanzado por la composición real de bash con sandbox.

stderr sigue siendo un canal de atribución en banda. Un hijo confinado puede reproducir deliberadamente la línea fatal protegida del runner y su estado de salida, provocando una falsa atribución de disponibilidad/diagnóstico. La conjunción más estricta evita la colisión accidental de este incidente, pero no autentica al escritor; un protocolo de estado fuera de banda sigue siendo un endurecimiento aparte, no una corrección que eluda el sandbox.

## Protecciones añadidas

- [`RunnerFailureRule`](../subsystems/sandbox.es.md#wrapped-argv-and-classification-dialects) lleva códigos de salida permitidos opcionales, firmas fatales por línea insensibles a mayúsculas y exclusiones exactas de líneas informativas insensibles a mayúsculas.
- [`dsh-sandbox-local`](../../packages/sandbox/sandbox-local/) asigna Landlock al exit 125 más una línea `landlock-run:` que no es el aviso, mientras que bwrap, Seatbelt y los runners personalizados siguen siendo solo de firma.
- [`dsh-bash-sandbox`](../../packages/shell/bash-sandbox/) hace spawn directo del argv del provider, así que un rechazo previo al arranque usa el canal de error de spawn en lugar de diagnósticos de shell localizados. La ejecución asentada de primer plano y de fondo comparte un único clasificador que devuelve evidencia; la evidencia fatal tiene prioridad sobre la denegación, y los errores de primer plano informan de la línea fatal coincidente sin alterar el stderr capturado.
- [`dsh-tool-fs-search`](../../packages/fs/tool-fs-search/) usa ripgrep empaquetado a través de `ctx.subprocess` y permanece fuera del seam de bash con sandbox.
- Los casos de regresión de la frontera nativa viven en [`partial-landlock.spec.ts`](../../packages/shell/bash-sandbox/tests/partial-landlock.spec.ts), e incluyen avisos informativos, evidencia fatal y clasificación de primer plano/fondo.
- La ruta ensamblada del producto queda fijada por la [composición de instantánea `partial-landlock`](../../examples/acp-agent/partial-landlock.cordis.snapshot.yml), con independencia de las decisiones de implementación de la búsqueda de sistema de archivos.

## Lecciones

- La atribución de procesos exige una conjunción de evidencia independiente; un prefijo compartido no es un protocolo.
- Los diagnósticos informativos y fatales pueden compartir un espacio de nombres, así que las exclusiones deben ser exactas y estrechas mientras las líneas fatales desconocidas sigan siendo fail-closed.
- Un adaptador debe preservar los fallos estructurados que pertenecen al seam que tiene debajo, en lugar de reemplazarlos con su propia categoría genérica más cercana.
- El comportamiento dependiente de la plataforma necesita un fake determinista en la frontera nativa más una ruta ensamblada del producto; una prueba de kernel real que se auto-omite no puede soportar esa regresión por sí sola.
