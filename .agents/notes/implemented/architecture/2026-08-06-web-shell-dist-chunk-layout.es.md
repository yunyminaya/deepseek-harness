# Agent Note: División de chunks del dist del web shell y estructura de directorios

Status: implemented

[English](2026-08-06-web-shell-dist-chunk-layout.md) | Español

## Problema

Antes, el shell de apps/web se compilaba en un único chunk index de ~1,2 MB (minificado), del que el 80 % aproximadamente eran bytes de vendor — KaTeX, las gramáticas de arranque y el motor shiki, react-dom, la tubería de markdown — fundidos con todo el código del shell del workspace (alrededor de una quinta parte). Cualquier cambio de una línea en el shell re-hasheaba el chunk entero, obligando a los clientes que volvían a descargarlo todo de nuevo; `dist/assets/` era una extensión plana de un solo nivel con más de 100 archivos (el chunk principal, 23 chunks de gramáticas con carga diferida, 59 caras de fuente KaTeX y sourcemaps entremezclados), imposible de navegar.

## Decisión

`apps/web/vite.config.ts` divide el shell en dos chunks iniciales mediante `manualChunks` y ordena la salida en directorios mediante funciones de nombres; toda la configuración contiene cero regex — un Set de nombres de paquete exactos, una lista de nombres de archivo, una lista de extensiones.

**Pertenencia** (`VENDOR_PACKAGES`, por nombre de paquete npm exacto):

- `vendor` = las tres familias de renderizado pesadas: matemática (katex), resaltado (shiki), markdown (la tubería de análisis micromark/mdast — el renderizador React incremental que va encima es código del workspace y no forma parte de esto). La pertenencia viva es `VENDOR_PACKAGES`; la lista son los paquetes que el código del workspace **importa directamente**: el resto de dependencias transitivas privadas (la familia oniguruma, @shikijs/core, tablas de caracteres, docenas más) solo las referencian los miembros listados, así que el coloreado de chunks de rollup las arrastra a vendor automáticamente; las dependencias compartidas con el lado index caen a index, diluyéndolo unos pocos KB — no es un problema de corrección.
- **Cada miembro de vendor debe estar libre de react (la invariante de frontera)**: rollup pliega un módulo compartido entre la entrada y un manual chunk dentro del manual chunk — que un paquete listado importe react/jsx-runtime arrastraría la única copia compartida de react a vendor, lejos de index. El lado React del renderizado de markdown/matemática es código del workspace y vive naturalmente en index, así que toda la familia react queda clavada a index.
- `index` (el chunk por defecto) = la familia react (react, react-dom, scheduler, use-sync-external-store), cordis vendorizado, todo el código del workspace y las piezas pequeñas no listadas (anser, clsx).
- `@shikijs/langs` tiene un caso especial: las gramáticas de arranque (`BOOT_GRAMMAR_FILES`: typescript, shellscript, json — las tres que highlight.ts importa estáticamente, todas módulos de datos autocontenidos con cero imports internos) van a vendor; las 23 gramáticas restantes con carga diferida no reciben asignación y cada una conserva su propio chunk bajo demanda.
- `index.html` queda conectado automáticamente por vite: index carga vía `<script>` y vendor vía `<link rel="modulepreload">`, de modo que los dos chunks se obtienen en paralelo sin waterfall.

**Estructura de directorios** (`chunkFileNames` + `assetFileNames`):

- La raíz `assets/` conserva solo el js de index y vendor (con sus sourcemaps adyacentes) y el css.
- Los chunks de gramáticas van bajo `assets/langs/`. El criterio es si los `moduleIds` de un chunk incluyen un miembro de `@shikijs/langs`, no la facade: los chunks compartidos de gramáticas embebidas (php/ruby/mdx embeben html+javascript, que rollup separa para compartir) **no tienen facade**, así que un criterio de facade los pasaría por alto; index y vendor se excluyen por nombre, porque vendor lleva legítimamente las tres gramáticas de arranque.
- Las fuentes van bajo `assets/fonts/` (`FONT_EXTENSIONS`: woff2/woff/ttf; hoy todas son caras KaTeX referenciadas por vendor.css — katex.min.css lo importa un componente del lado de index, pero los módulos CSS pasan por manualChunks como cualquier módulo y siguen a `katex` dentro de vendor.css; el navegador solo obtiene woff2, bajo demanda y solo cuando se renderiza una fórmula).
- Los sourcemaps no necesitan ordenación: rollup escribe cada `.map` junto a su js y lo referencia por nombre de archivo relativo desnudo, así que cuando un chunk cambia de directorio su map lo sigue automáticamente.

Todas las referencias entre directorios (los imports dinámicos de index hacia `langs/`, las referencias relativas dentro del mismo directorio entre chunks de gramáticas, las referencias relativas de vendor.css hacia `fonts/`) las emite el bundler, así que el runtime no necesita ningún cambio acompañante; el servidor web del lado host sirve las rutas anidadas tal cual bajo su prefijo estático.

## Alternativas consideradas

- **Servir react y los demás vendors desde una CDN**: dsh web apunta a hosts locales/intranet (a menudo sin acceso a internet), así que una CDN simplemente no está disponible; react es la seed de plataforma externa a todo bundle de plugin (el shell es su único proveedor), y cambiar a la forma de variable global de CDN tocaría tres lugares — el manifest de la plataforma, la seed y la tabla de módulos; el beneficio de caché ya lo entrega la división de vendor.
- **Una regla catch-all inversa (todo lo de node_modules excepto la familia react va a vendor)**: la pertenencia no puede leerse de la configuración, y piezas pequeñas como anser/clsx se asignan mal a vendor; superada por la lista positiva de nombres de paquete exactos.
- **Coincidencia por familia de regex**: difícil de leer; los nombres de paquete exactos más el coloreado automático de dependencias transitivas de rollup hacen innecesaria la coincidencia por patrones.
- **Identificar los chunks de gramáticas por facadeModuleId**: los chunks compartidos sin facade de gramáticas embebidas pasarían desapercibidos y caerían al directorio raíz; el criterio de pertenencia por `moduleIds` cubre ambas formas.
- **Resguardar en vendor una facade de renderizado con borde react** (la react-markdown histórica era una): el plegado de módulos compartidos de rollup arrastraría la única copia de react a vendor, rompiendo la frontera de «react pertenece a index»; la restricción queda codificada como la invariante de frontera de la lista.
- **Cargar KaTeX entero con carga diferida, o volver diferida la gramática TypeScript de arranque**: cualquiera de las dos cambiaría el comportamiento de renderizado del primer frame (el fallback para fórmulas / el primer bloque de código); ese trade-off es independiente de la estructura del dist y se decide por separado.

## Verificación

La herramienta de auditoría viaja con el repositorio: `node scripts/attribute-chunk-bytes.mjs <chunk.js>` (atribución de bytes VLQ de sourcemap sin dependencias, agregada por paquete npm / directorio del workspace). Verifica que vendor no contiene bytes del workspace, que la familia react (incluido react/jsx-runtime) está íntegra en index, y que el lado npm de index conserva solo la familia react más anser/clsx; el número de chunks de gramáticas diferidas coincide uno a uno con la tabla `LAZY_GRAMMARS`; el caso de replay sin clave del navegador es idéntico verbatim al baseline previo al cambio (salvo los rojos locales específicos del entorno), así que el shell de dos chunks carga y renderiza sin regresión.

## Consecuencias

- Un cambio de código del shell re-hashea solo index (alrededor de un tercio de la salida del dist); vendor (alrededor de dos tercios) permanece estable en caché entre releases del shell y solo se invalida con actualizaciones de dependencias.
- `dist/assets/` es navegable: dos pares js/css en la raíz, gramáticas bajo demanda en `langs/`, fuentes en `fonts/`.
- Coste de mantenimiento: cuando el código del workspace añade un import directo del paquete facade de una familia de renderizado, hay que actualizar `VENDOR_PACKAGES` a la vez (una omisión solo diluye index, no rompe nada); cuando el conjunto de gramáticas de arranque crece en highlight.ts sin que `BOOT_GRAMMAR_FILES` lo siga, esa gramática aterriza silenciosamente en index, visible solo para una auditoría del dist.
- La superficie estática del servidor web aún no tiene compresión, así que la ganancia de tamaño gzip sigue sobre la mesa; la compresión en la capa de transporte es una decisión separada e independiente.
