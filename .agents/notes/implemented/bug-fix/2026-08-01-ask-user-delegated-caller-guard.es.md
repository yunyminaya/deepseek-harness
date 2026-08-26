# Agent Note: Rechazar la interacción humana de los subagentes propiedad del runtime

[English](2026-08-01-ask-user-delegated-caller-guard.md) | [中文](2026-08-01-ask-user-delegated-caller-guard.zh.md) | Español

Status: implemented

## Problema

Un subagente de un solo disparo que llama a `ask_user_question` puede bloquearse indefinidamente. La llamada espera una respuesta humana, pero el hijo no tiene un canal humano de propiedad independiente, por lo que tanto la finalización del hijo como la del padre que espera esa finalización se estancan.

El linaje de sesión durable no puede decidir si existe un respondedor. Una sesión hija puede reanudarse más tarde como una nueva raíz de runtime de nivel superior, mientras que un hijo vivo propiedad del runtime puede tener una profundidad de delegación durable de cero o ausente. La guía de errores en el seam compartido debe encajar también con todos los consumidores: `exit_plan_mode` usa `ctx.userQuestions.ask()` sin llamar a `ask_user_question`.

## Decisión

Cuando `AskUserQuestionRequest.agent` está presente, `UserQuestionService.ask()` autentica al agente vivo exacto a través de `ctx.agents` y solo lo admite cuando `ctx.agents.roots()` contiene esa instancia. Un registro ausente o un objeto obsoleto con el mismo id falla con `CALLER_NOT_LIVE`; un agente vivo propiedad de otro agente vivo falla con `DELEGATED_CALLER`. La comprobación se ejecuta después de las guardas existentes de aborted y de lote vacío y antes de la validación de intención o del despacho al provider, por lo que un hijo con propietario nunca crea una espera de interfaz.

La propiedad del runtime es la autoridad. Una sesión con linaje reanudada sin propietario es una raíz de runtime y puede preguntar; un hijo vivo sigue sin ser elegible incluso cuando su `delegationDepth` durable es cero. Las llamadas programáticas sin agent conservan la ruta de provider existente.

El texto de error compartido es neutro para el consumidor y accionable: el hijo incluye la pregunta o decisión no resuelta en su resultado final. El padre ya recibe ese resultado a través del contrato de delegación y puede decidir si pregunta al humano. Ni el servicio ni un hijo reivindican una capacidad de mensajería ascendente o de reenvío de respuestas que no existe.

Este límite de seguridad es independiente de la elección de composer del navegador. Las [fases de composer semántico propuestas](../../proposed/architecture/2026-08-08-semantic-composer-chain-phases.es.md) abordan cómo deben ordenarse una interacción ya pendiente y una superficie de subagente de solo lectura; no debilitan esta guarda de runtime.

## Alternativas consideradas

**Usar `session.header.delegationDepth > 0`.** Rechazado porque el linaje durable sobrevive a la reanudación y no da fe del propietario actual local del proceso. Rechaza raíces reanudadas válidas y puede admitir a un hijo vivo cuyo encabezado durable es incompleto.

**Rechazar solo dentro de `dsh-tool-ask-user`.** Rechazado porque `exit_plan_mode` y los llamadores directos comparten `ctx.userQuestions.ask()`. El servicio es el límite de operación estrecho común a todos los consumidores de interacción humana.

**Decir al hijo que delegue hacia arriba o que espere el reenvío.** Rechazado porque la delegación de un solo disparo no expone ningún canal de petición hijo-a-padre ni protocolo de reenvío de respuestas. La única ruta de retorno garantizada es el resultado final del hijo.

**Confiar en la corrección del composer del navegador.** Rechazado porque la presentación no puede hacer que exista un canal humano sin propietario, y los despliegues que no son de navegador siguen necesitando que la llamada termine.

## Consecuencias

Las llamadas de hijos propiedad del runtime fallan rápido con un error estructurado estable en lugar de colgarse. Las raíces vivas exactas y las llamadas programáticas sin agent siguen siendo elegibles, incluidas las sesiones reanudadas con linaje histórico de hijo. `ask_user_question` y `exit_plan_mode` reciben la misma guía correctiva neutra, mientras que sus schemas visibles para el modelo y sus prefijos de system prompt permanecen sin cambios; solo difiere el resultado de error añadido, por lo que los prefijos existentes de KV-cache siguen siendo reutilizables.

## Pruebas

Las pruebas del servicio cubren un hijo vivo de profundidad cero, una raíz de runtime reanudada de profundidad uno, un registro ausente, un objeto obsoleto con el mismo id y la no invocación del provider en cada rechazo. Las pruebas de herramienta y de modo plan demuestran que ambos consumidores muestran el resultado neutro de `DELEGATED_CALLER` y nunca llegan al provider. La instantánea ensamblada sin clave delega en un hijo que intenta llamar a `ask_user_question`, fija el resultado de error de herramienta del hijo y su entrega final, y demuestra que el padre completa en lugar de esperar una respuesta.
