# @cordisjs/plugin-group

[English](README.md) | Español

Plugin de grupo del loader para anidar entradas de Cordis.

## Uso

```yaml
- id: tools
  name: '@cordisjs/plugin-group'
  group: true
  config:
    - id: logger
      name: '@cordisjs/plugin-logger-console'
```

Los grupos siempre se consideran habilitados en sí mismos, pero deshabilitar una entrada de grupo impide que sus entradas hijas se ejecuten. Los ids de entradas anidadas usan separadores `:`, por ejemplo `tools:logger`.

El paquete re-exporta la implementación `Group` de `@cordisjs/plugin-loader` como su plugin por defecto.
