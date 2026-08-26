# Agent Note: La escalera de disposición pertenece a su consumidor, no al seam de subprocesos

Status: implemented

[English](2026-07-27-dispose-ladder-to-consumer.md) | Español

## Problema

`SubprocessHandle.dispose(graces)` y `SubprocessDisposeGraces` ponían una *política* completa de teardown — espera de EOF de stdin, luego SIGTERM, luego SIGKILL, cada nivel acotado por una ventana suministrada por el llamador — en un seam cuyos otros verbos son mecanismos individuales. Solo un consumidor la llamó alguna vez (el backend ACP de subagentes); bash usa `terminate()` y el teardown del servicio, y el host LSP ejecuta su propio apagado protocolo-primero. Todo backend futuro debía sin embargo implementar la escalera para satisfacer la interfaz, y la implementación arrastraba una dependencia de `dsh-timeout` únicamente para los límites de nivel de la escalera.

## Decisión

La escalera se mueve a su único consumidor. `dsh-subagent-acp` es dueño de `disposeAcpChild(child, eofGraceMs)`, construida enteramente sobre los verbos públicos del seam: cerrar `stdin`, acotar un `waitForExit` en `eofGraceMs`, luego llamar a `terminate()`, cuya escalada SIGTERM→spec-grace→SIGKILL ya es dueña del temporizador de señal, y esperar un `waitForExit()` sin acotar la prueba de salida de todo el árbol del propietario del subproceso. El seam conserva `kill`/`terminate`/`waitForExit` — mecanismos, no política — y `waitForExit(signal?)` es exactamente la sonda de quiescencia que una escalera de consumidor necesita para mantener el nivel cooperativo en la salida real del árbol sin derivar otro temporizador de la gracia de terminación. El handle del seam pierde un método y una interfaz exportada.

## Alternativas consideradas

**Conservar la escalera en el handle como conveniencia.** Rechazada: un método de Service Definition que todo Service Provider debe implementar no es una conveniencia, es superficie de contrato — y este codifica la forma de cooperación de un consumidor (EOF-de-stdin-primero) como si fuera vocabulario de procesos. El propio README del seam ya tenía que advertir que los hijos que quiescen ante otras señales necesitan «su propio nivel 1», lo cual es la admisión de que la escalera es política.

**Mover la escalera a un paquete helper compartido.** Rechazada: un único consumidor. Un segundo backend fuera de proceso con la misma forma de cooperación EOF-de-stdin puede subir `disposeAcpChild` a código compartido cuando exista; extraerla ahora recrearía `dsh-subagent-subprocess`, la librería de un solo propósito que este cambio eliminó.

## Consecuencias

Ganado: la Service Definition es un método y un tipo más pequeños; los Service Providers deben cuatro verbos y ninguna política de teardown; la ventana cooperativa de EOF vive junto al campo de configuración ACP que la ajusta, mientras el propietario del subproceso es dueño en solitario de la ventana de terminación y del join final. Coste: un backend futuro que quiera teardown EOF-primero escribe ~20 líneas contra los verbos (o sube el helper de ACP); los tests de nivel de la escalera viven en la suite ACP, y la suite de Service Definition fija los verbos que la escalera compone (un `waitForExit` acotado falso antes de la escalada y un join de todo el árbol sin acotar después) en lugar de la política compuesta.
