# Agent Note: Mensaje de cierre de la ronda de objetivo

[English](2026-08-02-goal-round-wrapup-message.md) | [中文](2026-08-02-goal-round-wrapup-message.zh.md) | Español

Status: implemented

## Problema

Una ronda de objetivo autónoma que informó de `update_goal` con `complete` o `blocked` concluía el turno físico en el resultado de la herramienta, por lo que el modelo nunca hablaba después de la llamada. Las sesiones terminaban en una tarjeta de `update_goal` desnuda, y los evaluadores internos lo leían como si el agent se detuviera a mitad de frase: el texto previo a la llamada del modelo anuncia de forma rutinaria un informe («goal achieved, marking complete:») que nunca llega, porque la expectativa estándar del uso de herramientas es un mensaje de assistant más después del resultado de una herramienta, y ni el prompt de la ronda de objetivo ni la descripción de la herramienta decían que la llamada fuera terminal. La detención dura venía de la [decisión de las goal tools](../feature/2026-07-19-model-facing-goal-tools.es.md), cuya cláusula de fin de turno esta nota sustituye.

## Decisión

Un éxito de `complete` o `blocked` en una ronda de objetivo ya no llama a `concludeTurn()`. En su lugar, la herramienta difiere un contexto de cierre sobre su propio resultado: un mensaje de usuario con origen en `{ kind: 'plugin', plugin: 'tool-goal' }` que lleva una instrucción `<goal_complete>`/`<goal_blocked>` de escribir un mensaje de cierre fundamentado al usuario y no llamar a más herramientas. El turno termina entonces a través de la detención ordinaria del agent loop por ausencia de llamadas de herramienta, por lo que no existe ningún primitivo de bucle nuevo y la semántica de steering no se toca. Las mutaciones directas a humanos permanecen sin instrucciones exactamente como antes. El coste es una petición de modelo adicional por ciclo de vida de objetivo, no por ronda.

La redacción de la instrucción se seleccionó mediante muestreo A/B en `deepseek-v4-pro` con un transcript de ronda de objetivo reconstruido: una instrucción estructurada (resultado, verificación, artefactos, siguientes pasos) superó sistemáticamente a una mínima de «resume» en cuanto a completitud; añadir una cláusula de anclaje a la sesión desplazó el detalle no respaldado de hecho afirmado a sugerencia matizada; y el control sin instrucción produjo cierres de alta varianza, incluido detalle a nivel de archivo fabricado con seguridad.

Para automatizar la prueba sin clave se necesitó una adición al harness de instantáneas: `dsh-llm-replay` resuelve los marcadores `{{fromRequest:<regex>}}` de las entradas guionizadas contra la petición en vivo, porque un sidecar estático no puede conocer el id de objetivo acuñado aleatoriamente que el modelo debe repetir en `update_goal`.

## Verificación

Las pruebas del paquete `tool-goal` fijan el contexto inyectado (fuente, etiqueta, objetivo, cláusula de no-más-herramientas) y la ausencia de `concludesTurn` para ambas acciones terminales, además de las rutas de pausa y finalización directas a humanos sin instrucciones, con una cobertura del 100 % del archivo. Las pruebas unitarias de `llm-replay` fijan el contrato de los marcadores: captura de última coincidencia ganadora, respaldo de coincidencia completa y fallos ruidosos para patrones sin coincidencia, inválidos y sin terminar. La nueva instantánea ACP sin clave `goal-wrapup` conduce la aplicación publicada a través de crear → ronda uno → complete autónomo y afirma la inyección de cierre con origen en el plugin, el mensaje de assistant de cierre en el mismo turno y el fin de turno `completed` tanto en el registro de sesión durable como en el flujo de stdout de ACP.

## Alternativas consideradas

- **Mostrar el texto de finalización en la tarjeta de UI de `update_goal`** — rechazado: `complete` no admite texto libre hoy, y añadir un argumento `summary` enrutaría un informe orientado al usuario a través de los argumentos de la herramienta y aun así cortaría el mensaje natural posterior al resultado del modelo.
- **Mantener `concludeTurn()` y añadir un primitivo de bucle de «un paso más solo de texto»** — rechazado: maquinaria nueva de `agent-loop` para un comportamiento que la detención ordinaria ya proporciona una vez que nada concluye el turno.
- **Instruir dentro del contenido del resultado de la herramienta** — rechazado: la salida canónica de las goal tools es JSON compacto consumido programáticamente; un bloque de instrucción en prosa dentro mezclaría el contrato orientado al modelo con el valor reproducible de la herramienta.

## Consecuencias

Todo objetivo autónomo termina con un mensaje de cierre orientado al usuario en lugar de una tarjeta de herramienta desnuda, al coste de una petición de modelo por ciclo de vida de objetivo. `concludeTurn()` conserva su semántica de bucle pero pierde su único llamador de primera parte fuera de la salida estructurada de subagente. Los escenarios de instantáneas ahora pueden guionizar valores que solo existen en tiempo de ejecución mediante `{{fromRequest:...}}`, lo que desbloquea la cobertura sin clave de cualquier flujo de herramienta de repetir-un-id, de objetivo o de otro tipo.
