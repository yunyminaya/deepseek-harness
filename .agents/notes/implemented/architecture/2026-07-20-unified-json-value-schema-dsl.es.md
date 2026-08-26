# Agent Note: DSL unificado de schema de valores JSON

Status: implemented

[English](2026-07-20-unified-json-value-schema-dsl.md) | Español

## Problema

Los parámetros de herramienta usaban un DSL de autor pequeño mientras que la salida estructurada de subagente (subagent)/flujo de trabajo usaba un subconjunto y validador separados de JSON Schema crudo. Los dos vocabularios discrepaban sobre raíces, restricciones escalares y validación, de modo que un contrato canónico tipado de salida de herramientas duplicaría de nuevo ambas rutas o aceptaría schemas que alguna proyección no pudiera aplicar.

## Decisión

`dsh-tools` es dueño de un vocabulario de schema de valores JSON con dos representaciones. `ValueSchemaSpec` es la forma de autor para cualquier raíz JSON; `ParameterSchemaSpec` es su forma implícita de mapa de propiedades de objeto con `required: true` por propiedad. `JsonSchemaNode` es la forma de cable cruda. Ambas soportan string, número finito, entero, booleano, null, array, objeto, `enum`/`const` escalar correctamente tipado y `oneOf` de exactamente uno; `{ type: 'json' }` es azúcar solo de autor para un nodo crudo sin restricciones, solo de anotación.

Un objeto de autor explícito debe declarar `additionalProperties: true | false`. La raíz de parámetros implícita y el JSON Schema crudo conservan el valor por defecto abierto estándar. Los registros de schema contienen solo claves string enumerables propias, los arrays de schema son arrays intrínsecos densos, y las palabras clave soportadas se leen como propiedades propias; los prototipos personalizados, las restricciones heredadas, los símbolos y las decoraciones invisibles al JSON no pueden por tanto hacer que la compilación, la proyección y la validación observen declaraciones distintas. Los contenedores intrínsecos plain Object y Array permanecen plain entre realms, mientras que las subclases y los prototipos de constructor forjados siguen siendo exóticos.

`InferValue<S>` e `InferArgs<P>` derivan valores de TypeScript de las mismas declaraciones que compilan `valueSchemaSpecToJsonSchema()` y `parameterSchemaSpecToJsonSchema()`. La inferencia exacta está acotada a 16 niveles de contenedor y después usa `JsonValue`, impidiendo que la pila de instanciación de tipos de TypeScript se convierta en el límite de autoría. `assertSupportedJsonSchema()` rechaza palabras clave no soportadas o mal ubicadas, y `validateJsonSchemaValue()` aplica el subconjunto aceptado contra la frontera de `JsonValue` sin pérdida: sin `undefined`, cero negativo, números no finitos, arrays dispersos, ciclos, objetos exóticos, funciones, símbolos ni otros valores coercitivos. La compilación de autor, la afirmación de schema crudo, la validación de valores, el renderizado de schema a TypeScript, el desacople del registro y la normalización y clonación dinámicas de Cordis entre realms usan pilas de trabajo explícitas, de modo que el anidamiento en runtime queda limitado por la memoria disponible en lugar de por la pila de llamadas de JavaScript.

La raíz de objeto es una regla de consumidor en lugar de una restricción del vocabulario. Las salidas estructuradas definidas por el llamador de subagente/flujo de trabajo usan `assertObjectJsonSchema()` y `ObjectJsonSchema`; las salidas de herramienta pueden usar cualquier raíz. Los registros dinámicos de Cordis reconstruyen los schemas ajenos al realm en JSON propiedad del host, conservan la apertura cruda del wrapper y exigen apertura de objeto del DSL directo antes de llamar al mismo compilador. La frontera dinámica rechaza las claves de registro invisibles al JSON y los arrays de schema exóticos antes de la normalización, de modo que no puede descartar en silencio una restricción ni consumir semántica de iteración personalizada.

## Alternativas consideradas

- **Mantener sistemas de schema separados para parámetros y salida estructurada:** rechazado porque cada constructo de salida añadido exigiría cambios paralelos de inferencia, compilación, validación y generación de código sin una frontera de propiedad útil.
- **Usar Schemastery para los parámetros de herramienta:** rechazado porque Schemastery apunta a la validación y transformación mediante Standard Schema en lugar de a la generación de JSON Schema. Añadiría una capa de adaptador sin producir el schema de cable orientado al modelo ni el vocabulario de salida compartido.
- **Adoptar JSON Schema completo o Ajv:** rechazado porque el harness debe fallar en cada constructo que no pueda proyectar en su SDK generado y sus validadores; aceptar un lenguaje mayor haría deshonesta la aplicación y la guía de modelo.
- **Hacer implícitamente abierto o cerrado todo objeto:** rechazado porque cualquiera de las dos elecciones oculta una decisión de autor con consecuencias. Solo la raíz de parámetros implícita con forma heredada y el schema crudo externo conservan un valor por defecto intencional.
- **Definir `oneOf` como primera coincidencia:** rechazado porque el orden de ramas cambiaría la semántica de validación y permitiría que ramas solapadas ocultaran valores ambiguos.

## Consecuencias

- La validación de parámetros, la validación de salida, la generación de schema a TypeScript, las salvaguardas de subagente/flujo de trabajo y el registro dinámico comparten un único vocabulario aplicado.
- Las declaraciones de salida pueden inferir raíces de objeto, array, escalar o null; las salidas estructuradas de subagente/flujo de trabajo permanecen con raíz de objeto en sus seams existentes.
- La apertura explícita de objeto y las restricciones literales correctamente tipadas hacen que las declaraciones malformadas fallen durante la autoría o el registro en lugar de durante una llamada de modelo posterior.
- La inferencia de tipos acotada conserva tipos exactos útiles para las declaraciones ordinarias y degrada las colas inusualmente profundas a `JsonValue`; la aplicación del schema en runtime sigue siendo exacta en cada profundidad.
- Las herramientas crudas pueden seguir registrando JSON Schema más amplio directamente, pero la generación de código unificada trata los schemas no soportados como desconocidos en lugar de fingir que los aplica.
- El `required: true` por propiedad sigue siendo el contrato del autor de herramientas, y la cobertura de regresión a nivel de tipos fija las claves requeridas como no opcionales después de que la ruta de inferencia original expusiera un bug de opcionalidad.
- Los tests de runtime y de compilación cubren cada raíz, el comportamiento exacto-uno de solape/sin coincidencia, los valores por defecto abiertos crudos, la apertura explícita, los valores JSON con pérdida, la inferencia, el anidamiento profundo a través de las proyecciones de core y dinámicas, las claves dinámicas invisibles al JSON y los arrays de schema exóticos.
