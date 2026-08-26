# Schemastery

[English](README.md) | Español

[![Codecov](https://img.shields.io/codecov/c/github/shigma/schemastery?style=flat-square)](https://codecov.io/gh/shigma/schemastery)
[![downloads](https://img.shields.io/npm/dm/schemastery?style=flat-square)](https://www.npmjs.com/package/schemastery)
[![npm](https://img.shields.io/npm/v/schemastery?style=flat-square)](https://www.npmjs.com/package/schemastery)
[![GitHub](https://img.shields.io/github/license/shigma/schemastery?style=flat-square)](https://github.com/shigma/schemastery/blob/master/LICENSE)

Validador de schema dirigido por tipos.

## Funcionalidades

- **Ligero.** Mucho más pequeño que otras librerías de validación.
- **Fácil de usar.** Puedes usar cualquier schema directamente como función o como constructor.
- **Potente.** Schemastery admite algunos tipos avanzados como `union`, `intersect` y `transform`.
- **Extensible.** Puedes crear tus propios tipos de schema mediante `Schema.extend()`.
- **Serializable.** Los objetos de schema se pueden serializar a JSON y luego hidratarse en otro entorno.

## Ejemplos básicos

### usar como validador (JavaScript)

```js
const Schema = require('schemastery')

const validate = Schema.number().default(10)

validate(0)     // 0
validate(null)  // 10
validate('')    // TypeError
```

### usar como constructor (TypeScript)

```ts
import Schema from 'schemastery'

interface Config {
  foo: Record<string, string>
  bar: string[]
}

const Config = Schema.object({
  foo: Schema.dict(Schema.string()).default({}),
  bar: Schema.array(Schema.string()).default([]),
})

// config is an instance of Config
// in this case, that is { foo: {}, bar: [] }
const config = new Config()
```

## Tipos generales

### Schema.any()

Afirma que el valor es de cualquier tipo.

```js
const validate = Schema.any()

validate()            // undefined
validate(0)           // 0
validate({})          // {}
```

### Schema.never()

Afirma que el valor es anulable.

```js
const validate = Schema.never()

validate()            // undefined
validate(0)           // TypeError
validate({})          // TypeError
```

### Schema.const(value)

Afirma que el valor es igual a la constante dada.

```js
const validate = Schema.const(10)

validate(10)          // 10
validate(0)           // TypeError
```

### Schema.number()

Afirma que el valor es un número.

```js
const validate = Schema.number()

validate()            // undefined
validate(1)           // 1
validate('')          // TypeError
```

### Schema.string()

Afirma que el valor es una cadena.

```js
const validate = Schema.string()

validate()            // undefined
validate(0)           // TypeError
validate('foo')       // 'foo'
```

### Schema.boolean()

Afirma que el valor es un booleano.

```js
const validate = Schema.boolean()

validate()            // undefined
validate(0)           // TypeError
validate(true)        // true
```

### Schema.is(constructor)

Afirma que el valor es una instancia del constructor dado.

```js
const validate = Schema.is(RegExp)

validate()            // undefined
validate(/foo/)       // /foo/
validate('foo')       // TypeError
```

### Schema.array(inner)

Afirma que el valor es un array de `inner`. El valor por defecto será `[]` si no se especifica.

```js
const validate = Schema.array(Schema.number())

validate()                  // []
validate(0)                 // TypeError
validate([0, 1])            // [0, 1]
validate([0, '1'])          // TypeError
```

### Schema.dict(inner)

Afirma que el valor es un diccionario de `inner`. El valor por defecto será `{}` si no se especifica.

```js
const validate = Schema.dict(Schema.number())

validate()                  // {}
validate(0)                 // TypeError
validate({ a: 0, b: 1 })    // { a: 0, b: 1 }
validate({ a: 0, b: '1' })  // TypeError
```

### Schema.tuple(list)

Afirma que el valor es una tupla en la que cada elemento es del subtipo correspondiente. El valor por defecto será `[]` si no se especifica.

```js
const validate = Schema.tuple([
  Schema.number(),
  Schema.string(),
])

validate()                  // []
validate([0])               // { a: 0 }
validate([0, 1])            // TypeError
validate([0, '1'])          // [0, '1']
```

### Schema.object(dict)

Afirma que el valor es un objeto en el que cada propiedad es del subtipo correspondiente. El valor por defecto será `{}` si no se especifica.

```js
const validate = Schema.object({
  a: Schema.number(),
  b: Schema.string(),
})

validate()                  // {}
validate({ a: 0 })          // { a: 0 }
validate({ a: 0, b: 1 })    // TypeError
validate({ a: 0, b: '1' })  // { a: 0, b: '1' }
```

### Schema.union(list)

Afirma que el valor es uno de los tipos especificados.

```js
const validate = Schema.union([
  Schema.number(),
  Schema.string(),
])

validate()                  // undefined
validate(0)                 // 0
validate('1')               // '1'
validate(true)              // TypeError
```

### Schema.intersect(list)

Afirma que el valor debe cumplir cada uno de los tipos especificados.

```js
const validate = Schema.intersect([
  Schema.object({ a: Schema.string().required() }),
  Schema.object({ b: Schema.number().default(0) }),
])

validate()                  // TypeError
validate({ a: '' })         // { a: '', b: 0 }
validate({ a: '', b: 1 })   // { a: '', b: 1 }
validate({ a: '', b: '2' }) // TypeError
```

### Schema.transform(inner, callback)

Afirma que el valor es del subtipo especificado y luego lo transforma con `callback`.

```js
const validate = Schema.transform(Schema.number().default(0), n => n + 1)

validate()                  // 1
validate('0')               // TypeError
validate(10)                // 11
```

## Métodos de instancia

Nota: `default` y `required` son mutuamente excluyentes.

### schema.required()

Afirma que el valor no es anulable.

### schema.default(value)

Establece el valor de respaldo cuando es anulable.

### schema.description(text)

Establece la descripción del schema.

### schema.simplify(value)

Normaliza un valor eliminando las partes que son iguales a los valores por defecto del schema. Esto es útil al almacenar la configuración del usuario y para mantener compactos los archivos persistidos.

```js
const Config = Schema.object({
  foo: Schema.string().default(''),
  bar: Schema.number().default(0),
})

Config.simplify({ foo: '', bar: 1 }) // { bar: 1 }
```

## Opciones de validación

Todos los schemas son invocables. El segundo argumento acepta opciones de validación:

```js
const Config = Schema.object({
  foo: Schema.number(),
})

Config({ foo: '1' }, { autofix: true }) // {}
```

- `autofix`: elimina las propiedades de objeto inválidas cuando es posible.
- `ignore`: omite la validación para los valores y nodos de schema seleccionados.
- `path`: proporciona una ruta inicial para los errores de validación anidados.

## Sintaxis abreviada

Existe cierta sintaxis abreviada para los tipos internos.

- `undefined` -> `Schema.any()`
- `String` -> `Schema.string()`
- `Number` -> `Schema.number()`
- `Boolean` -> `Schema.boolean()`
- `1` -> `Schema.const(1)` (solo para tipos primitivos)
- `Date` -> `Schema.is(Date)`

```js
Schema.array(String)        // Schema.array(Schema.string())
Schema.dict(RegExp)         // Schema.dict(Schema.is(RegExp))
Schema.union([1, 2])        // Schema.union([Schema.const(1), Schema.const(2)])
```

También puedes usar `Schema.from()` para obtener el schema inferido a partir de un valor abreviado.

```js
Schema.from()               // Schema.any()
Schema.from(Date)           // Schema.is(Date)
Schema.from('foo')          // Schema.const('foo')
```

## Ejemplos avanzados

Aquí hay algunos ejemplos que demuestran cómo definir tipos avanzados.

### Enumeración

```js
const Enum = Schema.union(['red', 'blue'])

Enum('red')                 // 'red'
Enum('blue')                // 'blue'
Enum('green')               // TypeError
```

### ToString

```js
const ToString = Schema.transform(Schema.any(), v => String(v))

ToString('')                // ''
ToString(0)                 // '0'
ToString({})                // '{}'
```

### Listable

```js
const Listable = Schema.union([
  Schema.array(Number),
  Schema.transform(Number, n => [n]),
]).default([])

Listable()                  // []
Listable(0)                 // [0]
Listable([1, 2])            // [1, 2]
```

### Alias

```js
const Config = Schema.dict(Number, Schema.union([
  'foo',
  Schema.transform('bar', () => 'foo'),
]))

Config({ foo: 1 })          // { foo: 1 }
Config({ bar: 2 })          // { foo: 2 }
Config({ bar: '3' })        // TypeError
```

## Extensibilidad

Los tipos de schema personalizados se registran con `Schema.extend(type, resolve)`. Un resolver recibe el valor de entrada, el nodo de schema, las opciones de validación y una bandera de estricto. Devuelve `[value]` para la entrada aceptada, o `[value, adapted]` cuando el llamador debe escribir un valor adaptado de vuelta en el objeto fuente.

```js
Schema.extend('trimmed', (data, schema, options) => {
  if (typeof data !== 'string') {
    throw new Schema.ValidationError(`expected string but got ${data}`, options)
  }
  return [data.trim()]
})
```

## Serializabilidad

```js
const schema1 = Schema.object({
  foo: Schema.string(),
  bar: Schema.number(),
})

// should have the same effect as schema1
const schema2 = new Schema(JSON.parse(JSON.stringify(schema1)))
```

Schemastery también expone la propiedad `~standard` del Standard Schema, de modo que las herramientas compatibles pueden validar valores sin depender de las API específicas de Schemastery.
