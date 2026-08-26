# dsh-native-command

[English](README.md) | Español

Un **runner `execFile` sin shell y sin dependencias** compartido por las integraciones nativas del SO del host: una llamada a `runNativeCommand(command, args, signal)` ejecuta el ejecutable directamente (nunca una cadena de shell), captura stdout/stderr en utf8, propaga la anulación del llamador a la terminación del hijo y oculta la ventana de consola transitoria en Windows. Los fallos rechazan con el `code` de salida y ambos flujos capturados adjuntos, de modo que los llamadores clasifican (herramienta ausente, cancelado, fallo real) sin volver a ejecutar nada.

Sus dos consumidores son las integraciones nativas del lado host: los comandos de selector del SO del backend [`directory-picker-native`](../../host/directory-picker-native/README.md) y el traspaso de abrir-con-aplicación-por-defecto del gateway ([`dsh-host-apiproxy`](../../host/apiproxy/README.md) `host.openPath`). El tipo `NativeCommandRunner` es su frontera de comando inyectable.

Es una **biblioteca, no un servicio ni un plugin**: sin `ctx`, no registra nada, no mantiene estado, no emite eventos.

## Superficie

```ts
import { runNativeCommand, type NativeCommandRunner } from '@deepseek-ai/dsh-native-command'
```

## Experiencia del modelo

Ninguna, ya que es plomería de subprocesos del lado host; nada de esto llega a una solicitud de modelo.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una solicitud de provider.

## Limitaciones conocidas y trabajo diferido

- **Sin acotación de salida** — ambos flujos se almacenan en memoria sin límite; todos los llamadores actuales invocan herramientas nativas pequeñas cuya salida es una ruta o una línea de error. Adopta la acotación de `dsh-output-retention` antes de apuntar esto a comandos con un volumen de salida significativo.
