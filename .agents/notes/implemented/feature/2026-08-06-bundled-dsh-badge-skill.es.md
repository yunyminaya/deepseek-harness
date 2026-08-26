# Agent Note: Skill de insignia dsh incluida

Status: implemented

[English](2026-08-06-bundled-dsh-badge-skill.md) | Español

## Problema

El [tutorial de Cordis](../../../../docs/cordis-tutorial/index.es.md) usa una insignia oficial «powered by dsh» en sus páginas, pero el CLI incluido no tiene instrucciones reutilizables ni un provider de opt-in explícito para aplicar la misma atribución en otros lugares.

## Decisión

`@deepseek-ai/dsh-skill-badge` es un plugin nativo de Cordis que registra un provider inmutable incluido en `ctx.skills`. El provider es dueño del resumen `dsh-badge`, del cuerpo de instrucciones y de la base de recursos PNG; `dsh-tool-skill` sigue siendo el único dueño del catálogo orientado al modelo y del renderizado del loader.

La composición CLI incluida declara `skill-badge` como deshabilitado. Habilitar esa fila existente es el opt-in explícito; las instalaciones deshabilitadas no anuncian ninguna skill (destreza) de insignia y no ganan contenido visible para el modelo.

El provider usa el rango incluido después de las fuentes del filesystem de proyecto, personal y de usuario, así que una definición `dsh-badge` propiedad del usuario puede sobreescribirlo a través del contrato ordinario de precedencia del registro. La disposición del provider elimina la contribución mediante el efecto propiedad del registro.

## Alternativas consideradas

**Montar los archivos empaquetados a través de `dsh-skill-filesystem`.** Rechazado porque el descubrimiento, el análisis y la vigilancia del filesystem añaden maquinaria de ciclo de vida que un provider de una sola skill inmutable no necesita.

## Consecuencias

Las instrucciones de la insignia y el PNG fuente se versionan con DSH y se resuelven a través de una base de recursos de directorio empaquetado. El provider no tiene superficie de configuración. Los tests del paquete fijan el ciclo de vida del provider y los bytes oficiales del PNG, mientras que un snapshot de aplicación ensamblada sin clave fija el catálogo habilitado y el cuerpo de skill cargado.
