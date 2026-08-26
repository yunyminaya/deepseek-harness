# @deepseek-ai/dsh-typert-generator

[English](README.md) | Español

Analizador de proyectos TypeScript y generador Typert dirigido por modelo. Convierte el árbol de tipos de origen escrito por el desarrollador en datos `FaceModel` y `TypeGraph` independientes del compilador antes de renderizar cualquier artefacto. El análisis estático puede consumir ese modelo sin Cordis; los emisores nunca reciben objetos del AST ni del checker de TypeScript.

El analizador puede usar instancias `ts.Program` independientes sembradas desde `tsconfig.host.json` o `tsconfig.client.json`. Las referencias directas de proyecto establecen la pertenencia a la cara de compilador, mientras que los subpaths de paquete establecen las contribuciones a la cara de runtime de Typert: un paquete ordinario de un solo proyecto que declare `dsh.client` puede contribuir tanto al modelo de runtime de Host como al de Cliente, y solo un proyecto dividido referenciado explícitamente a través de `tsconfig.host.json` o `tsconfig.client.json` queda restringido a esa cara correspondiente. `package.json#exports` establece cada frontera pública entre paquetes, y los imports y re-exports de origen son las únicas aristas entre caras permitidas. Los tipos propiedad de dependencias NPM, incluidas las declaraciones globales de los paquetes `@types`, siguen siendo referencias `external` en lugar de expandirse.

## Modelo de análisis

Cada cara contiene las exportaciones de paquete, los servicios y eventos de Cordis, los objetos y schemas etiquetados explícitamente, y un grafo de tipos para sus declaraciones alcanzables. El grafo conserva la identidad de las declaraciones, los parámetros y aplicaciones genéricos, la herencia explícita, los tipos condicionales y mapeados, los atributos de import, los modificadores abstract y el JSDoc de origen. Las APIs de servicio y de `@typert object` exponen solo los miembros públicos de instancia; los constructores, los miembros estáticos y los miembros no públicos quedan excluidos.

`WorkspaceAnalyzer` usa por defecto el modo `check` y falla ante diagnósticos de sintaxis o semánticos de TypeScript, anotaciones públicas alcanzables ausentes, referencias privadas entre paquetes y fusiones de declaración alcanzables que el modelo no puede conservar sin pérdida. El modo `write` inserta anotaciones derivadas del checker, reconstruye el programa y devuelve un modelo limpio de modo check.

## Emisión y publicación opt-in

`FaceModelEmitter` consume solo el modelo. Emite JavaScript ejecutable que contiene los schemas Zod admitidos y una contribución `TYPERT`, además de un archivo de declaración cuyos schemas se tipan como `z.ZodType<SourceType>` a través de la exportación pública del paquete. Las proyecciones Zod no admitidas fallan en lugar de aplanar o debilitar el tipo de origen.

`WorkspaceTypertGenerator` descubre los contribuyentes recorriendo las exportaciones públicas de los paquetes alcanzables desde las aumentaciones de `Context` o `Events` de Cordis y las declaraciones `@typert` explícitas. Cuando se invoca para publicar artefactos, exige los artefactos de host en `lib/typert.host.{js,d.ts}` expuestos como `package/typert`, y los artefactos de cliente en `lib/typert.client.{js,d.ts}` expuestos como `package/client/typert`. Las declaraciones generadas exponen `TYPERT` como `unknown`, de modo que los paquetes de negocio que contribuyen no dependen del registro de runtime.

La publicación es opt-in por paquete, y los paquetes de negocio sin la entrada pública correspondiente no necesitan artefactos Typert. El tsdown de Host del repositorio ejecuta la generación Typert del workspace con `tsconfig.host.json` como única semilla de programa; produce tanto los artefactos de reflexión de Host como la proyección `typert.remote-client.*` de los contratos Remote de Host para el Cliente. El tsdown de Cliente posterior no inicia Typert ni analiza `tsconfig.client.json`. Los consumidores estáticos aún pueden llamar a `WorkspaceAnalyzer` directamente, seleccionar explícitamente una cara y un subconjunto de paquetes, y procesar paquetes por lotes sin publicar ni cargar artefactos de runtime.

## Proyección Cordis específica del repositorio

La exportación raíz del paquete incluye la extracción dirigida por modelo, las comprobaciones de completitud y los renderizadores de texto deterministas que usan los catálogos Cordis de este repositorio. Aceptan una `CordisCatalogPolicy`; los enlaces de tipos propiedad del repositorio, las clasificaciones de fundación/exención y las entradas Cordis heredadas permanecen en `scripts/gen-cordis-catalog.ts` y se pasan explícitamente. Por tanto, el paquete del generador contiene la mecánica de proyección, no una copia oculta de la taxonomía documental de este repositorio.

## Experiencia del modelo

Ninguna, ya que este paquete se ejecuta en tiempo de compilación o de prueba y nunca contribuye a una solicitud de modelo.

#### Efecto de KV Cache

Ninguno.

## Limitaciones conocidas y trabajo diferido

- Los patrones de exportación de paquete se omiten; los paquetes que contribuyen necesitan destinos de exportación concretos.
- Los re-exports con nombre y con comodín entre caras producen enlaces; los re-exports de namespace fallan hasta que `TypeTargetModel` pueda representar un namespace de módulo sin aplanarlo.
- El emisor Zod admite un subconjunto deliberado del grafo TypeScript modelado. Las declaraciones de schema genéricas y las construcciones calculadas, como las raíces de schema condicionales o mapeadas, fallan hasta que exista una política concreta de fábrica de schemas.
- Los enlaces entre caras se representan para el análisis, pero ningún schema generado requiere hoy un import Zod entre caras en runtime.
- El descubrimiento sigue los archivos de origen alcanzables desde las exportaciones públicas concretas; las declaraciones que ese grafo ni exporta ni importa quedan deliberadamente fuera del modelo del paquete.
