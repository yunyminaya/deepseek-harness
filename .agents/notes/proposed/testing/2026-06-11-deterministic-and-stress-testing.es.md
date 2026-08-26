# Agent Note: Pruebas deterministas, el fixture de invariante de replay y estrés de carreras

Status: proposed

[English](2026-06-11-deterministic-and-stress-testing.md) | Español

## Problema

Varias pruebas del bucle se sincronizan con sleeps de `setTimeout(30)` — deuda de inestabilidad que desperdicia ciclos de agent en reintentos y puede enmascarar bugs de ordenación. Aparte, nuestra promesa arquitectónica central (cualquier registro de sesión se reproduce a una historia derivada idéntica) se afirma en dos pruebas pero es barata de afirmar *en todas partes*. Y la carrera de wakeup de la bandeja de entrada se verificó a mano exactamente una vez; nada la reverifica continuamente.

## Propuesta

Tres medidas:

1. **Nada de sleeps de reloj de pared en pruebas.** Reemplazar las esperas de `setTimeout(N)` por esperas dirigidas por eventos (el patrón `waitForIdle` existente, extendido a `waitForStatus`, `waitForEvent(n)`) o timers falsos de vitest cuando el tiempo mismo está bajo prueba. Aplicarlo con una regla de lint que prohíba `setTimeout` en `packages/*/tests` fuera de un módulo helper permitido explícitamente.
2. **Fixture de replay universal.** Un helper de prueba compartido envuelve el harness del bucle de modo que, tras cada prueba, el registro de sesión del agent se reproduce en una Session nueva y la igualdad de `deriveMessages()` se afirma automáticamente. El invariante queda entonces comprobado cientos de veces por ejecución de CI en cada escenario que la suite produce, no dos veces.
3. **Estrés nocturno de carreras.** Un trabajo de CI que corre las suites de agent-loop e inbox con `vitest --repeat=200` (y `--shuffle`) para sacar a la luz fallos dependientes de la programación; cualquier inestabilidad hallada es un bug por corregir, nunca un reintento.

## Plan

Aterrizar 1 y 2 juntos (tocan los mismos helpers); añadir el trabajo nocturno cuando la suite esté libre de sleeps para que las repeticiones sean rápidas.

## Criterios de aceptación

- No queda ningún `setTimeout` en `packages/*/tests` fuera del módulo helper permitido, aplicado por la regla de lint.
- El harness compartido reproduce el registro de sesión de cada prueba en una `Session` nueva y afirma la igualdad de `deriveMessages()` automáticamente, en toda la suite.
- El trabajo nocturno corre las suites de agent-loop e inbox con `--repeat` y `--shuffle`; una inestabilidad que encuentre se triaja como bug, nunca se reintenta hasta que desaparezca.

## Riesgos

Los timers falsos interactúan sutilmente con la programación de Promises en el bucle — preferir esperas dirigidas por eventos; reservar los timers falsos para el comportamiento del servicio de timers mismo.

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
