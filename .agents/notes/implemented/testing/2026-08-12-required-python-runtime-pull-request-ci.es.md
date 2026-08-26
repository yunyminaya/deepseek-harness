# Agent Note: Validación requerida del runtime de Python en el pull request

Status: implemented

[English](2026-08-12-required-python-runtime-pull-request-ci.md) | Español

## Problema

El CI ordinario de pull requests corre la suite pytest completa del SDK de Python contra peers de runtime falsos, mientras que las instantáneas de Node ejercitan clientes y salidas esperadas distintas. El cliente real de Python, el ejecutable JSON-RPC empaquetado, la instantánea específica del ejecutable, las wheels con forma de release y la instalación limpia se encuentran solo en los flujos opcionales de ejecutable único o de release de Python. Un cambio de evento de runtime o de cierre puede por tanto fusionarse con una proyección de Python obsoleta o un camino de wheel roto y fallar solo cuando alguien construya después un candidato a release de Python.

## Decisión

Cada pull request tiene un job `python-runtime` requerido en [CI](../../../../.github/workflows/ci.yml). Llama al [builder de ejecutable único](../../../../.github/workflows/build-exe-for-python-sdk.yml) compartido para `node24-linux-x64` sin filtro de path y participa en `all checks passed`. El flujo llamado construye el ejecutable real, corre todos los escenarios de Python sin clave de turno completo y de binario directo incluidos ambos instantáneas confirmadas, construye las wheels del SDK y del runtime, las instala en un entorno virtual limpio, comprueba los requisitos GLIBC del ejecutable y del addon nativo, y corre las wheels instaladas en un contenedor manylinux 2.28.

El job requerido y el [flujo de publicación de Python](../process/2026-08-11-python-publication-workflow.es.md) usan el mismo builder. Su clave de concurrencia incluye el flujo llamante, así que el CI requerido y una validación completa explícita de release para el mismo ref no se cancelan mutuamente. La matriz completa de linux-x64, linux-arm64 y macos-arm64 sigue siendo una validación de release porque el comportamiento de runtime, SDK e instantáneas independiente de plataforma necesita un portador nativo único que bloquee merges, mientras que el comportamiento específico de arquitectura de ejecutable, addon, wheel-tag y deployment-target sigue necesitando todos los targets de release antes de la publicación.

La instantánea avanzada del ejecutable normaliza identificadores opacos de sesión, mensaje, subagente y ejecución de workflow antes de la comparación. Un evento de workflow recién persistido cambia por tanto la salida esperada revisada sin convertir un identificador de ejecución aleatorio en parte de esa salida. La [instantánea visible para el modelo](2026-08-13-python-minimal-model-visible-snapshot.es.md) del escenario mínimo cubre el system prompt ensamblado, los schemas de herramienta y la lista de mensajes que esta tokeniza.

## Alternativas consideradas

**Correr la matriz nativa completa en cada pull request.** Esto duplica el comportamiento de turno completo e instantáneas independiente de plataforma en tres jobs y consume capacidad de ARM64 Linux y macOS en cada cambio. El flujo de publicación conserva esa evidencia en el punto donde se requieren los tres artefactos.

**Correr la instantánea contra el portador Node de desarrollo.** Esto atrapa la deriva de protocolo y de proyección de eventos pero no prueba el ensamblaje de pkg, el cierre de runtime desplegado, el staging del addon nativo, la construcción de wheels, los pins exactos de dependencias ni la instalación limpia. El camino de ejecutable Linux requerido cubre el path publicado directamente.

**Seleccionar el job con filtros de path o labels.** El comportamiento de Python depende de código compartido de agent, sesión, workflow, subagente, carga de plugins y empaquetado fuera de `python/`. Un filtro de dependencias incompleto recrea el fallo retardado, y un label deja la evidencia opcional.

## Consecuencias

Cada pull request paga por un build de ejecutable y wheel de Linux alojado estándar, y `all checks passed` lo espera. Esto convierte la distribución de Python de primera parte en un contrato en tiempo de merge y reutiliza la implementación de release en lugar de mantener una tubería sustituta más pequeña.

Una arquitectura requerida no puede detectar regresiones de empaquetado de macOS ni de Linux ARM64. La validación completa explícita de release sigue siendo obligatoria antes de la publicación y es dueña de esos resultados específicos de plataforma.
